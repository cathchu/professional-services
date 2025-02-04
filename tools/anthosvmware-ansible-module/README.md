# Ansible Automation for Anthos on VMware

Automated install of Anthos clusters with Ansible, including
templating YAML files with Ansible for Anthos on VMware.

## About this repository

This git repository uses GitHub Actions (see `.github/workflows/`).

- [https://github.com/marketplace/actions/ansible-lint](https://github.com/marketplace/actions/ansible-lint)
- [https://github.com/actionshub/markdownlint](https://github.com/actionshub/markdownlint)

### Markdown linting

- [https://github.com/actionshub/markdownlint](https://github.com/actionshub/markdownlint)
- [Markdown linting rules](https://github.com/markdownlint/markdownlint/blob/master/docs/RULES.md)

### Ansible linting

- [https://github.com/ansible/ansible-lint-action](https://github.com/ansible/ansible-lint-action)

## Assumptions

Consider these assumptions when you wonder how certain tasks are implemented or missing.

- Ansible is run from the command-line
- Ansible `>= 2.9`
- Greenfield deployment: No Anthos resources exist when applying the automation.
- Anthos preview features are not considered in this automation but can be added easily (`vim roles/*cluster/templates/{admin,user}-cluster.yaml.j2`)
- If `secretsEncryption` is enabled for the admin cluster, it will be enabled for user clusters;
  however, the keyversions are tracked in separate Ansible vars for rotation purposes
- One admin workstation per admin cluster

## Prerequisites

- Ansible
- Authenticate `gcloud` on jumphost with `gcloud auth login` so that Ansible can run the `gsutil` command on the jumphost
- vSphere: Create VM-Folder for Anthos VMs
- vSphere: Create folder on vSAN for Anthos Admin Cluster (if using vSAN).
  Consider using value of Ansible variable `{{ ac_name }}` as the vSAN folder name to be consistent.

Last tested with the following versions.

```shell
ansible --version
ansible [core 2.11.4]
[...]
  python version = 3.9.7 (default, Oct 13 2021, 06:45:31) [Clang 13.0.0 (clang-1300.0.29.3)]
  jinja version = 3.0.1
  libyaml = True
```

### Prerequisites for Google Service Accounts 

Google Service Accounts ("GSA") are required for the automated cluster and Anthos component lifecycle operations performed by Ansible:
- Create GCP service accounts (SA) as per documentation [Creating Service Accounts](https://cloud.google.com/anthos/clusters/docs/on-prem/how-to/service-accounts)
- Download each GCP SA JSON key file to your machine/jumphost

The below sections highlight the GSAs used in this automation code and their necessary permissions.

#### Component Access GSA (Required)

The Component Access service account downloads cluster artifacts from Google Container Registry (ex: images). It requires the below permissions in the target Google Cloud project:

* `serviceusage.serviceUsageViewer`
* `iam.roleViewer`
* `iam.serviceAccountViewer`

Set the JSON file key path with `ac_componentserviceaccount_keypath` and `uc_componentserviceaccount_keypath` within the target cluster inventories.

#### Connect Register GSA (Required)

The Connect Register service account is responsible for registering your admin and user clusters to a fleet in a GCP project. It requires the below permission in the target Google Cloud project:

* `gkehub.admin`

Set the JSON file key path with `ac_gkeconnect_registerserviceaccountkeypath` and `uc_gkeconnect_registerserviceaccountkeypath` within the target cluster inventories.

#### Logging and Monitoring GSA (Optional)

The Logging and Monitoring service account can be used to export logs and metrics from your clusters to Google Cloud Logging and Monitoring. It requires the below permissions in the target Google Cloud project:

* `stackdriver.resourceMetadata.writer`
* `opsconfigmonitoring.resourceMetadata.writer`
* `logging.logWriter`
* `monitoring.metricWriter`
* `monitoring.dashboardEditor`

Set the JSON file key path with `ac_stackdriver_serviceaccountkeypath` and `uc_stackdriver_serviceaccountkeypath` within the target cluster inventories. 

#### Audit Logging GSA (Optional)

The Audit Logging service account can be used to export Kubernetes audit logs from your clusters to Google Cloud Audit logs. It does not require any permissions to be granted to it. 

Set the JSON file key path with `ac_cloudauditlogging_serviceaccountkeypath` and `uc_cloudauditlogging_serviceaccountkeypath` within the target cluster inventories. 

#### Anthos Service Mesh Config GSA (Optional)

The Anthos Service Mesh Config service account is used when installing ASM onto user clusters. It requires the below permissions in the target Google Cloud project:

* `gkehub.editor`
* `meshconfig.admin`
* `iam.roleViewer`
* `serviceusage.serviceUsageViewer`

> **Note 1:** This list is a subset of permissions outlined in the [public docs for installing ASM](https://cloud.google.com/service-mesh/docs/installation-permissions), which lists all of the permissions required from cluster preparation for ASM installation to the actual installation. Since much of the platform is already configured by the time of installation, this service account can operate with a more restrictive permissions.
>
>Also notably, this list omits `roles/privateca.admin`, which is a role required for using the [Certificate Authority Service](https://cloud.google.com/certificate-authority-service). The automation leverages IstioCA instead, so this role is not required.

Set the JSON file key path with `asm_gcpsapath` within the target cluster inventory.

For more information about the permissions required for installing Anthos Service Mesh, please refer to the [documentation](https://cloud.google.com/service-mesh/docs/installation-permissions)

#### Anthos Config Management GSA (Optional)

The Anthos Config Management service account is used when installing ACM onto user clusters. It requires the below permissions in the target Google Cloud project:

* `gkehub.editor`
* `serviceusage.serviceUsageViewer`
* `source.reader`, if using Google Cloud Source repositories

Set the JSON file key path with `acm_gcpsapath` within the target cluster inventory. 

## Playbook Execution

This section outlines the execution of the two playbooks for the
installation and uninstallation of the admin workstation.
You must use verbosity level one (`-v`) or higher to see output from `gkeadm` and `gkectl`.

### Export shell environment variables that are used by Ansible

Should you not want to store sensitive values directly in the inventory (or any Ansible YAML file for that matter), playbooks can leverage environment variables. Below are the environment variables currently referenced in the automation. Some of them are required, but others are optional depending on your deployment set up. 

```shell
export VMWARE_HOST="<FQDN>"
export VMWARE_USER="username@vcenter.domain"
export VMWARE_PASSWORD="secret"

# optional - private docker registry
export PRIV_REG_ADDRESS=""
export PRIV_REG_CA_FILE="ca.crt" #  If using Ansible Tower, certificate file found on Tower
export PRIV_REG_USERNAME="username"
export PRIV_REG_PASSWORD="AnotherSecret"
export PRIV_REG_EMAIL="username@email.com"

# optional - private generic artifact registry
export ARTIFACT_HOST="host"
export ARTIFACT_USERNAME="username"
export ARTIFACT_ENC_PASS="encryptedSecret"
export ARTIFACT_API_TOKEN="token"

# optional - AIS configuration
export AIS_OIDC_CLIENTID="clientID"
export AIS_OIDC_CLIENTSECRET="clientSecret"
export AIS_GCS_BUCKET="uploadBucket" # If uploading the AIS login config file to a Cloud Storage bucket 
```

These values can then be passed in during playbook runs (as seen below), or just referenced in the inventory as they are now. 

```shell
VMWARE_USER="user@vcenter.local" \
VMWARE_PASSWORD="secret" \
ansible-playbook ... -v -e adminws_vc_username='$VMWARE_USER' \
  -e adminws_vc_password='$VMWARE_PASSWORD'
```

For some of these sensitive values, consider using a external service for injecting them.

### Admin Workstation - Playbook for Installation

Run the playbook.

Estimates for the admin workstation:

- takes around 37 minutes when OVAs need to be downloaded from Google and uploaded to vCenter
- takes around 3 minutes 30 seconds to complete if OVAs are already uploaded

```shell
# at least one -v is required to see output from gkeadm/gkectl
ansible-playbook -i inventory/site-a/admin-ws01.yml playbooks/adminws_install.yml -v

# example summary
Thursday 26 August 2021  16:58:40 -0400 (0:00:00.029)       0:03:32.864 *******
===============================================================================
adminws : [adminws] Create admin workstation -- 199.44s
adminws : [adminws] Templating YAML files -- 6.89s
adminws : [adminws] Templating YAML files -- 6.24s
adminws : [adminws] Sanity Checks -- 0.10s
adminws : include_tasks -- 0.03s
adminws : include_tasks -- 0.03s
adminws : include_tasks -- 0.02s
adminws : [adminws] Note on optional values -- 0.02s
Playbook run took 0 days, 0 hours, 3 minutes, 32 seconds
```

### Admin Workstation - Playbook for Uninstallation

Run the playbook (takes around 5 seconds to complete).

```shell
# at least one -v is required to see output from gkeadm/gkectl
ansible-playbook -i inventory/site-a/admin-ws01.yml playbooks/adminws_uninstall.yml -v 

# example summary
Thursday 26 August 2021  19:41:31 -0400 (0:00:04.887)       0:00:05.051 *******
===============================================================================
adminws : [adminws] Delete admin workstation -- 4.89s
adminws : [adminws] Include tasks - uninstall -- 0.04s
adminws : [adminws] Include tasks - asserts -- 0.03s
adminws : [adminws] Include tasks - install -- 0.03s
Playbook run took 0 days, 0 hours, 0 minutes, 5 seconds
```

### Admin Workstation - Playbook for Upgrade

Run the playbook (takes about 20 minutes to complete) after updating the `glb_anthos_version` variable defined in `inventory/site-a/all.yml` to the newer version.

``` shell
# at least one -v is required to see output from gkeadmin/gkectl 
ansible-playbook -i inventory/site-a/admin-ws01.yml playbooks/adminws_upgrade.yml -v

# example 
# TODO: add summary output 
```

### Admin Cluster - Playbook for Installation

Run the playbook (takes about 39 minutes to complete, potentially more with the use of a private registry).

```shell
ansible-playbook -i inventory/site-a/admincluster-01.yml playbooks/admincluster_install.yml -v

# example summary
Monday 04 October 2021  11:36:04 -0400 (0:28:03.499)       0:29:50.318 ********
===============================================================================
admincluster : [ac] Create cluster  1683.50s
admincluster : [ac] Templating YAML files -- 42.81s
admincluster : [ac] Templating YAML files -- 35.62s
admincluster : [ac] Templating YAML files -- 28.12s
admincluster : [ac] Sanity Checks - 0.10s
admincluster : [ac] Include tasks - install.yml -- 0.04s
admincluster : [ac] Include tasks - asserts.yml -- 0.04s
admincluster : [ac] Note on optional values -- 0.03s
Playbook run took 0 days, 0 hours, 29 minutes, 50 seconds
```

Optionally, you can skip some Ansible tasks, if desired.

```shell
# --skip-tags is optional to skip specific Ansible tasks
ansible-playbook -i inventory/site-a/admincluster-01.yml playbooks/admincluster_install.yml \
  -v --skip-tags preflight,prepare
```

### Admin Cluster - Playbook for Uninstallation

Run the playbook (takes less than 10 minutes to complete).

```shell
ansible-playbook -i inventory/site-a/admincluster-01.yml playbooks/admincluster_uninstall.yml -v

# example summary
Monday 02 May 2022  01:52:21 +0000 (0:00:00.016)       0:00:28.701 ************
===============================================================================
admincluster : [ac] Delete Monitoring System Pods -- 14.05s
admincluster : [ac] Unregister Admin Cluster from GKE Hub -- 4.96s
copy_gcp_sa : [copy-gcp-sa] Copy GCP SA JSON key files to admin workstation -- 2.94s
admincluster : [ac] Get Register Service Account Name -- 1.14s
admincluster : [ac] Delete VMs from vCenter -- 1.13s
admincluster : [ac] Switch to Register Service Account Name -- 1.04s
admincluster : [ac] Clean up old SSH host keys from incomplete runs -- 0.60s
admincluster : [ac] Get vSphere VM Names -- 0.47s
cleanup : Find GCP SA JSON files on admin workstation -- 0.46s
admincluster : [ac] Delete Logging System Pods -- 0.43s
cleanup : Clean up GCP SA JSON files on admin workstation -- 0.37s
cleanup : Find *credential*.yaml files on admin workstation -- 0.33s
copy_gcp_sa : [copy-gcp-sa] Create folder for GCP SA JSON key files on admin workstation -- 0.30s
cleanup : Clean up GCP SA JSON files on admin workstation -- 0.04s
cleanup : Clean up GCP SA JSON files on jumphost -- 0.04s
Include role for GCP SA JSON key files -- 0.04s
copy_gcp_sa : [copy-gcp-sa] Copy GCP SA JSON key files to jump host -- 0.03s
admincluster : [ac] Include tasks - uninstall.yml -- 0.03s
Include role to delete sensitive files -- 0.03s
cleanup : Clean up *credential*.yaml on admin workstation -- 0.03s
Playbook run took 0 days, 0 hours, 0 minutes, 28 seconds
```

### Admin Cluster - Playbook for Upgrade

Run the playbook (takes about 30 minutes to complete) after updating the `glb_anthos_version` variable defined in `inventory/site-a/all.yml` to the newer version.

> **Note:** Upgrades to the admin workstation and user clusters are a prerequisite to upgrading the admin cluster. Upgrades to the admin cluster will otherwise result in a failure. 

```shell
ansible-playbook -i inventory/site-a/admincluster-01.yml playbooks/admincluster_upgrade.yml -v

# example summary
Friday 23 September 2022  17:10:56 -0400 (0:00:00.046)                                             1:37:25.400 ****** 
===============================================================================
admincluster : [ac] Upgrade admin cluster -- 5712.50s
admincluster : [ac] Upload new bundle to vSphere -- 120.96s
[copy_gcp_sa] Copy GCP SA JSON key files to admin workstation -- 2.96s
admincluster : [ac] Detect if Admin Cluster already exists -- 1.21s
admincluster : [ac] Clean up old SSH host keys from incomplete runs -- 1.20s
[copy_gcp_sa] Templating YAML files - vCenter credentials -- 1.00s
admincluster : [ac] Get target version from Admin Workstation -- 0.87s
[cleanup] Find GCP SA JSON files on admin workstation -- 0.49s
admincluster : [ac] Replace bundlePath in admin cluster config YAML -- 0.45s
[cleanup] Clean up GCP SA JSON file folder on admin workstation -- 0.40s
admincluster : [ac] Detect if Admin Cluster kubeconfig exists -- 0.38s
[cleanup] Clean up *credential*.yaml on admin workstation in admin cluster subfolder -- 0.30s
[cleanup] Find *credential*.yaml files on admin workstation in admin cluster subfolder -- 0.25s
[cleanup] Find *credential*.yaml files on admin workstation -- 0.24s
[copy_gcp_sa] Ensure credentials directory exists -- 0.23s
[copy_gcp_sa] Create folder for GCP SA JSON key files on admin workstation -- 0.22s
[cleanup] Clean up sensitive file on admin workstation -- 0.20s
admincluster : [ac] Sanity Checks -- 0.12s
admincluster : [ac] Include tasks - upgrade.yml -- 0.11s
[cleanup] Clean up sensitive files on jumphost -- 0.10s

Playbook run took 0 days, 1 hours, 37 minutes, 25 seconds
```

### User Cluster - Playbook for Installation

Run the playbook (takes about 40 minutes to complete).

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/usercluster_install.yml -v

# example summary
Monday 04 October 2021  17:17:38 -0400 (0:00:00.045)       0:28:36.093 ********
===============================================================================
usercluster : [uc] Create user cluster -- 1485.98s
usercluster : [uc] Create load balancer -- 138.95s
usercluster : [uc] Templating YAML files -- 34.05s
usercluster : [uc] Templating YAML files -- 28.60s
usercluster : [uc] Templating YAML files -- 28.22s
usercluster : [uc] Sanity Checks -- 0.07s
usercluster : [uc] Include tasks - uninstall.yml - 0.05s
usercluster : [uc] Include tasks - install.yml  0.04s
usercluster : [uc] Include tasks - asserts.yml  0.04s
usercluster : [uc] Note on optional values - 0.03s
Playbook run took 0 days, 0 hours, 28 minutes, 36 seconds
```

### User Cluster - Playbook for Uninstallation

Run the playbook (takes less than 10 minutes to complete).

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/usercluster_uninstall.yml -v

# example summary
Monday 02 May 2022  00:19:09 +0000 (0:00:00.039)       0:08:03.354 ************
===============================================================================
usercluster : [uc] Delete user cluster -- 482.59s
usercluster : [uc] Clean up old SSH host keys from incomplete runs -- 0.62s
usercluster : [uc] Delete load balancer -- 0.04s
usercluster : [uc] Include tasks - uninstall.yml -- 0.03s
usercluster : [uc] Include tasks - install.yml -- 0.02s
usercluster : [uc] Include tasks - asserts.yml -- 0.02s
Playbook run took 0 days, 0 hours, 8 minutes, 3 seconds
```

### User Cluster - Playbook for Upgrade

Run the playbook (takes about 30 minutes to complete) after updating the `glb_anthos_version` variable defined in `inventory/site-a/all.yml` to the newer version.

> **Note:** Upgrade to the admin workstation is a prerequisite to upgrading user clusters. Upgrades to the user clusters will otherwise result in a failure. 

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/usercluster_upgrade.yml -v

# example summary
Friday 23 September 2022  15:24:56 -0400 (0:00:00.186)                                              1:14:18.537 ****** 
===============================================================================
usercluster : [uc] Upgrade user cluster -- 3937.77s
usercluster : [uc] Upload new bundle to vSphere -- 505.80s
[copy_gcp_sa] Copy GCP SA JSON key files to admin workstation -- 2.89s
usercluster : [uc] Check if existing api server is up -- 1.42s
[cleanup] Find GCP SA JSON files on admin workstation -- 1.13s
usercluster : [uc] Templating YAML files -- 0.86s
[copy_gcp_sa] Templating YAML files - vCenter credentials -- 0.76s
usercluster : [uc] Get current user cluster version -- 0.74s
usercluster : [uc] Get gkectl version from Admin workstation -- 0.72s
usercluster : [uc] Clean up old SSH host keys from incomplete runs -- 0.68s
usercluster : [uc] Templating YAML files - IP block -- 0.50s
[cleanup] Clean up sensitive file on admin workstation -- 0.48s
usercluster : [uc] Check for existing cluster kubeconfig -- 0.36s
usercluster : [uc] Replace cluster version in its config YAML file -- 0.35s
[cleanup] Clean up GCP SA JSON file folder on admin workstation -- 0.31s
[cleanup] Find *credential*.yaml files on admin workstation -- 0.27s
[cleanup] Find *credential*.yaml files on admin workstation in user cluster subfolder -- 0.25s
[cleanup] Clean up *credential*.yaml on admin workstation in user cluster subfolder -- 0.25s
[cleanup] Clean up sensitive files on jumphost -- 0.24s
[copy_gcp_sa] Ensure credentials directory exists -- 0.22s

Playbook run took 0 days, 1 hours, 14 minutes, 18 seconds
```

### ASM - Playbook for Installation

Anthos Service Mesh (ASM) can be optionally installed onto a user cluster. 

Run the playbook (takes about 5 minutes to complete).

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/asm_install.yml -v

# example summary
Monday 26 September 2022  14:16:22 -0400 (0:00:00.021)       0:02:44.482 ****** 
===============================================================================
[asm] Install Anthos Service Mesh -- 116.65s
[asm] Create ASM offline package bundle -- 17.65s
[asm] Add mesh_id label to cluster -- 4.27s
[copy_gcp_sa] Copy GCP SA JSON key files to admin workstation -- 3.96s
[asm] Create asm-ingress namespace -- 2.92s
[asm] Switch to Connect Register Service Account Name -- 1.49s
asm : [ASM] Detect if ASM already installed and get current revision -- 1.46s
[asm] Switch to asm Service Account Name -- 1.16s
[asm] Create RBAC rule required for asm -- 1.09s
[asm] Copy istioctl to istioctl_dest_path -- 1.03s
[asm] Download asmcli -- 1.01s
[copy_gcp_sa] Templating YAML files - vCenter credentials -- 0.79s
[asm] Create the Ingress Gateway -- 0.77s
[asm] Create istio-system namespace -- 0.73s
[asm] Template asm-ingress namespace -- 0.71s
[asm] Delete RBAC rule created above -- 0.68s
[asm] Template the ASM control plane config file -- 0.65s
[cleanup] Find GCP SA JSON files on admin workstation -- 0.64s
[asm] Template RBAC rule required for asm -- 0.60s
[asm] Template Istio system namespace -- 0.60s
Playbook run took 0 days, 0 hours, 2 minutes, 44 seconds
```

### ASM - Playbook for Uninstallation

Run the playbook (takes less than 5 minutes to complete). 

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/asm_uninstall.yml -v

# example summary
Monday 26 September 2022  14:53:22 -0400 (0:00:00.023)       0:00:42.965 ****** ------- 0.38s
=============================================================================== 
[asm] Create ASM offline package bundle -- 18.25s
[asm] Delete istio-system, asm-ingress and asm-system namespaces -- 13.94s
[asm] Remove the in-cluster control plane -- 4.89s
asm : [ASM] Detect if ASM already installed and get current revision -- 1.44s
[asm] Download asmcli -- 0.89s
[asm] Delete webhooks -- 0.48s
[cleanup] Find GCP SA JSON files on admin workstation -- 0.47s
[cleanup] Delete temp directory created for ASM files -- 0.47s
[asm] Create folder for ASM files on admin workstation -- 0.29s
[cleanup] Find *credential*.yaml files on admin workstation in user cluster subfolder -- 0.21s
[asm] Make asmcli executable -- 0.21s
[cleanup] Find *credential*.yaml files on admin workstation -- 0.19s
[cleanup] Clean up sensitive file on admin workstation -- 0.13s
asm : [mesh] Setup cluster config -- 0.08s
asm : [ASM] Sanity Checks -- 0.07s
[asm] Copy istioctl to istioctl_dest_path -- 0.05s
asm : [ASM] Include tasks - uninstall.yml -- 0.04s
asm : [mesh] Sanity Checks -- 0.04s
Include role to delete sensitive files -- 0.04s
[cleanup] Include tasks - asserts.yml -- 0.03s
Playbook run took 0 days, 0 hours, 0 minutes, 42 seconds
```

### ASM - Playbook for Upgrade

Run the playbook (takes about 5 minutes to complete) after updating the `asm_version`, `asm_revision` and `asm_asmcli_version` variables in the target user cluster's inventory file.

For more information, refer to the `role/asm/README.md`.

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/asm_upgrade.yml -v

# example summary
Monday 26 September 2022  14:37:58 -0400 (0:00:16.585)       0:02:43.074 ****** 
=============================================================================== 
[asm] Install Anthos Service Mesh -- 101.68s
[asm] Create ASM offline package bundle -- 17.00s
[asm] Wait until the ingress gateway pods are ready -- 16.59s
[copy_gcp_sa] Copy GCP SA JSON key files to admin workstation -- 4.18s
[asm] Add mesh_id label to cluster -- 3.23s
[asm] Restart the Pods to trigger re-injection -- 2.57s
[asm] Create asm-ingress namespace -- 2.27s
asm : [ASM] Detect if ASM already installed and get current revision -- 1.77s
[asm] Switch to Connect Register Service Account Name -- 1.34s
[asm] Switch to asm Service Account Name -- 1.08s
[asm] Copy istioctl to istioctl_dest_path -- 1.08s
[asm] Download asmcli -- 0.94s
[copy_gcp_sa] Templating YAML files - vCenter credentials -- 0.80s
[asm] Create RBAC rule required for asm -- 0.74s
[asm] Template asm-ingress namespace -- 0.73s
[asm] Template the ASM control plane config file -- 0.68s
[asm] Template RBAC rule required for asm -- 0.63s
[cleanup] Find GCP SA JSON files on admin workstation -- 0.47s
[copy_gcp_sa] Ensure credentials directory exists -- 0.42s
[asm] Delete old RBAC rule required for asm (if exists) -- 0.38s
Playbook run took 0 days, 0 hours, 2 minutes, 43 seconds
```

### ACM - Playbook for Installation

Anthos Config Management (ACM) can be optionally installed onto a user cluster.

Run the playbook (takes about 15 minutes to complete).

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/acm_install.yml -v

# example summary
Monday 19 September 2022  08:58:27 -0400 (0:00:00.020)       0:07:09.933 ****** 
=============================================================================== 
[acm] Wait for Config Sync to be synced -- 411.97s
[copy_gcp_sa] Copy GCP SA JSON key files to admin workstation -- 3.22s
[acm] Enable ACM -- 3.19s
[acm] Switch to ACM Service Account Name -- 1.38s
[acm] Get current ACM version -- 1.32s
[acm] Create ACM namespace -- 1.25s
[acm] Check ACM Feature Status -- 1.18s
[acm] Clean up old SSH host keys from incomplete runs -- 0.99s
[acm] Copy ACM namespace file -- 0.86s
[copy_gcp_sa] Templating YAML files - vCenter credentials -- 0.78s
[acm] Templating YAML files -- 0.56s
[cleanup] Find GCP SA JSON files on admin workstation -- 0.50s
[copy_gcp_sa] Create folder for GCP SA JSON key files on admin workstation -- 0.26s
[acm] Clean up root repository ssh key file -- 0.24s
[cleanup] Clean up *credential*.yaml on admin workstation in user cluster subfolder -- 0.21s
[cleanup] Clean up GCP SA JSON file folder on admin workstation -- 0.21s
[cleanup] Find *credential*.yaml files on admin workstation in user cluster subfolder -- 0.20s
[copy_gcp_sa] Ensure credentials directory exists -- 0.20s
[cleanup] Find *credential*.yaml files on admin workstation -- 0.19s
[cleanup] Clean up sensitive file on admin workstation -- 0.13s
Playbook run took 0 days, 0 hours, 5 minutes, 9 seconds
```

### ACM - Playbook for Uninstallation

Run the playbook (takes less than 10 minutes to complete).

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/acm_uninstall.yml -v

# example summary
Tuesday 20 September 2022  09:40:18 -0400 (0:00:00.020)       0:01:05.454 ****** 
===============================================================================
[acm] Remove ACM namespaces -- 47.00s
[acm] Remove ACM operator configuration -- 3.79s
[copy_gcp_sa] Copy GCP SA JSON key files to admin workstation -- 3.52s
[acm] Disable ACM -- 3.21s
[acm] Switch to ACM Service Account Name -- 1.52s
[acm] Get current ACM version -- 1.40s
[copy_gcp_sa] Templating YAML files - vCenter credentials -- 0.81s
[acm] Remove ACM operator CRD -- 0.62s
[acm] Clean up old SSH host keys from incomplete runs -- 0.51s
[cleanup] Find GCP SA JSON files on admin workstation -- 0.48s
```

### ACM - Playbook for Upgrade

Run the playbook (takes about 15 minutes to complete) after updating the `acm_version` variable defined in the relevant user cluster's inventory file.

```shell
ansible-playbook -i inventory/site-a/usercluster-01.yml playbooks/acm_upgrade.yml -v

# example summary
Wednesday 21 September 2022  10:30:18 -0400 (0:00:00.028)       0:02:12.284 ****** 
=============================================================================== 
[acm] Wait for Config Sync -- 88.43s
[acm] Wait for upgrade to start -- 30.07s
[acm] Upgrade ACM -- 3.36s
[acm] Switch to ACM Service Account Name -- 1.49s
[acm] Get current ACM version -- 1.40s
[acm] Clean up old SSH host keys from incomplete runs -- 0.48s
[acm] Wait for Policy Controller -- 0.05s
[acm] Sanity Checks -- 0.04s
[acm] Compare current version with requested -- 0.04s
Playbook run took 0 days, 0 hours, 2 minutes, 12 seconds
```

## Ansible Tower Prerequisites

Custom Credential Types to store GSA JSON key file content.

### GSA - Component Access

```yaml
# INPUT CONFIGURATION
fields:
  - id: json
    type: string
    label: JSONcontent
    secret: true
required:
  - json

# INJECTOR CONFIGURATION
env:
  GCPSA_COMACC_FILE: '{{ tower.filename }}'
file:
  template: '{{ json }}'
```

### GSA - Logging-Monitoring

```yaml
# INPUT CONFIGURATION
fields:
  - id: json
    type: string
    label: JSONcontent
    secret: true
required:
  - json

# INJECTOR CONFIGURATION
env:
  GCPSA_LOGMON_FILE: '{{ tower.filename }}'
file:
  template: '{{ json }}'
```

### GSA - Connect Register

```yaml
# INPUT CONFIGURATION
fields:
  - id: json
    type: string
    label: JSONcontent
    secret: true
required:
  - json

# INJECTOR CONFIGURATION
env:
  GCPSA_CONREG_FILE: '{{ tower.filename }}'
file:
  template: '{{ json }}'
```

### GSA - Audit Logging

```yaml
# INPUT CONFIGURATION
fields:
  - id: json
    type: string
    label: JSONcontent
    secret: true
required:
  - json

# INJECTOR CONFIGURATION
env:
  GCPSA_AUDLOG_FILE: '{{ tower.filename }}'
file:
  template: '{{ json }}'
```

### Optional - privateRegistry for container images

```yaml
# INPUT CONFIGURATION
fields:
  - id: address
    type: string
    label: Address
  - id: username
    type: string
    label: Username
  - id: password
    type: string
    label: Password
    secret: true
  - id: email
    type: string
    label: Email
  - id: ca_cert
    type: string
    label: CA_CERT
    multiline: true
required:
  - username
  - password

# INJECTOR CONFIGURATION
env:
  PRIV_REG_CA_FILE: '{{ tower.filename.ca_file }}'
  PRIV_REG_PASSWORD: '{{ password }}'
  PRIV_REG_USERNAME: '{{ username }}'
  PRIV_REG_ADDRESS: '{{ address }}'
  PRIV_REG_EMAIL: '{{ email }}'
file:
  template.ca_file: '{{ ca_cert }}'
```

### Optional - private repository for artifacts

```yaml
# INPUT CONFIGURATION
fields:
- id: repo_base_url
  type: string
  label: "The base url for the repository, for artifactory this would be https://{hostname}/artifactory/{repository}"
- id: repo_username
  type: string
  label: "The username for authenticating to the repository"
- id: repo_enc_password
  type: string
  label: "The encrypted password for the user. Authentication can use this or the API token"
  secret: true
- id: repo_api_token
  type: string
  label: "The API token for the user. Authentication can use this or the encrypted password"
  secret: true
required:
  - repo_url
  - repo_username

# INJECTOR CONFIGURATION
env:
  ARTIFACT_BASE_URL: '{{ repo_base_url }}'
  ARTIFACT_USERNAME: '{{ repo_username }}'
  ARTIFACT_ENC_PASS: '{{ repo_enc_password }}'
  ARTIFACT_API_TOKEN: '{{ repo_api_token }}'
```

### Optional - GCP SA - Enabling GKE Hub feature Anthos Config Management (ACM)

```yaml
# INPUT CONFIGURATION
fields:
  - id: gcpsaacm
    type: string
    label: JSONcontent
    secret: true
required:
  - gcpsaacm

# INJECTOR CONFIGURATION
env:
  GCPSA_ACM_FILE: '{{ tower.filename }}'
file:
  template: '{{ gcpsaacm }}'
```

### Optional - GCP SA - Installing Anthos Service Mesh (ASM)

```yaml
# INPUT CONFIGURATION
fields:
  - id: gcpsaasm
    type: string
    label: JSONcontent
    secret: true
required:
  - gcpsaasm

# INJECTOR CONFIGURATION
env:
  GCPSA_ASMCFG_FILE: '{{ tower.filename }}'
file:
  template: '{{ gcpsaasm }}'
```

### Optional - SSH Private Key - Authenticate ACM Config Sync to the main root repository

```yaml
# INPUT CONFIGURATION
fields:
  - id: acmsshprivkey
    type: string
    label: "SSH Private Key for ACM root repository"
    secret: true
    format: ssh_private_key
    multiline: true
required:
  - acmsshprivkey

# INJECTOR CONFIGURATION
env:
  GIT_ACMSSH_FILE: '{{ tower.filename }}'
file:
  template: '{{ acmsshprivkey }}'
```

### Optional - AIS OIDC Identity Provider Secrets - Configure OIDC connection to Identity provider from AIS

```yaml
# INPUT CONFIGURATION
fields:
  - id: clientid
    type: string
    label: "Client ID for OIDC"
  - id: clientsecret
    type: string
    label: "Client Secret for OIDC"
    secret: true
  - id: cadata
    type: string
    label: "Certificate chain to verify SSL for Identity Privider. Certificates must be in PEM format, concatenated and base64 encoded"
required:
  - clientid
  - clientsecret

# INJECTOR CONFIGURATION
env:
  AIS_OIDC_CLIENTID: '{{ clientid }}'
  AIS_OIDC_CLIENTSECRET: '{{ clientsecret }}'
  AIS_OIDC_CADATA: '{{ cadata }}'
```

### Optional - AIS LDAP Identity Provider Secrets - Configure LDAP connection to Identity provider from AIS

```yaml
# INPUT CONFIGURATION
fields:
  - id: username
    type: string
    label: "username for service account to connect to LDAP"
  - id: password
    type: string
    label: "password for service account to connect to LDAP"
    secret: true
  - id: cadata
    type: string
    label: "Certificate chain to verify SSL for Identity Privider. Certificates must be in PEM format, concatenated and base64 encoded"
required:
  - username
  - password

# INJECTOR CONFIGURATION
env:
  AIS_LDAP_USERNAME: '{{ username }}'
  AIS_LDAP_PASSWORD: '{{ password }}'
  AIS_LDAP_CADATA: '{{ cadata }}'
```

### Optional - AIS OIDC login config file - Copy to Google Cloud Storage bucket

This Google Service Account is optional.
It can be used to copy the Anthos user cluster login config to a Google Cloud Storage bucket.

```yaml
# INPUT CONFIGURATION
fields:
  - id: gcpsaais
    type: string
    label: JSONcontent
    secret: true
  - id: gcsbucket
    type: string
    label: "GCS bucket name"
required:
  - gcpsaais
  - gcsbucket

# INJECTOR CONFIGURATION
env:
  GCPSA_AIS_FILE: '{{ tower.filename }}'
  AIS_GCS_BUCKET: '{{ gcsbucket }}'
file:
  template: '{{ gcpsaais }}'
```

### Optional - ASM Ingress Secrets - Configure TLS secret for ingress gateway

```yaml
# INPUT CONFIGURATION
fields:
  - id: tls_cert
    type: string
    label: The certificate for TLS
    multiline: true
  - id: tls_key
    type: string
    label: The TLS key
    multiline: true
    secret: true
required:
  - tls_cert
  - tls_key

# INJECTOR CONFIGURATION
env:
  ASM_TLSCERT_FILE: '{{ tower.filename.tls_cert }}'
  ASM_TLSKEY_FILE: '{{ tower.filename.tls_key }}'
file:
  template.tls_cert: '{{ tls_cert }}'
  template.tls_key: '{{ tls_key }}'
```

### Optional - ASM mTLS Secrets - Configure certs for mTLS connection between the workloads

```yaml
# INPUT CONFIGURATION
fields:
  - id: root_cert
    type: string
    label: The root certificate for mTLS
    multiline: true
  - id: cert_chain
    type: string
    label: The cert chain for mTLS
    multiline: true
  - id: ca_cert
    type: string
    label: The CA certificate for mTLS
    multiline: true
  - id: ca_key
    type: string
    label: The CA key for mTLS
    multiline: true
    secret: true
required:
  - root_cert
  - cert_chain
  - ca_cert
  - ca_key

# INJECTOR CONFIGURATION
env:
  ASM_ROOTCERT_FILE: '{{ tower.filename.root_cert }}'
  ASM_CERTCHAIN_FILE: '{{ tower.filename.cert_chain }}'
  ASM_CACERT_FILE: '{{ tower.filename.ca_cert }}'
  ASM_CAKEY_FILE: '{{ tower.filename.ca_key }}'
file:
  template.root_cert: '{{ root_cert }}'
  template.cert_chain: '{{ cert_chain }}'
  template.ca_cert: '{{ ca_cert }}'
  template.ca_key: '{{ ca_key }}'
```

## **License**

Copyright 2022 Google LLC. This software is provided as-is, without warranty or representation for any use or purpose.
Your use of it is subject to your agreement with Google.