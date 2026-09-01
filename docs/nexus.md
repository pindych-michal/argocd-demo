


```

Playbook execution

devsecopsadmin@registry:~/ansible-nexus$ pwd
/home/devsecopsadmin/ansible-nexus

ansible-playbook -i inventory/hosts.yml playbooks/configure-nexus.yml -vvv

ansible-playbook -i inventory/hosts.yml playbooks/mirror-to-nexus.yml -vvv --check

```
