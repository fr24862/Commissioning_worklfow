# Commissioning Workflows

An Ansible workflow for validating and provisioning Linux virtual machines from VMware templates.

## Current Workflow

`playbooks/01_vm_build_skeleton.yml` performs the following steps:

- Validates the requested build action and required VM variables.
- Confirms forward and reverse DNS records match the requested hostname and IP address.
- Checks that the proposed IP address does not respond to ping.
- Confirms the target folder, source template, and datastore exist in vCenter.
- Refuses to overwrite an existing VM.
- Prints a validated build plan by default.
- Optionally clones and configures the VM, verifies it, and powers it on.

Operating-system commissioning stages such as storage configuration, directory integration, monitoring registration, security tooling, and final health checks are currently placeholders in the playbook.

## Prerequisites

- Ansible Core installed on the controller.
- The following Ansible collections:

  ```bash
  ansible-galaxy collection install community.general community.vmware ansible.utils
  ```

- Network access from the controller to vCenter, DNS, and the proposed VM IP.
- A VMware-capable Python environment. The playbook currently expects:
  `/home/fr24862a/.venvs/ansible-vmware/bin/python`

Update `ansible_python_interpreter` in the playbook or inventory when that path differs on your controller.

## Inventory

Create `inventory/build_hosts.yml` with one host per VM. The host name should be the requested FQDN. Each host must define the variables below:

```yaml
all:
  hosts:
    app01.example.com:
      change_control_id: CHG000000
      build_environment: dev       # dev, test, prod, or dr
      vm_fqdn: app01.example.com
      vm_domain: example.com
        vm_ip_address: 192.0.2.10
      vm_netmask: 255.255.255.0
      vm_gateway: 192.0.2.1
      vm_dns_servers:
        - 192.0.2.53
      vm_vcenter: vcenter.example.com
      vm_datacenter: DC01
      vm_cluster: Compute01
      vm_folder: /DC01/vm/Applications
      vm_template: rhel9-template
      vm_datastore: datastore01
      vm_network: VM Network
      vm_cpus: 4
      vm_memory_mb: 8192
      vm_additional_disks: []
```

Replace the example values before running the playbook. The `vm_ip_address` example is documentation-only and must be changed to a real address.

## Running the Workflow

Run plan mode first. This is the default and does not create a VM:

```bash
ansible-playbook \
  -i inventory/build_hosts.yml \
  playbooks/01_vm_build_skeleton.yml
```

Creation requires both an explicit action and a confirmation token:

```bash
ansible-playbook \
  -i inventory/build_hosts.yml \
  playbooks/01_vm_build_skeleton.yml \
  -e build_action=create \
  -e confirm_create=CREATE-VM
```

The playbook prompts for the vCenter username and password. Credentials are not stored in this repository.

## Safety Controls

- `build_action` defaults to `plan`.
- Creation is rejected unless `confirm_create=CREATE-VM` is supplied.
- VM creation starts in a powered-off state.
- `power_on_after_build` defaults to `false`.
- Existing VM names and responding IP addresses are rejected.
- TLS certificate validation is currently disabled by default through `validate_certs: false`; enable it for environments with trusted vCenter certificates.

Review the plan output and confirm IPAM, DNS, vCenter placement, template, datastore, and change-control details before using create mode.

## Useful Tags

Run a focused portion of the workflow with tags such as:

```bash
ansible-playbook -i inventory/build_hosts.yml playbooks/01_vm_build_skeleton.yml --tags check_dns
ansible-playbook -i inventory/build_hosts.yml playbooks/01_vm_build_skeleton.yml --tags check_vmware
ansible-playbook -i inventory/build_hosts.yml playbooks/01_vm_build_skeleton.yml --tags plan
```

Available tag groups include `checks`, `check_dns`, `check_ip`, `check_vmware`, `check_capacity`, `plan`, `build`, `verify`, `power_on`, and `post_build`.

## Project Layout

```text
inventory/                       # Local inventory files
inventory/group_vars/all.yml     # Shared workflow and derived variables
inventory/host_vars/              # Per-VM commissioning variables
playbooks/01_vm_build_skeleton.yml
roles/                           # Reserved for future commissioning roles
var/                             # Reserved for workflow variables
```