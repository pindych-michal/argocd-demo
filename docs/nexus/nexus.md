
# Playbook execution

```

Playbook execution

devsecopsadmin@registry:~/ansible-nexus$ pwd
/home/devsecopsadmin/ansible-nexus

ansible-playbook -i inventory/hosts.yml playbooks/configure-nexus.yml -vvv

ansible-playbook -i inventory/hosts.yml playbooks/mirror-to-nexus.yml -vvv --check


// Synchronization
nsible-playbook sync-nexus.yaml -e ocp_channel=stable-4.22 -e ocp_min_version=4.22.0 -e ocp_max_version=4.22.0

ansible-playbook sync-nexus.yaml -e mirror_content=operators -e ocp_channel=stable-4.22 \
  -e '{"mirror_operator_packages":[{"name":"kubevirt-hyperconverged","channels":[{"name":"stable"}]},{"name":"openshift-gitops-operator","channels":[{"name":"latest"}]},{"name":"lvms-operator","channels":[{"name":"stable-4.22"}]}]}'


```



# OC Mirror Synchronization 
