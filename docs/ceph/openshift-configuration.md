
# 0. Introduction 


```

1. Install the operator (with SCC for OpenShift)
1a. Patch the SCC ClusterRole (mandatory, manifest bug)
2. Deploy the RBD driver
3. Connection to Ceph
4. Secret, StorageClass, VolumeSnapshotClass
4a. Connectivity test without a PVC
5. End-to-end test
6. Clean up the test

```


```
1. Install the operator (with SCC for OpenShift)
Put the management layer in place: the Ceph-CSI Operator is a Kubernetes controller that will later build and maintain the actual CSI driver pods for you.
What is deployed:
- namespace ceph-csi-operator-system — operator and drivers live here
- 6 CustomResourceDefinitions in API group csi.ceph.io/v1 (Driver, CephConnection, ClientProfile, ClientProfileMapping, ClientProfileReplication, OperatorConfig)
- 9 ServiceAccounts
- Deployment ceph-csi-operator-controller-manager — the only running code; a small, non-privileged pod with a reconcile loop
- OpenShift-only part: SecurityContextConstraints ceph-csi-operator-ceph-csi-scc
- a NetworkPolicy isolating the operator pod.

Note: installed from YAML, not through OLM

1a. Patch the SCC ClusterRole (mandatory, manifest bug)
Make the SCC actually usable. In v1.1.0



2. Deploy the RBD driver
Turn the operator into something useful: ask it for an RBD driver instance. This is the moment the actual CSI components appear in the cluster.
One object — Driver rbd.csi.ceph.com with controllerPlugin.replicas: 1 (single node). The object's name is the CSI driver name and must match provisioner: in the StorageClass later.

What is deployed:
- Deployment rbd.csi.ceph.com-ctrlplugin — 5 containers: the Ceph plugin plus the CSI sidecars provisioner, attacher, resizer, snapshotter. Runs on the pod network. Handles control operations: rbd create / rm / resize / snap / clone
- DaemonSet rbd.csi.ceph.com-nodeplugin — 2 containers (plugin + driver-registrar), one pod per node
- CSIDriver rbd.csi.ceph.com — registers the driver with kubelet;
- ConfigMap ceph-csi-config




3. Connection to Ceph
Give the driver the address of the Ceph cluster and a name under which StorageClasses will refer to it. Despite the name, no network connection is made here — this is pure configuration.

What is deployed:
- CephConnection ceph1 — spec.monitors: [192.168.200.66:6789]
- ClientProfile ceph1 — cephConnectionRef: ceph1




4. Secret, StorageClass, VolumeSnapshotClass
Supply the identity the driver will use against Ceph, and expose Ceph to cluster users as an ordinary storage class. After this sub-step anyone can request Ceph-backed storage with a plain PVC.

What is deployed:
- Secret csi-rbd-secret — userID: ocp-rbd, userKey: AQCY…
- StorageClass ceph-rbd
- VolumeSnapshotClass ceph-rbd-snap

4a. Connectivity test without a PVC
First genuine contact with Ceph from inside OpenShift, before involving the provisioning machinery.


5. End-to-end test
Exercise the whole chain: PVC → provisioner creates an RBD image and a PV → pod is scheduled → node plugin maps the image into the kernel, formats and mounts it → application writes. Then look at the same thing from the Ceph side.

What is deployed:
- PVC ceph-rbd-fs (2Gi, RWO, Filesystem)
- PVC ceph-rbd-block (2Gi, RWO, volumeMode: Block)
- Pod ceph-writer (ubi9-minimal) appending the date to /data/log.txt every 5 s;


6. Clean up the test
Remove the proof, keep the platform. oc delete of the snapshot manifest and the test namespace; because the StorageClass has reclaimPolicy Delete, the driver removes the RBD images and snapshots on Ceph by itself.




```

# 1. Install the operator (with SCC for OpenShift)

```
mkdir -p ~/ceph-lab && cd ~/ceph-lab
OP=v1.1.0
oc create -f https://raw.githubusercontent.com/ceph/ceph-csi-operator/$OP/deploy/all-in-one/install-openshift.yaml
# gdyby tag nie miał tego pliku (404): zamień $OP na main

oc -n ceph-csi-operator-system get pods -w                   # ceph-csi-operator-controller-manager-…  1/1 Running
oc get scc ceph-csi-operator-ceph-csi-scc                    # SCC z instalatora (nazwa z prefiksem, inaczej niż w docs)
oc get clusterrolebinding | grep -c 'ceph-csi-operator-ceph-csi-.*-scc'   # 8 — po jednym na SA każdego sterownika
```

