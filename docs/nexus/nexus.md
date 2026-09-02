# OC mirror playbook preparation 


```
1. Pull secret generation for public registry (~/pull-secret.json )
console.redhat.com/openshift/install/pull-secret  
- cloud.openshift.com
- quay.io
- registry.connect.redhat.com
- registry.redhat.io

2. Then we must add to this file nexus secret


cd ~
read -rsp 'Haslo uzytkownika ocp-mirror: ' NEXUS_PASS; echo
NEXUS_AUTH=$(printf '%s' "ocp-mirror:${NEXUS_PASS}" | base64 -w0)
unset NEXUS_PASS

jq --arg reg 'registry.m1.local.net:5000' --arg auth "$NEXUS_AUTH" \
   '.auths[$reg] = {"auth": $auth}' \
   pull-secret.json > pull-secret-merged.json

unset NEXUS_AUTH
chmod 600 pull-secret-merged.json





So we have one faille which will be used to authenticate to public registry and local registry 

```




# Playbook execution

```

Playbook execution

devsecopsadmin@registry:~/ansible-nexus$ pwd
/home/devsecopsadmin/ansible-nexus

ansible-playbook -i inventory/hosts.yml playbooks/configure-nexus.yml -vvv

ansible-playbook -i inventory/hosts.yml playbooks/mirror-to-nexus.yml -vvv --check


// Synchronization releases 
ansible-playbook -i inventory/hosts.yml playbooks/mirror-to-nexus.yml -e ocp_channel=stable-4.22 -e ocp_min_version=4.22.10 -e ocp_max_version=4.22.10 -vvv --check

// Synchronization operators 
ansible-playbook sync-nexus.yaml -e mirror_content=operators -e ocp_channel=stable-4.22 \
  -e '{"mirror_operator_packages":[{"name":"kubevirt-hyperconverged","channels":[{"name":"stable"}]},{"name":"openshift-gitops-operator","channels":[{"name":"latest"}]},{"name":"lvms-operator","channels":[{"name":"stable-4.22"}]}]}'


```



# OC Mirror Synchronization 
