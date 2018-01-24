## openshift-ansible-bootstrap

Virtual and baremetal system bootstrap prior to deployment of OpenShift via the
openshift-ansible playbooks.

## Usage

### Quickstart

* Install galaxy roles with `ansible-galaxy install -r requirements.yml`
* Copy and modify example inventories in `./inventory/example_virtual/`
* Run these three playbooks:

```
ansible-playbook -i ./inventory/your.inventory virt-host-setup.yml
ansible-playbook -i ./inventory/your.inventory bootstrap.yml
```

* Log into your virtualization host and run the OpenShift-Ansible playbooks

More details follow for information on each step.

### Step 0: Download Ansible Galaxy roles

Install role dependencies with `ansible-galaxy`.

```
ansible-galaxy install -r requirements.yml
```

### Step 1: Create new inventory directory from examples

In the `inventory/` directory exists both an `example_virtual` and
`example_baremetal` directory which provides an example configuration for both
virtual and baremetal deployments.

A virtual deployment will instantiate a new virtual environment on your virtual
host and setup the bridge interface.

For a baremetal deployment, significantly less pre-deployment work needs to be
done as it is assumed your baremetal nodes have had their operating system and
partitioning done ahead of time and are ready on the network for bootstrapping.

## Step 2: Pre-deployment configuration (virtual and baremetal)

Copy the contents of the `inventory/example_virtual/` or
`inventory/example_baremetal` directory into a new environment directory:

```
cp -r inventory/example_virtual/ inventory/testing/
```

If performing a virtual deployment, modify the
`./inventory/testing/openshift-ansible.inventory` to set the virtual host IP
address. If you have local DNS setup, you can also use the virthost's hostname.