## 1a. Patch the SCC ClusterRole (mandatory, manifest bug)

```
oc patch clusterrole ceph-csi-operator-ceph-csi-scc-user --type=json \
  -p='[{"op":"replace","path":"/rules/0/resourceNames","value":["ceph-csi-operator-ceph-csi-scc","ceph-csi-scc"]}]'
oc auth can-i use securitycontextconstraints/ceph-csi-operator-ceph-csi-scc \
  --as=system:serviceaccount:ceph-csi-operator-system:ceph-csi-operator-rbd-nodeplugin-sa      # yes
```

# 2. Deploy the RBD driver

```
cat > ocp-driver-rbd.yaml <<'EOF'
apiVersion: csi.ceph.io/v1
kind: Driver
metadata:
  name: rbd.csi.ceph.com
  namespace: ceph-csi-operator-system
spec:
  controllerPlugin:
    replicas: 1          # SNO; domyślnie 2
EOF
oc create -f ocp-driver-rbd.yaml

oc -n ceph-csi-operator-system get pods -w
# rbd.csi.ceph.com-ctrlplugin-…   5/5 Running
# rbd.csi.ceph.com-nodeplugin-…   2/2 Running   (DaemonSet, hostNetwork)
oc get csidriver rbd.csi.ceph.com

```

```
Please refer to list of all supported ceph's drivers at this moment:

- rbd.csi.ceph.com - block/RWO
- cephfs.csi.ceph.com - file/RWX
- nfs.csi.ceph.com - nfs/RWX
- nvmeof.csi.ceph.com NVMe/TCP/RWO

```

3. Connection to Ceph


```

cat > ocp-ceph-connection.yaml <<'EOF'
apiVersion: csi.ceph.io/v1
kind: CephConnection
metadata:
  name: ceph1
  namespace: ceph-csi-operator-system
spec:
  monitors:
    - 192.168.200.66:6789
---
apiVersion: csi.ceph.io/v1
kind: ClientProfile
metadata:
  name: ceph1                      # = clusterID w StorageClass i VolumeSnapshotClass
  namespace: ceph-csi-operator-system
spec:
  cephConnectionRef:
    name: ceph1
EOF
oc create -f ocp-ceph-connection.yaml
oc -n ceph-csi-operator-system get cephconnection,clientprofile
oc -n ceph-csi-operator-system get cm ceph-csi-config -o jsonpath='{.data.config\.json}{"\n"}'
# → [{"clusterID":"ceph1","monitors":["192.168.200.66:6789"], …}]

```

# 4. Secret, StorageClass, VolumeSnapshotClass

```

cat > ocp-rbd.yaml <<'EOF'
apiVersion: v1
kind: Secret
metadata:
  name: csi-rbd-secret
  namespace: ceph-csi-operator-system
stringData:
  userID: ocp-rbd
  userKey: AQCY3+lq2uDsKBAAzb0NDw4Td/5rA+wXGstVSg==
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-rbd
  annotations:
    storageclass.kubevirt.io/is-default-virt-class: "true"   # krok 3: domyślna klasa dla dysków VM; uruchamia też import golden images (sekcja 5)
provisioner: rbd.csi.ceph.com
parameters:
  clusterID: ceph1
  pool: ocp-rbd
  imageFeatures: layering
  csi.storage.k8s.io/fstype: ext4
  csi.storage.k8s.io/provisioner-secret-name: csi-rbd-secret
  csi.storage.k8s.io/provisioner-secret-namespace: ceph-csi-operator-system
  csi.storage.k8s.io/controller-expand-secret-name: csi-rbd-secret
  csi.storage.k8s.io/controller-expand-secret-namespace: ceph-csi-operator-system
  csi.storage.k8s.io/node-stage-secret-name: csi-rbd-secret
  csi.storage.k8s.io/node-stage-secret-namespace: ceph-csi-operator-system
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ceph-rbd-snap
driver: rbd.csi.ceph.com
parameters:
  clusterID: ceph1
  csi.storage.k8s.io/snapshotter-secret-name: csi-rbd-secret
  csi.storage.k8s.io/snapshotter-secret-namespace: ceph-csi-operator-system
deletionPolicy: Delete
EOF
oc create -f ocp-rbd.yaml
oc get sc,volumesnapshotclass              # lvms-vg1 (default) zostaje; ceph-rbd bez (default)




```
