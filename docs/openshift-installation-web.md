# 0. Prerequisite

- Bind DNS server configuration 
- IOS on Proxmox (URL link from redhat cloude) 
- GIT repo prepared - please refer to https://github.com/pindych-michal/argocd-demo





# 1. Openshift instalaltion from cloud

```
https://console.redhat.com/openshift/create/datacenter 

Bare metal installtion 
Interactive (Web based) 

// This values must be identical to support DNS config 
clustername m1 
domain local.net 
DNS: 192.168.1.14 (Ubuntu Bind, Ansible, JumpHost, NTP, Nexus) 


// Network/Infra configuration 
-- This mac address must be used (reserved in network) 52:AB:3E:7D:91:F4 , 52:ab:3e:7d:91:f4
-- Static IP: 192.168.1.15  


//custom manifest to install ARGO CD  
apiVersion: v1
kind: Namespace
metadata:
  name: openshift-gitops-operator
  labels:
    openshift.io/cluster-monitoring: "true"
---
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  upgradeStrategy: Default
---
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-gitops-operator
spec:
  channel: latest
  installPlanApproval: Automatic
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: argocd-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: openshift-gitops-argocd-application-controller
    namespace: openshift-gitops

```








# 3. Proxmox configuration  


```
download ISO from prevoius step 

correct order of booting (scsi0, ide2, net0, scsi1) 

mem:
memory 29500
allow KSM 

cpu:
Sockets 1 
cores 16 
type host 
enable numa 
nasted-virt on 

disc:
200 GB - sytem  
600 GB - images 

network:
52:AB:3E:7D:91:F4 (mac address)

boot the VM 

```


# 4. ARGO CD Sync 

oc apply -f https://raw.githubusercontent.com/pindych-michal//argocd-demo/main/argocd-apps/root.yaml 

```
Deployment of:
- LVM operator - recommended for SNO
- Virtualization Operator (Manual install from GitOps) - we need to have default storage class 
- Nginx demo 

```

# 5. Final testing after deployment 

```
disk: 

NODE=$(oc get nodes -o jsonpath='{.items[0].metadata.name}')
oc get --raw /api/v1/nodes/$NODE/proxy/stats/summary | jq '.node | {nodeFs: .fs, imageFs: .runtime.imageFs}'
oc debug node/m1 -- chroot /host lvs -o lv_name,lv_size,data_percent,metadata_percent vg1
oc get pvc -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,SIZE:.spec.resources.requests.storage,SC:.spec.storageClassName

```


```
#cluster status bash script 


#!/bin/bash

# OpenShift Cluster Health Check Script
# Zapisuje wynik do pliku w bieżącym katalogu

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
OUTPUT_FILE="openshift_healthcheck_${TIMESTAMP}.txt"
LOG_DIR="."

# Kolory (opcjonalne, działają w terminalu)
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo -e "${GREEN}=== OpenShift Cluster Health Check ===${NC}"
echo "Start: $(date)"
echo "Wynik zostanie zapisany do: ${OUTPUT_FILE}"
echo

# Sprawdzenie czy oc jest dostępne
if ! command -v oc &> /dev/null; then
    echo -e "${RED}Błąd: komenda 'oc' nie została znaleziona.${NC}"
    exit 1
fi

# Sprawdzenie czy jesteśmy zalogowani
if ! oc whoami &> /dev/null; then
    echo -e "${RED}Błąd: nie jesteś zalogowany do klastra (oc whoami nie działa).${NC}"
    exit 1
fi

{
    echo "========================================================"
    echo "  OpenShift Cluster Health Check"
    echo "  Data: $(date)"
    echo "  Użytkownik: $(oc whoami)"
    echo "  Serwer: $(oc whoami --show-server 2>/dev/null || echo 'nieznany')"
    echo "========================================================"
    echo

    echo ">>> 1. Cluster Version"
    echo "--------------------------------------------------------"
    oc get clusterversion -o wide 2>&1
    echo
    oc get clusterversion -o yaml 2>&1 | head -100
    echo
    echo

    echo ">>> 2. Cluster Operators"
    echo "--------------------------------------------------------"
    oc get clusteroperators 2>&1
    echo
    echo "--- Operatory w stanie DEGRADED / PROGRESSING / niedostępne ---"
    oc get co --no-headers 2>/dev/null | awk '$3!="True" || $4!="False" || $5!="False" {print}'
    echo
    echo

    echo ">>> 3. Nodes"
    echo "--------------------------------------------------------"
    oc get nodes -o wide 2>&1
    echo
    echo

    echo ">>> 4. Pody niebędące w stanie Running / Succeeded"
    echo "--------------------------------------------------------"
    oc get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded 2>&1
    echo
    echo

    echo ">>> 5. Krytyczne namespace'y"
    echo "--------------------------------------------------------"
    for ns in openshift-kube-apiserver openshift-etcd openshift-authentication openshift-console openshift-ingress openshift-monitoring; do
        echo "----- Namespace: $ns -----"
        oc get pods -n $ns -o wide 2>&1
        echo
    done
    echo

    echo ">>> 6. Zużycie zasobów (jeśli metrics dostępne)"
    echo "--------------------------------------------------------"
    echo "--- Nodes ---"
    oc adm top nodes 2>&1
    echo
    echo "--- Top 20 podów według CPU ---"
    oc adm top pods -A --sort-by=cpu 2>&1 | head -25
    echo
    echo

    echo ">>> 7. Ostatnie eventy (potencjalne problemy)"
    echo "--------------------------------------------------------"
    oc get events -A --sort-by='.lastTimestamp' 2>&1 | tail -50
    echo
    echo

    echo ">>> 8. Machine Config / MCP (jeśli używasz)"
    echo "--------------------------------------------------------"
    oc get mcp 2>&1 || echo "Brak MachineConfigPools (OK dla niektórych instalacji)"
    echo
    oc get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.machineconfiguration\.openshift\.io/state}{"\n"}{end}' 2>/dev/null || true
    echo
    echo

    echo "========================================================"
    echo "  Koniec raportu: $(date)"
    echo "========================================================"

} > "${OUTPUT_FILE}" 2>&1

# Podsumowanie na ekranie
echo -e "${GREEN}Gotowe!${NC}"
echo "Raport zapisany do pliku: ${OUTPUT_FILE}"
echo
echo "Szybkie podsumowanie:"
echo "---------------------"
echo -n "ClusterVersion: "
oc get clusterversion --no-headers 2>/dev/null | awk '{print $2, $3, $4, $5}'
echo
echo "Problematyczne operatory:"
oc get co --no-headers 2>/dev/null | awk '$3!="True" || $4!="False" || $5!="False" {print "  - "$1}'
echo
echo "Node'y nie Ready:"
oc get nodes --no-headers 2>/dev/null | awk '$2!="Ready" {print "  - "$1" ("$2")"}'
echo
echo "Aby zobaczyć pełny raport:"
echo "  less ${OUTPUT_FILE}"
echo "  cat ${OUTPUT_FILE}"



```