For a virtual deployment, we'll address the OpenShift master and minion node IP
addresses after our initial deployment (we haven't built the nodes yet, so
don't know their IP addresses). You may be able to use DNS hostnames if those
will resolve correctly for you.

For a baremetal deployment, update the OpenShift master and minion node IP
addresses now (or set their locally resolving hostnames).

Next, we need to address the variables in the `group_vars/` directory. There
are three files you need to modify:

* `all.yml`
* `openshiftnodes.yml`
* `virthosts.yml`

> **NOTE**
>
> For a baremetal deployment, you'll likely only need to modify the
> `openshiftnodes.yml` file.

### `all.yml`

You'll likely not need to do anything to the `all.yml` file, but if you'd like
to pass a different virtual disk device name or change the SSH common args that
Ansible will use, you can do it here.

### `virthosts.yml`

The `virthosts.yml` file contains information about what virtual machines we're
going to create, the bridge network that will get created on the virtual host,
and the virtual machine parameters. It also contains the source of the virtual
machine qcow2 image and the virthost paths of where to store the base image and
the virtual machine images at instantiation.

Pay particular attention to the `bridge_network_cidr` (should match your LAN,
to have VMs on the LAN [as opposed to NAT'ed]).

### `openshiftnodes.yml`

Primarily you only need to update the `ansible_ssh_private_key_file` variable
which contains the path to your private key for accessing the nodes. If you're
not running this from the virthost directly, this would be the key created
during the playbook run. You'll need to copy it to your Ansible control host
for further connections.

> **PRO TIP**
>
> If you're running the deployment from a remote control machine that isn't the
> virtual host, then you'll want to add an `ansible_ssh_common_args` line that
> provides a method of creating an SSH tunnel to the nodes via the virtual
> host. You can do this by adding a line like the following:
> ```
> ansible_ssh_common_args: '-o ProxyCommand="ssh -W %h:%p root@virthost.management.local"'
> ```

> **NOTE**
>
> If you're performing a baremetal deployment, skip down to the **Baremetal
> deployment** section.

### Step 3: Executing the virtual deployment

There are 3 playbooks you'll need to run to configure the entire setup:

* `virt-host-setup.yml`
* `bootstrap.yml`

The `virt-host-setup.yml` will get the virtual host setup and ready to deploy our virtual machines and instantiate the virtual machines, create storage
disks, and attach them to the virtual machines via KVM. If you need to remove
the virtual machines and their storage (say in the case you want to destroy and
re-instantiate a clean environment), you can run the `vm-teardown.yml`
playbook.

After all your virtual machines are instantiated, you can then run the
`bootstrap.yml` playbook against the new OpenShift virtual machine nodes.

So now it's time to run the virtual host deployment.

> **PRO TIP**
>
> If your virtual host already has the networking configuration setup the way
> that you want, you can skip the bridge network configuration, set the
> `bridge_networking` value to `false`.

```
ansible-playbook -i inventory/virtual_testing/ virt-host-setup.yml
```

This deployment of the the virtual host has also resulted in the 
instantiation of our virtual machines for the OpenShift master and minions.

If all of that has gone well, we should be able to bootstrap the nodes and get them ready for an OpenShift deployment. The bootstrap process will setup Docker and get the thinpool ready for persistent storage via the `direct-lvm`
configuration instead of the default `loopback` storage.

You can now jump down to the end and read the **Ready to go!** section.

## Step 4: Baremetal Deployment

A baremetal deployment is significantly simpler, since it's assumed you've done
some of the hard work ahead of time. There are a couple of assumptions prior to
running this for a baremetal deployment to be aware of.

* You've deployed a CentOS 7 operating system to your baremetal nodes
* Your LVM thinpool has been created ahead of time
* You've added a correct `docker-thinpool` file to the LVM configuration
  directory

There are two main sections you'll need in your Kickstart file (or you can
follow along with the Kickstart file and configure your disks the same way
through the graphical interface):

1. The partitioning configuration for the disk
2. The thinpool configuration for LVM

### Step 5: Partitioning layout

First, we need to create our partitioning layout. OpenShift has some pretty
specific tests about size of the partitions and mounts, and they are slightly
different for the OpenShift Master and Minion.

* Master
  * `/var/` must be 40G
  * `/tmp/` must be 1G
* Minion
  * `/var/` must be 15G
  * `/tmp/` must be 1G

#### (Optional) Kickstart file snippet for master partitioning layout

Kickstart file snippet for the partitioning layout. We've done the following:

* Use disk `sda` and clear the partitioning info and master boot record
* Create `/boot/` at 500MB
* Create swap with the recommended size (usually matches RAM value)
* Create 2 physical volumes with LVM
  * `pv.01` at a size of 40+8+2 GB for each of our logical volumes, plus 1GB
    extra to grow `/var/` into.
  * `pv.02` at a size of 10GB, growing to finish filling the disk
* Create 2 volume groups against the physical volumes
  * `vg_system` to hold the system logical volumes
  * `vg_docker` to hold our thinpool
* Create logical volumes on `vg_system`
  * `/` has a size of 8GB
  * `/var/` has a size of 40960, growing into the extra 1GB to avoid boundary
    issues
  * `/tmp` has a size of 2GB to have plenty of extra space
* Create thinpool logical volumes, unmounted, on `vg_docker`

```
# System bootloader configuration
zerombr
clearpart --drives=sda --all --initlabel
part /boot --fstype ext4 --size=500
part swap --recommended

# create physical volumes
part pv.01 --size=52224 --ondisk=sda
part pv.02 --fstype="lvmpv" --size=10240 --grow --ondisk=sda

# create volume groups
volgroup vg_system pv.01
volgroup vg_docker pv.02

# create logical volumes
logvol / --vgname=vg_system  --fstype=ext4  --size=8192 --name=lv_root
logvol /var --vgname=vg_system --fstype=ext4 --size=40960 --grow --name=lv_var
logvol /tmp --vgname=vg_system --fstype=ext4 --size=2048 --name=lv_tmp

logvol none fstype=none --vgname=vg_docker --thinpool --percent=80 --grow --name=thinpool --metadatasize=1000 --chunksize=512 --profile=docker-thinpool

bootloader --append=" crashkernel=auto" --location=mbr --boot-drive=sda
```

The other snippet we need to add is to the `%post` section of our Kickstart
file. This will allow LVM to properly mount and define the thinpool based on
our values for data and meta information stores.

```
# Docker LVM thinpool profile
cat << EOF > /etc/lvm/profile/docker-thinpool.profile
activation {
    thin_pool_autoextend_threshold=80
    thin_pool_autoextend_percent=20
}
EOF
```

### Executing the baremetal deployment

With our baremetal nodes configured and the partitioning all dealt with, we can
simply execute the `bootstrap.yml` playbook against our nodes.

```
ansible-playbook -i inventory/testing/ bootstrap.yml
```

# Ready to go!

At this point all your nodes and storage configuration should be ready to go.
You can then move onto executing an OpenShift deployment against your freshly
bootstrapped environment.

More information about deploying OpenShift can be found on Doug's blog at 
http://dougbtv.com//nfvpe/2017/07/18/openshift-ansible-lab-byo/
