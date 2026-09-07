# Phase 0 ( input for instalaltion) 

The same input as for ABI 

```
https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

https://console.redhat.com/openshift/create/datacenter 
```

# Phase 1 ( Nexus deployment/configuration ) - automated by ansible playbook 

```
# pełny reset przed ponownym wdrożeniem (with external option) 
ansible-playbook playbooks/cleanup-nexus.yaml -e cleanup_confirm=registry.m1.local.net

# bash script to deploy nexus 
sudo ./nexus-installation.sh

# configure nexus 
ansible-playbook -i inventory/hosts.yml playbooks/configure-nexus.yaml -vvv

```


# Phase 2 ( OC mirror ) - automated by ansible playbok 

```

# oc mirror - releases 
ansible-playbook -i inventory/hosts.yml playbooks/sync-nexus.yaml -e ocp_channel=stable-4.22 -e ocp_min_version=4.22.11 -e ocp_max_version=4.22.11 -vvv

#oc mirror - operator 
ansible-playbook playbooks/sync-nexus.yaml \
    -e mirror_content=operators \
    -e '{"mirror_operator_packages":[
          {"name":"openshift-gitops-operator","channels":[{"name":"latest"}]},
          {"name":"lvms-operator","channels":[{"name":"stable-4.21"}]}
        ]}'

```


# Phase 3 ( Input files, install-config.yaml / agent-config.yaml ) 



## Cert file 

```
# cert for local registry (CA + nexus cert, we can take both and add it to the input file) 
openssl s_client -connect registry.m1.local.net:5000 -showcerts </dev/null 2>/dev/null \
  | openssl crl2pkcs7 -nocrl -certfile /dev/stdin \
  | openssl pkcs7 -print_certs

```

## openshift-isntall estraction from local registry 

```

# to save authetication info on jumhost 
podman login https://registry.m1.local.net:5000/

curl -su 'admin:devsecops135!!!' https://registry.m1.local.net:5000/v2/_catalog
curl -su 'admin:devsecops135!!!' https://registry.m1.local.net:5000/v2/openshift/release-images/tags/list

#idms file search, created by oc mirror (not ansible) 
find / -name 'idms-oc-mirror.yaml' 2>/dev/null

cat /opt/mirror/cluster-resources/release/idms-oc-mirror.yaml
cat /opt/mirror/workspace-release/working-dir/cluster-resources/idms-oc-mirror.yaml


---
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  annotations:
    createdAt: Wednesday, 02-Sep-26 13:16:28 UTC
    createdBy: oc-mirror v2
    oc-mirror_version: 4.22.0-202608170914.p2.g88debf3.assembly.stream.el9-88debf3
  name: idms-release-0
spec:
  imageDigestMirrors:
  - mirrors:
    - registry.m1.local.net:5000/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - registry.m1.local.net:5000/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release


# extract openshift-install  
oc adm release extract \
  -a $XDG_RUNTIME_DIR/containers/auth.json \
  --idms-file=/opt/mirror/cluster-resources/release/idms-oc-mirror.yaml \
  --command=openshift-install \
  --to . \
  registry.m1.local.net:5000/openshift/release-images:4.22.11-x86_64

```

## install-config.yaml

