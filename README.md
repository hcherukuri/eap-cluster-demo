# eap-cluster-demo

A demo to set up a JBoss EAP 8.0 cluster (with an optional upgrade to 8.1) using the Ansible `redhat.eap` collection.

## Overview

This demo provisions a multi-node JBoss EAP 8.0 cluster using TCP PING for JGroups discovery. [`upgrade.yml`](upgrade.yml) can migrate the cluster to EAP 8.1 and apply the latest 8.1.x update. It uses the following `redhat.eap` collection roles:

- **`eap_install`** – installs EAP 8.0.0 via JBoss Installation Manager (Prospero)
- **`eap_systemd`** – configures the systemd service and applies YAML configuration
- **`eap_firewalld`** – opens the required firewall ports (enabled via `eap_firewalld_enabled: true`)
- **`eap_validation`** – verifies the server is running (ports, systemd, management CLI)
- **`eap_app_deploy`** – deploys the demo application via JBoss CLI

Shared variables are in [`vars.yml`](vars.yml). See [Repository layout](#repository-layout) for all files in this demo.

## Repository layout

| Path | Purpose |
|---|---|
| [`playbook.yml`](playbook.yml) | Install, validate, and deploy the cluster |
| [`upgrade.yml`](upgrade.yml) | Upgrade 8.0.0 to 8.1.0, apply latest 8.1.x update, then validate |
| [`validate.yml`](validate.yml) | Validate only (imported by `playbook.yml` after install) |
| [`vars.yml`](vars.yml) | Shared playbook variables |
| [`upgrade-vars.yml`](upgrade-vars.yml) | Upgrade-only overrides (`8.0.0` → `8.1.0`) |
| [`inventory`](inventory) | Cluster inventory template |
| [`tasks/cluster_nodes.yml`](tasks/cluster_nodes.yml) | Shared JGroups TCP PING cluster node list |
| [`tasks/eap_version.yml`](tasks/eap_version.yml) | Shared EAP version reporting from `server.log` |
| [`templates/eap_ymlconfig.yml.j2`](templates/eap_ymlconfig.yml.j2) | JGroups TCP PING YAML configuration |
| [`ansible.cfg.example`](ansible.cfg.example) | Example local Ansible config for vendored collections |
| [`check_collections.yml`](check_collections.yml) | Optional helper to list installed collections |
| [`collections/`](collections/) | Vendored `redhat.eap` collection for AAP execution environments |

## Requirements

- **ansible-core** >= 2.16.0
- **`redhat.eap`** collection >= 1.5.11 from [Red Hat Ansible Automation Hub](https://console.redhat.com/ansible/automation-hub)
- A valid **Red Hat subscription** to download EAP online, or set `eap_offline_install: true` in [`vars.yml`](vars.yml) for offline installs
- For **online installs** (`eap_offline_install: false`, the default), pass Red Hat Customer Portal credentials to the playbook as `rhn_username` and `rhn_password` (see [Online install credentials](#online-install-credentials))

### Collection install

This repo ships a vendored copy under `collections/` for Ansible Automation Platform job templates and local runs. Point Ansible at it with:

```bash
cp ansible.cfg.example ansible.cfg
```

Optional check:

```bash
ansible-playbook check_collections.yml
```

## Inventory

Edit [`inventory`](inventory) and fill in `ansible_host` and `ansible_ssh_private_key_file` for each node:

```ini
[eap]
eap-cluster-demo-1 ansible_host="<IP1>" ansible_user=root ansible_ssh_private_key_file="<key>"
eap-cluster-demo-2 ansible_host="<IP2>" ansible_user=root ansible_ssh_private_key_file="<key>"
eap-cluster-demo-3 ansible_host="<IP3>" ansible_user=root ansible_ssh_private_key_file="<key>"
```

When running from Ansible Automation Platform, the job template inventory may include `localhost` as the execution node. The playbooks exclude it from EAP install and upgrade work using `hosts: all:!localhost`. [`playbook.yml`](playbook.yml) also includes a localhost pre-play that disables privilege escalation on the execution node to avoid AAP sudo issues.

## Variables

Key variables in [`vars.yml`](vars.yml):

| Variable | Value | Description |
|---|---|---|
| `eap_version` | `8.0.0` | EAP baseline version to install |
| `eap_config_base` | `standalone-full-ha.xml` | Base server configuration (full profile + HA) |
| `eap_offline_install` | `false` | Online install from Red Hat (set to `true` for offline ZIP installs) |
| `rhn_username` | *(required for online install)* | Red Hat service account client ID |
| `rhn_password` | *(required for online install)* | Red Hat service account client secret |
| `eap_enable_yml_config` | `true` | Enable YAML configuration extension (required for TCPPING) |
| `eap_firewalld_enabled` | `true` | Configure firewalld rules automatically via `eap_firewalld` role |
| `app.name` | `helloworld.war` | Demo application deployed by `playbook.yml` |
| `app.url` | Google Drive URL | Download location for the demo WAR (fetched on the controller) |

## Online install credentials

When `eap_offline_install` is `false` (the default in [`vars.yml`](vars.yml)), the playbooks download EAP from the Red Hat Customer Portal. You must provide a [service account](https://console.redhat.com/application-services/service-accounts) client ID and secret via `rhn_username` and `rhn_password`.

Do not store these values in [`vars.yml`](vars.yml). Pass them as secrets at runtime:

```bash
# Local run (use Ansible Vault, a credentials file, or prompt instead of plain text)
ansible-playbook -i inventory playbook.yml \
  -e rhn_username='<client_id>' \
  -e rhn_password='<client_secret>'
```

On **Ansible Automation Platform**, add `rhn_username` and `rhn_password` as credentials or survey secrets on the job template so they are injected as extra variables at launch. The upgrade playbook also needs these credentials when downloading the migration tool and 8.1.x updates online.

## Usage

```bash
# Install, validate, and deploy (online install — pass RHN credentials as above)
ansible-playbook -i inventory playbook.yml \
  -e rhn_username='<client_id>' \
  -e rhn_password='<client_secret>'

# Validate only
ansible-playbook -i inventory validate.yml

# Upgrade 8.0.0 to 8.1.0, apply latest 8.1.x update, and validate
ansible-playbook -i inventory upgrade.yml \
  -e rhn_username='<client_id>' \
  -e rhn_password='<client_secret>'
```

### Version reporting

Both install and upgrade playbooks print the runtime EAP version from the server log after the service is running:

```text
TASK [Display EAP version] *****************************************************
ok: [eap-cluster-demo-1] => {
    "msg": "eap-cluster-demo-1 running JBoss EAP 8.0 Update 9 (...)"
}
```

After upgrade:

```text
ok: [eap-cluster-demo-1] => {
    "msg": "eap-cluster-demo-1 upgraded to JBoss EAP 8.1 Update 7.5 (...)"
}
```

### Upgrade behavior

The install playbook provisions EAP `8.0.0` under `/opt/jboss_eap/jboss-eap-8.0/`. The upgrade playbook loads [`upgrade-vars.yml`](upgrade-vars.yml) after [`vars.yml`](vars.yml) so `eap_version: "8.1.0"` wins over the install baseline.

Upgrading from 8.0 to 8.1 is a **major version migration**, not a Prospero in-place patch. The upgrade playbook:

1. Stops the running `eap` service
2. Installs EAP 8.1 to `/opt/jboss_eap/jboss-eap-8.1/` via Prospero (`eap_version: "8.1.0"` from [`upgrade-vars.yml`](upgrade-vars.yml))
3. Migrates configuration from the 8.0 home to the 8.1 home (`eap_migration` role)
4. Reconfigures systemd for the 8.1 installation (`eap_systemd` role)
5. Applies available 8.1.x updates (`eap_prospero_update: true`)

Do not set `eap_version` to a patch level such as `8.1.7`; that artifact is not published separately in Red Hat Customer Portal. If your AAP job template passes `eap_version` as an extra variable, remove it or set it to `8.1.0` so it does not override `upgrade-vars.yml`.

## Cluster configuration

The [`templates/eap_ymlconfig.yml.j2`](templates/eap_ymlconfig.yml.j2) template configures:

- A `JBOSS_ID` system property per node (set to `inventory_hostname`)
- Remote outbound socket bindings for each cluster member (dynamically looped over `eap_cluster_nodes`)
- JGroups `tcp` stack with `TCPPING` protocol replacing the default `MPING`
- JGroups `ee` channel pointed at the `tcp` stack

The cluster node list is built dynamically in [`tasks/cluster_nodes.yml`](tasks/cluster_nodes.yml) from the playbook inventory batch.

## Author

**Harsha Cherukuri** — Red Hat  
[hcheruku@redhat.com](mailto:hcheruku@redhat.com)  
GitHub: [hcherukuri](https://github.com/hcherukuri)
