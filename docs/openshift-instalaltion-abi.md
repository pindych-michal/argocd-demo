# Phase 0 ( input for instalaltion) 

For ABI we need to provide two input files: install-config.yaml and agent-config - then download openshift-install binary

```
# redhat documentation 
https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/installing_an_on-premise_cluster_with_the_agent-based_installer/preparing-to-install-with-agent-based-installer

# ABI dowlaod 
https://console.redhat.com/openshift/create/datacenter 

```

```
//SSH generation from jumphost 
ssh-keygen -t ed25519 -C "m1.local.net" -f ~/.ssh/ocp-sno

~/.ssh/ocp-sno       <- klucz PRYWATNY, nigdy go nie pokazuj ani nie wklejaj do YAML
~/.ssh/ocp-sno.pub   <- klucz PUBLICZNY, ten idzie do install-config.yaml
cat ~/.ssh/ocp-sno.pub

// Pull secret downlaod from redhat (access to public registry) 
https://console.redhat.com/openshift/create/datacenter 

```


# Phase 1 ( Input file preparation: install-config.yaml / agent-config.yaml ) 


## install-config

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

# Bez tego: "invalid install-config: pullSecret: Required value", pobieranie obrazow z neta 
pullSecret:
{
  "auths": {
    "cloud.openshift.com": {
      "auth": "secret",
      "email": "secret"
    },
    "quay.io": {
      "auth": "secret",
      "email": "secret"
    },
    "registry.connect.redhat.com": {
      "auth": "secret",
      "email": "secret"
    },
    "registry.redhat.io": {
      "auth": "secret",
      "email": "secret"
    }
  }
}




# Formalnie opcjonalny, ale bez niego nie wejdziesz na węzeł
# przez SSH gdy instalacja się wywali - a przy pierwszym podejściu
# wywali się prawie na pewno.
sshKey: '................ m1.local.net'

```


## agent-config

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
        macAddress: 52:ab:3e:7d:91:f4

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
          mac-address: 52:ab:3e:7d:91:f4
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

# Phase 2 ( IOS generation ) 

```
# downlaod openshift-install from READHAT, wersja tej binarki decyduje o tym co sciagac  
# ewentualnie najnowsza paczke mozna pobrac z https://console.redhat.com/openshift/create/datacenter 
curl -O https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/4.22.0/openshift-install-linux.tar.gz
tar xzf openshift-install-linux.tar.gz

devsecopsadmin@registry:~$ ./openshift-install version
./openshift-install 4.22.11
built from commit 27857bd7921fc12dc3681c4812892cff949a1d34
release image quay.io/openshift-release-dev/ocp-release@sha256:7130a86441e55382a1eecf141edefd4a83c6eb7b27c6cb67883027736b8242b3
release architecture amd64


# potem kopiowanie plikow do katalogu 
mdir sno-abi 
copy input files (installer/agnet) 
cp -r sno-abi/ sno-abi-backup/



#binarka robi dodatkowa walidacje yamls nmstate i potrzebuje tej paczki 
podman run --rm registry.fedoraproject.org/fedora:latest bash -c \
  "dnf install -y nmstate >/dev/null 2>&1 && cat /usr/bin/nmstatectl" \
  > /tmp/nmstatectl

sudo install -m 0755 /tmp/nmstatectl /usr/local/bin/nmstatectl
nmstatectl --version
  

# odpalnie samej binarki 
./openshift-install agent create image --dir ~/sno-abi
./openshift-install agent create image --dir ~/sno-abi --log-level debug


```
# Phase 3 ( Booting ) 


```
# copy the files between jumphost and proxmos 
scp ubuntu-24.04.iso root@192.168.1.200:/var/lib/vz/template/iso/

```

```
# Monitoring progress of installation 
openshift-install agent wait-for bootstrap-complete
openshift-install wait-for install-complete
```
