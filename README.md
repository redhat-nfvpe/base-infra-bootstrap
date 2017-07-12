## openshift-ansible-bootstrap

A way to bootstrap my nodes before an openshift-ansible install

## Usage

### Step 0: Modify the inventory 

Modify the `./inventory/inventory` to meet your needs.

Pay particular attention to:

* The host IP addresses
* The `bridge_network_cidr` (should match your LAN, to have VMs on the LAN [as opposed to NAT'ed])
* 

### Step 1: Attach disks to VMs

Note that the the list of `virtual_machines` in the inventory are the names that `virsh` sees in order to attach disks ()

    ansible-playbook -i inventory/inventory vm-disks.yaml

### Step 2: Bootstrap openshift nodes with Docker setup

**NOTE**: This **will remove all Docker everything!**

     ansible-playbook -i inventory/inventory bootstrap.yml