```
apiVersion: v1

# Domena bazowa. FQDN API powstaje jako api.<metadata.name>.<baseDomain>
baseDomain: local.net

metadata:
  # Nazwa klastra. Razem z baseDomain musi zgadzać się z rekordami DNS.
  name: m1

compute:
  - name: worker
    # SNO = zero osobnych węzłów compute. Węzeł master pełni obie role.
    replicas: 0
    architecture: amd64

controlPlane:
  name: master
  # replicas: 1 to jedyna wartość definiująca SNO.
  # Ta liczba = liczba endpointów etcd, więc musi odpowiadać
  # liczbie faktycznie bootowanych maszyn.
  replicas: 1
  architecture: amd64

networking:
  # OVNKubernetes to jedyna wspierana wartość dla platformy none.
  networkType: OVNKubernetes
  clusterNetwork:
    # Pula adresów dla podów. Nie może kolidować z Twoją siecią fizyczną.
    - cidr: 10.128.0.0/14
      # /23 na węzeł = 510 adresów podów.
      hostPrefix: 23
  serviceNetwork:
    # Pula adresów dla Service (ClusterIP).
    - 172.30.0.0/16
  machineNetwork:
    # Twoja realna sieć LAN, w której siedzi węzeł.
    # UWAGA: nie może zawierać 10.88.0.0/16 - to domyślny bridge Podmana.
    - cidr: 192.168.1.0/24

platform:
  # Dla SNO 'none' jest wymagane.
  none: {}

fips: false


# Pull secret. W środowisku odłączonym wystarczą tu poświadczenia
# do Nexusa. Jeśli węzeł ma dostęp do internetu, możesz dokleić
# oryginalne wpisy Red Hata. Ansible playbook to scala z pull-secret.json 
# ale mozna to wycaignac poleceniem 
pullSecret: '{"auths":{"registry.m1.local.net:5000":{"auth":"b2NwLW1pcnJvcjpkZXZzZWNvcHMxMzUhISE=","email":"ocp-mirror@local"}}}'

# Klucz publiczny SSH dla użytkownika core. Bez niego nie zdebugujesz
# nieudanej instalacji.
sshKey: 'ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIG8NDDTxf+xahE2nJt8/ssNgnOhhvTVg0u88BBaXCFBi m1.local.net'


# --- CZĘŚĆ DISCONNECTED / NEXUS ---


# Certyfikat CA Twojego Nexusa (PEM). Instalator wstrzyknie go
# do trust store węzła, dzięki czemu kubelet/CRI-O zaufa rejestrowi.
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIF/TCCA+WgAwIBAgIBAjANBgkqhkiG9w0BAQsFADBvMQswCQYDVQQGEwJQTDEQ
  MA4GA1UECAwHTWF6b3ZpYTEPMA0GA1UEBwwGV2Fyc2F3MQswCQYDVQQKDAJtMTEX
  MBUGA1UECwwOSW5mcmFzdHJ1Y3R1cmUxFzAVBgNVBAMMDm0xIEludGVybmFsIENB
  MB4XDTI2MDkwNzEwNTUwN1oXDTI4MTIxMDEwNTUwN1owcDELMAkGA1UEBhMCUEwx
  EDAOBgNVBAgMB01hem92aWExDzANBgNVBAcMBldhcnNhdzELMAkGA1UECgwCbTEx
  ETAPBgNVBAsMCFJlZ2lzdHJ5MR4wHAYDVQQDDBVyZWdpc3RyeS5tMS5sb2NhbC5u
  ZXQwggIiMA0GCSqGSIb3DQEBAQUAA4ICDwAwggIKAoICAQDl3xolL2l6AIvPML6y
  bgO5UHDqCzUqrI9SK8lpI4IEtWCc0qgKqcQHPqxiBraP52r1eD0iCqpPn6rerDcW
  a38TjVThElUadCOEUx52sla+6jsKKecF/y81/IVmurFN1LqpDo17bvybiCg48/7I
  8ryHye0Y2CjelyqPOM1e5u5Npb7nMmroSDosxyoAwqNzp3Nurw06w6Np3wHta82s
  nKJnCzY/9e8ku/Z84vZY9Bfh0jZk/YPOdZ1j71vmIkZwsjHGUkwPv69jQiTTVVi1
  c7yDkgcOx5YeRwBHOa4PK7HS5Y4Q5MirX/5u0TNNf4GNwaxqKzS/q/r0IhBoInM6
  D4qJ7+LToxnW6iv3uoSkbXQazdX46UefJgGPrFXn7ngLyHJZHVD1GbOVLwSam6KU
  VUoqDoSm9Yl3MSN17e9DVM3OQXIU7PugZSHV+5vRdER9lnITCSCu7ePzopsBV2FM
  NLzmRPbm69kgTtMIuuO+P85tJ1RJGTbkfGLoGUGtKaej/KMf4N1GuKqFSC3JTEuH
  TzEWFfTDyCk2WvMO0+mOcNkU096UlaiFU2w3fi2qPN4gYgiw31ur4/OWYtCUZ9Q6
  JDhYoZh1ZfTO0mZEj4sVzyJpimy4vAP9vef8xrG190i1FE2RoRnJucwhDEht+a7t
  3o4Zq/mZDkrlac9lprBeEJrNLwIDAQABo4GiMIGfMB8GA1UdIwQYMBaAFGJhn2GI
  mB4CwvQhGi2AminHliv/MAkGA1UdEwQCMAAwCwYDVR0PBAQDAgTwMBMGA1UdJQQM
  MAoGCCsGAQUFBwMBMDAGA1UdEQQpMCeCFXJlZ2lzdHJ5Lm0xLmxvY2FsLm5ldIII
  cmVnaXN0cnmHBMCoAQ4wHQYDVR0OBBYEFIvLaKksNp3JN8BkL6w+70M0UkD7MA0G
  CSqGSIb3DQEBCwUAA4ICAQBwFODyd1ZeATjlOJb0J5FcWZTr7tyDNj/H2nRkBX9n
  j1f/xWDK8YaHolxoQFpNW8okx6gq9t4TYivObk6dFR7edJgc4hkRny5ab5UhQcku
  rfPvg0EEAD3FTguS2Q/7KJgG5K351jgCdw0w8f/Iiw0Jj+V7XOVMP1oypngkgKje
  qYbDnwgbmPASxY+RfHPLkh66sGizpI73FdVFnkGMzd1mqYInwY1cqb/3WQsXJzym
  cO6vPzB+LW+uhYbed8vBcNlHhfBWf1bSHtr6bCuOr3fNYsm0WsFB+6WMvNH0CWpe
  UcslWmc9uI676xVGIuDCrYhY56/C0fYdbr4SS7K/Ih3F9T0wLUUwQmmGuBGeXtvb
  O3/q76LcX07hrOip8xJh6HRfZc5fc/+k+OYYNGCSewqQY+Ckx+G4dS3XEH720mgx
  ipZFqtEavMrzUCLzzRDzid19U/J8CGMXWDliuKiVRf07RcRXddJv5JvUaZ4tLH3e
  1JhW0LYXBSFj19MqR/Qq21EOIO0XdqUPzy0+gRnJ/2CFlwj06EzABUkdU/sLTQym
  jkxJRDo/d9ZdECzSBXqhdZabm41QLRkCUF+RYppnNgdLtSYVmD6nCx5FDw18++PF
  MFk63pi17YBAImorclolCznzG+ZsiJ9bc6L+Zl5ky/dyyGW2wS00LhXOFqmimvhN
  zQ==
  -----END CERTIFICATE-----
  -----BEGIN CERTIFICATE-----
  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
  9cMZncDl
  -----END CERTIFICATE-----

# Wymusza dodanie CA do trust store nawet gdy nie ma proxy.
additionalTrustBundlePolicy: Always

# Mapowanie: "gdy ktoś prosi o obraz z source, weź go z mirrors".
# Te wartości SKOPIUJ z pliku idms-oc-mirror.yaml wygenerowanego
# przez oc-mirror 
imageDigestSources:
  - mirrors:
    - registry.m1.local.net:5000/openshift/release
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - registry.m1.local.net:5000/openshift/release-images
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - registry.m1.local.net:5000/openshift4
    source: registry.redhat.io/openshift4
  - mirrors:
    - registry.m1.local.net:5000/openshift-gitops-1
    source: registry.redhat.io/openshift-gitops-1
  - mirrors:
    - registry.m1.local.net:5000/rhel9
    source: registry.redhat.io/rhel9
  - mirrors:
    - registry.m1.local.net:5000/lvms4
    source: registry.redhat.io/lvms4

```


