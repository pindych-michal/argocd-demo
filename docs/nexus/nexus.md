
# Playbook execution

```

Playbook execution

devsecopsadmin@registry:~/ansible-nexus$ pwd
/home/devsecopsadmin/ansible-nexus

ansible-playbook -i inventory/hosts.yml playbooks/configure-nexus.yml -vvv

ansible-playbook -i inventory/hosts.yml playbooks/mirror-to-nexus.yml -vvv --check


// Synchronization
nsible-playbook sync-nexus.yaml -e ocp_channel=stable-4.22 -e ocp_min_version=4.22.0 -e ocp_max_version=4.22.0


```



# OC Mirror Synchronization 
