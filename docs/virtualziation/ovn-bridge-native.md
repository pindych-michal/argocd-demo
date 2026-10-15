
```
# Wariant B (docelowy): dedykowany OVS bridge na ens19 zamiast linux-bridge br1.
# UWAGA: ens19 może należeć tylko do jednego bridge'a. Zanim to nałożysz, ustaw w NNCP br1-ens19
# interfejs br1 na "state: absent" (VM-ki na NAD vm-lan stracą sieć) i zmień w 10-nncp-ovn-localnet-vmlan.yaml
# bridge: br-ex -> bridge: br-vm, albo nałóż ten plik zamiast tamtego.
apiVersion: nmstate.io/v1
kind: NodeNetworkConfigurationPolicy
metadata:
  name: ovn-localnet-vmlan
spec:
  desiredState:
    interfaces:
      - name: br-vm
        type: ovs-bridge
        state: up
        ipv4:
          enabled: false
        ipv6:
          enabled: false
        bridge:
          allow-extra-patch-ports: true   # OVN dokłada tu patch-porty, bez tego nmstate będzie je usuwać
          options:
            stp: false
          port:
            - name: ens19
    ovn:
      bridge-mappings:
        - localnet: vmlan
          bridge: br-vm
          state: present

```


```

# Sieć localnet OVN-Kubernetes dla VM-ek. CUDN sam tworzy NAD "vm-lan-ovn" w każdym namespace
# pasującym do selektora. W VM: spec.template.spec.networks[].multus.networkName: vm-lan-ovn
#  - physicalNetworkName musi być równe "localnet" z bridge-mappings w NNCP (vmlan)
#  - ipam Disabled: adresy statycznie z cloud-init (jak na NAD vm-lan); alternatywa: subnets + DHCP z OVN
#  - vlan: brak (vmbr0 jest untagged); dla VLAN-u dodaj  vlan: {mode: Access, access: {id: 10}}
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: vm-lan-ovn
spec:
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: vm-deployment-test
  network:
    topology: Localnet
    localnet:
      role: Secondary
      physicalNetworkName: vmlan
      ipam:
        mode: Disabled

```



```
# Sieć localnet OVN-Kubernetes dla VM-ek. CUDN sam tworzy NAD "vm-lan-ovn" w każdym namespace
# pasującym do selektora. W VM: spec.template.spec.networks[].multus.networkName: vm-lan-ovn
#  - physicalNetworkName musi być równe "localnet" z bridge-mappings w NNCP (vmlan)
#  - ipam Disabled: adresy statycznie z cloud-init (jak na NAD vm-lan); alternatywa: subnets + DHCP z OVN
#  - vlan: brak (vmbr0 jest untagged); dla VLAN-u dodaj  vlan: {mode: Access, access: {id: 10}}
apiVersion: k8s.ovn.org/v1
kind: ClusterUserDefinedNetwork
metadata:
  name: vm-lan-ovn
spec:
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: vm-deployment-test
  network:
    topology: Localnet
    localnet:
      role: Secondary
      physicalNetworkName: vmlan
      ipam:
        mode: Disabled





```
