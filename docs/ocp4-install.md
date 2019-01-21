## OpenShift 4.0 install

### Note

* This is very 'work in progress', so it is not stable as you expected, as this document
* Currently HA is not supported yet

### Prerequisites

* need to have pull secret file from https://try.openshift.com
 
### Simple howto

```
# git pull https://github.com/redhat-nfvpe/base-infra-bootstrap
# cd base-infra-bootstrap
# cp inventory/examples/ocp4-ansible.inventory.yml ./inventory/ocp4-ansible.inventory.yml
# vi ./inventory/ocp4-ansible.inventory.yml
  > change the inventory as you want, e.g. 'ocp4_pull_secret_file'
# ansible-playbook -i inventory/ocp4-ansible.inventory.yml playbooks/virt-host-setup.yml
  > this ansible creates required files, as '~/openshift-ansible/...',
  > '{{ocp4_installer_bootstrap_ign_dir}}/bootstrap.ign' and '~/install-config.yaml'

# cd ../openshift-ansible
# ansible-playbook -i ./inventory/ocp4-vms-inventory.ini playbooks/byo/rhel_subscribe.yml
# ansible-playbook -i ./inventory/ocp4-vms-inventory.ini playbooks/deploy_cluster_40.yml
  > Note: During "Wait for core operators to appear and complete" message,
  > you need to run ./ocp4-change-dns.sh in another terminal to switch DNS entry.
```
