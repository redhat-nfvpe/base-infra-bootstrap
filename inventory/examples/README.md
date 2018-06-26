# Example Scenarios

Here we've got example scenarios, with our favorite recipes.

---

## Multus CNI

Location: `inventory/examples/multus`

This guy will load up a multus CNI integration with openshift-ansible by setting up a forked clone of openshift-ansible.

Example run:

```
$ ansible-playbook -i inventory/examples/multus/inventory.yml playbooks/vm-teardown.yml playbooks/virt-host-setup.yml 
$ ansible-playbook -i inventory/vms.local.generated playbooks/bootstrap.yml
```



