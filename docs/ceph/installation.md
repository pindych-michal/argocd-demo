

# Architecture

```
-------------------------------proxmox pve
ceph server (EL9 + cephadm)
pool ocp-rbd
client.ocp-rdb
--------------------------------
|
|
|
LAN
3300/6789
6800-7300
|
|
|
------------------------------proxmox mgt
VM: SNO Openshfit
ceph-csi (RBD) -> SC ceph-rbd
pod with PVC/ Opnshift Virt
-----------------------------


Native storage types:

1. Block – RBD (RADOS Block Device)
Single writer (RWO); RWX only in raw Block mode

2. File – CephFS (Ceph File System)
POSIX shared filesystem; requires MDS (Metadata Server) daemons;

3. Object – RGW (RADOS Gateway, “Ceph Object Gateway”)
HTTP REST gateway speaking S3 – apps talk S3 directly.

Sumamry:
RBD for RWO and VMs, CephFS for RWX, RGW for backups/registry/observability
```


# Deployment 

```

1. Ceph server installation 
2. Networking preparation
3. Openshift configuration
4. Workload/testing

clusterID -
MON - 192.168.200.66 (entry point, authentication, infra map)
user ID + userKey - credential's to access the ceph pool
pool - where to store the images


```
