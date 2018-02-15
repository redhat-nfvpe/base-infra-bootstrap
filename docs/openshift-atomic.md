## OpenShift install on CentOS Atomic Host

If you've already run the virt-host-setup playbook, you'll first need to remove (or move) the existing CentOS image to download a new one. (Or change the variables to otherwise put it in a new place, if you so please). Go ahead and move the CentOS cloud image...

```bash
$ cd /home/images/
$ ls -lh CentOS-7-x86_64-GenericCloud.qcow2 
-rw-r--r--. 1 root root 838M Feb  6 19:35 CentOS-7-x86_64-GenericCloud.qcow2
$ mv CentOS-7-x86_64-GenericCloud.qcow2 not.atomic.CentOS-7-x86_64-GenericCloud.qcow2
```

Now, run the virt-host-setup again. This time patch in some "extra vars" to change a few items.

Here's an inventory and extra vars, with `virthost.inventory` looking like:

```ini
virt_host ansible_host=192.168.1.42 ansible_ssh_user=root

[virthosts]
virt_host

[all:vars]
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Then have `./inventory/atomic_example.extra-vars.yml` setup like so:

```yaml
---
centos_genericcloud_url: http://cloud.centos.org/centos/7/atomic/images/CentOS-Atomic-Host-7-GenericCloud.qcow2
image_destination_name: CentOS-7-x86_64-GenericCloud.qcow2
host_type: "atomic"
```

Then go ahead and run the virt-host-setup playbook using that inventory and extra-vars...

```bash
ansible-playbook -i inventory/virthost.inventory -e "@./inventory/atomic_example.extra-vars.yml" virt-host-setup.yml
```

Run the bootstrap:

```bash
ansible-playbook -i inventory/vms.local.generated -e "@./inventory/atomic_example.extra-vars.yml" bootstrap.yml
```

You'll then setup an inventory for openshift-ansible, an example looks like (make sure you change out each of the `ansible_host` for your own, of course):

```ini
openshift-master ansible_host=192.168.122.119
openshift-node-1 ansible_host=192.168.122.70
openshift-node-2 ansible_host=192.168.122.133

[OSEv3:children]
masters
nodes
etcd

[OSEv3:vars]
ansible_ssh_user=centos
ansible_become=yes
debug_level=2
ansible_ssh_private_key_file=/root/.ssh/id_vm_rsa
openshift_master_unsupported_embedded_etcd=true 
openshift_disable_check=disk_availability,memory_availability,docker_image_availability
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]
openshift_deployment_type=origin 
containerized=true
openshift_release=3.9
openshift_image_tag=latest
enable_excluders=false
# new
openshift_hostname_check=false

[masters]
openshift-master

[etcd]
openshift-master

[nodes]
openshift-master openshift_node_labels="{'region': 'infra', 'zone': 'default'}" openshift_schedulable=true
openshift-node-[1:2] openshift_node_labels="{'region': 'primary', 'zone': 'default'}"
```

And running from your virthost, issue the openshift-ansible playbook commands like so:

```bash
$ ansible-playbook -i atomic.inventory playbooks/prerequisites.yml && ansible-playbook -i atomic.inventory playbooks/deploy_cluster.yml
```