## agent-config.yaml

```
apiVersion: v1beta1
kind: AgentConfig

metadata:
  name: m1

# IP hosta, na którym wystartuje Assisted Service (rendezvous host).
# Dla SNO to po prostu IP jedynego węzła.
# Ten adres musi być znany w momencie generowania ISO.
rendezvousIP: 192.168.1.15

hosts:
  - hostname: m1
    # Rola musi być 'master' lub 'worker'.
    # rendezvousIP zawsze musi trafić na hosta z rolą master.
    role: master

    interfaces:
      # MAC to identyfikator hosta - po nim ABI dopasowuje konfigurację.
      # Każdy interfejs musi mieć MAC i muszą być unikalne.
      - name: ens18
        macAddress: 52:AB:3E:7D:91:F4

    rootDeviceHints:
      # Na który dysk zapisać RHCOS. Instalator bierze pierwszy
      # pasujący dysk. Zalecana forma to /dev/disk/by-path/...
      # bo /dev/sda potrafi się przenumerować.
      deviceName: "/dev/sda"

    # Konfiguracja sieci w formacie NMState (ten sam co w nmstate/NNCP).
    networkConfig:
      interfaces:
        - name: ens18
          type: ethernet
          state: up
          mac-address: 52:AB:3E:7D:91:F4
          ipv4:
            enabled: true
            dhcp: false
            address:
              - ip: 192.168.1.15
                prefix-length: 24
          ipv6:
            enabled: false
      dns-resolver:
        config:
          server:
            - 192.168.1.14
      routes:
        config:
          - destination: 0.0.0.0/0
            next-hop-address: 192.168.1.1
            next-hop-interface: ens18
            # 254 to główna tablica routingu Linuksa.
            table-id: 254

# Źródła NTP. W środowisku odłączonym KRYTYCZNE - rozjazd zegara
# psuje walidację certyfikatów i bootstrap się wywala.
additionalNTPSources:
  - 192.168.1.14

```




# Phase 4 ( IOS generation ) 

```
# we arleady extract bianry from local repo (phase 3 ) 
./openshift-install agent create image --dir ~/sno-abi
./openshift-install agent create image --dir ~/sno-abi --log-level debug

```
# Phase 5 ( Booting ) 

```
scp agent.x86_64.iso root@192.168.1.200:/var/lib/vz/template/iso/

./openshift-install agent wait-for bootstrap-complete
./openshift-install wait-for install-complete

```

# Phase 6 ( Configuration after instalaltion ) 

```
oc apply -f https://raw.githubusercontent.com/pindych-michal/argocd-demo/main/argocd-apps/root.yaml

oc apply -f https://raw.githubusercontent.com/pindych-michal/argocd-demo/main/bootstrap/custom-manifest.yaml

```


