# Ansible Role: buildserver

Installs and configured a buildserver based on the Admin Workstation OVA.

## Requirements

No special requirements.

## Role Variables

Available variables are listed below, along with default values (see `defaults/main.yml`):

```
ssh_auth_key: "" # content of the ssh pub key "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKt ub@machine"
workdir: "~/buildserver"
```

Basic parameters.

```
# build server specific below until end of file
buildserver_install: true
buildserver_name: ""
buildserver_gcpsa: ""
buildserver_sakeyfolder: "" # folder where the sa keys are copied to
```

Toggle install/uninstall, VM name, component access Google Service Account (GSA), and GSA destination folder.

```
# vSphere/vCenter
buildserver_vc_fqdn: '{{ lookup("env", "VMWARE_HOST") }}'
buildserver_vc_username: '{{ lookup("env", "VMWARE_USER") }}'
buildserver_vc_password: '{{ lookup("env", "VMWARE_PASSWORD") }}'
buildserver_vc_validate_cert: false
buildserver_vc_datacenter: ""
buildserver_vc_datastore: ""
buildserver_vc_cluster: ""
buildserver_vc_network: "" # VM Network
buildserver_vc_folder: "" # optional
buildserver_vc_respool: "" # if default resourcePool use <buildserver_vc_cluster>/Resources
```

Settings to access vSphere/vCenter, including object names.

```
# Networking
buildserver_nw_ipallocmode: "" # dhcp or static
buildserver_nw_ip: "" # IP address of VM
buildserver_nw_gw: "" # gateway
buildserver_nw_cidr: "" # CIDR 24 or similar
buildserver_nw_dns: [""]
buildserver_ntp: "" # only one can be set default ntp.ubuntu.com
```

Network settings.

```
# upgrade
buildserver_upgrade: false
buildserver_upgrade_version: "" # version to update do (matches adminws version i.e 1.12.0-gke.446)
```

Toggle upgrade and target version.

## Dependencies

None.

## Example Playbook

```
# Install the Build Server
- name: "[buildserver] Installation"
  hosts: all
  connection: local # comment out if Ansible runs directly on the ansible runner
  gather_facts: False
  roles:
  - buildserver
```

## **License**

Copyright 2022 Google LLC. This software is provided as-is, without warranty or representation for any use or purpose.
Your use of it is subject to your agreement with Google.
