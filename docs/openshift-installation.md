# 0. Prerequisite
-- Bind DNS server configuration 
-- IOS on Proxmox (URL link from redhat cloude) 
-- GIT repo prepared - please refer to https://github.com/pindych-michal/argocd-demo





# 1. Openshift instalaltion from cloud

```
https://console.redhat.com/openshift/create/datacenter 

Bare metal installtion 
Interactive (Web based) 

// This values must be identical to support DNS config 
clustername m1 
domain local.net 
DNS: 192.168.1.14 (Ubuntu Bind, Ansible, JumpHost) or 192.168.1.1 ?? 


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

5. Final testing after deployment 

```
disk: 

NODE=$(oc get nodes -o jsonpath='{.items[0].metadata.name}')
oc get --raw /api/v1/nodes/$NODE/proxy/stats/summary | jq '.node | {nodeFs: .fs, imageFs: .runtime.imageFs}'
oc debug node/m1 -- chroot /host lvs -o lv_name,lv_size,data_percent,metadata_percent vg1
oc get pvc -A -o custom-columns=NS:.metadata.namespace,NAME:.metadata.name,SIZE:.spec.resources.requests.storage,SC:.spec.storageClassName

```
