

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

```
