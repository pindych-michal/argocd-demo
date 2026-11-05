

# Architecture

```
-------------------------------proxmox pve
ceph server (EL9 + cephadm)
pool ocp-rbd
client.ocp-rdb
MON MGR(dashboard) ODS.0 OSD.1 OSD.2
sda=OS  sdb/sdc/sdd = 3*100 GB (local-lvm)

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
OSD - raw disks entries


```


## VM on proxmox 

```
#proxmox shell 

VMID=200 ; VMNAME=ceph1 ; STORAGE=local-lvm ; BRIDGE=vmbr0            
IP=10.10.100.50/24 ; GW=10.10.100.1 ; DNS=10.10.100.1 ; DOMAIN=local.net  
IMG=Rocky-9-GenericCloud-Base.latest.x86_64.qcow2

pvesm status                      # $STORAGE: potrzeba ~340 GB wolnego (thin); gdy mniej → dyski OSD po 60 GB
[ -f ~/.ssh/id_ed25519.pub ] || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
wget -N -P /var/lib/vz/template https://dl.rockylinux.org/pub/rocky/9/images/x86_64/$IMG

qm create $VMID --name $VMNAME --ostype l26 --cpu host --cores 4 --memory 12288 --balloon 0 \
  --net0 virtio,bridge=$BRIDGE,tag=$VLAN --scsihw virtio-scsi-single --agent enabled=1 --serial0 socket --onboot 1
qm disk import $VMID /var/lib/vz/template/$IMG $STORAGE
qm set $VMID --scsi0 $STORAGE:vm-$VMID-disk-0,discard=on,iothread=1,ssd=1 --boot order=scsi0
qm disk resize $VMID scsi0 40G
qm set $VMID --ide2 $STORAGE:cloudinit \
  --ciuser rocky --cipassword 'ZMIEN-haslo-konsoli' --sshkeys ~/.ssh/id_ed25519.pub \
  --ipconfig0 ip=$IP,gw=$GW --nameserver $DNS --searchdomain $DOMAIN
for i in 1 2 3; do qm set $VMID --scsi$i $STORAGE:100,discard=on,iothread=1,ssd=1; done    # dyski pod OSD
qm start $VMID
```


## System preparation

```
# proxmox shell
# ssh rocky@10.10.100.50
sudo -i
hostnamectl set-hostname ceph1
sed -i '/127\.0\.1\.1/d' /etc/hosts
echo "10.10.100.50 ceph1.local.net ceph1" >> /etc/hosts            # [ZMIEŃ]
touch /etc/cloud/cloud-init.disabled     # konfiguracja już wstrzyknięta; cloud-init nie ma nadpisywać hostname/hosts przy każdym boocie
dnf -y update
dnf -y install podman lvm2 chrony python3 firewalld qemu-guest-agent epel-release
systemctl enable --now chronyd firewalld qemu-guest-agent
firewall-cmd --permanent --add-service=ceph-mon --add-service=ceph && firewall-cmd --reload   # 3300, 6789, 6800-7300
chronyc tracking | head -3               # "Leap status : Normal" = czas zsynchronizowany
reboot
```


## Cephadm/bootsrtap 

```
# ssh rocky@10.10.100.50 ; sudo -i
CEPH_RELEASE=20.2.5             # najnowsze 20.2.x wg https://docs.ceph.com/en/latest/releases/
cd /root
curl --fail --silent --show-error --location --remote-name \
  https://download.ceph.com/rpm-${CEPH_RELEASE}/el9/noarch/cephadm
chmod +x cephadm
./cephadm version               # cephadm version 20.2.x (...) tentacle (stable)
./cephadm add-repo --release tentacle
./cephadm install               # pakiet cephadm w PATH
cephadm install ceph-common     # natywne ceph / rbd w VM (bez "cephadm shell")

cephadm bootstrap --mon-ip 10.10.100.50 \
  --single-host-defaults --skip-monitoring-stack \
  --initial-dashboard-user admin --initial-dashboard-password 'devsecops135!!!' --dashboard-password-noupdate

ceph -s
ceph orch host ls 

```

## OSD 

```
ceph orch device ls                            # sdb, sdc, sdd → AVAILABLE Yes
for d in sdb sdc sdd; do ceph orch daemon add osd ceph1:/dev/$d; done
sleep 60 ; ceph -s                             # 3 osds: 3 up, 3 in ; HEALTH_OK ; 1 pool (.mgr), 1 pg active+clean
ceph osd tree
```

## User and pool for Openshift 

```
ceph osd pool create ocp-rbd
rbd pool init ocp-rbd
ceph mon set auth_allowed_ciphers aes,aes256k
ceph auth get-or-create client.ocp-rbd --key-type aes \
  mon 'profile rbd' osd 'profile rbd pool=ocp-rbd' mgr 'profile rbd pool=ocp-rbd' \
  -o /etc/ceph/ceph.client.ocp-rbd.keyring
# trzy ostrzeżenia oczekiwane w labie (klucz aes dla OCP) — wyciszamy:
ceph health mute AUTH_INSECURE_KEYS_ALLOWED --sticky         # mon dopuszcza szyfr aes
ceph health mute AUTH_INSECURE_KEYS_CREATABLE --sticky       # mon pozwala tworzyć klucze aes
ceph health mute AUTH_INSECURE_CLIENT_KEY_TYPE --sticky      # istnieje klient z kluczem aes
ceph config set mon mon_allow_pool_delete true                # lab: żeby dało się sprzątać pule

# test tą samą tożsamością, której użyje provisioner:
rbd --id ocp-rbd create ocp-rbd/test --size 1G --image-feature layering
rbd --id ocp-rbd info ocp-rbd/test
rbd --id ocp-rbd rm ocp-rbd/test
ceph -s                                                       # HEALTH_OK (muted: …), 2 pools, 33 pgs active+clean

```

```
#data required on openshift side

echo "fsid:    $(ceph fsid)"
ceph mon dump 2>/dev/null | grep '^0:'        # 0: [v2:10.10.10.11:3300/0,v1:10.10.10.11:6789/0] mon.ceph1
echo "userID:  ocp-rbd"
echo "userKey: $(ceph auth get-key client.ocp-rbd)"   # klucz typu aes zaczyna się od "AQ"

```

## Snapshot of VM, additional testing 

```
# Proxmox 1 — punkt powrotu:
qm shutdown 200 --timeout 120 && qm snapshot 200 ceph-ready --description "Tentacle, pool ocp-rbd, client.ocp-rbd (aes)" && qm start 200
# powrót po nieudanym eksperymencie: qm stop 200 ; qm rollback 200 ceph-ready ; qm start 200
# (po rollbacku ssh zgłosi zmianę host key: ssh-keygen -R 10.10.10.11)

# Proxmox 2 (albo dowolny host w LAN) — czy MON jest osiągalny:
for p in 3300 6789; do timeout 3 bash -c "</dev/tcp/10.10.10.11/$p" && echo "port $p OK" || echo "port $p FAIL"; done
```
