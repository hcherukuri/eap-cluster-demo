# eap-cluster-demo

A demo to set up a JBoss EAP 8.1 cluster using the Ansible `redhat.eap` collection.

## Overview

This demo provisions a 3-node JBoss EAP 8.1 cluster using TCP PING for JGroups discovery. It uses the following `redhat.eap` collection roles:

- **`eap_install`** – installs EAP 8.1.0 via JBoss Installation Manager (Prospero)
- **`eap_systemd`** – configures the systemd service and applies YAML configuration
- **`eap_firewalld`** – opens the required firewall ports (enabled via `eap_firewalld_enabled: true`)
- **`eap_validation`** – verifies the server is running (ports, systemd, management CLI)
- **`eap_app_deploy`** – deploys the demo application via JBoss CLI

Shared variables are in [`vars.yml`](vars.yml). See [Repository layout](#repository-layout) for all files in this demo.

## Repository layout

| Path | Purpose |
|---|---|
| [`playbook.yml`](playbook.yml) | Install, validate, and deploy the cluster |
| [`validate.yml`](validate.yml) | Validate only (reused after upgrade) |
| [`upgrade.yml`](upgrade.yml) | Upgrade 8.1.0 to 8.1.5, then validate |
| [`vars.yml`](vars.yml) | Shared playbook variables |
| [`inventory`](inventory) | 3-node cluster inventory |
| [`requirements.yml`](requirements.yml) | `redhat.eap` collection dependency |
| [`templates/eap_ymlconfig.yml.j2`](templates/eap_ymlconfig.yml.j2) | JGroups TCP PING YAML configuration |

## Requirements

- **ansible-core** >= 2.16.0
- **`redhat.eap`** collection >= 1.5.11 from [Red Hat Ansible Automation Hub](https://console.redhat.com/ansible/automation-hub)

Install the collection (requires Automation Hub access):

```bash
ansible-galaxy collection install -r requirements.yml
```

Also required:

- A valid **Red Hat subscription** to download EAP 8.1 (or pre-downloaded offline ZIP archives on the target hosts when `eap_offline_install: true`)
- Target hosts running **RHEL 8, 9, or 10** with **Java 17 or 21**

## Inventory

Edit [`inventory`](inventory) and fill in `ansible_host` and `ansible_ssh_private_key_file` for each node:

```ini
[eap]
eap-cluster-demo-1 ansible_host="<IP1>" ansible_user=root ansible_ssh_private_key_file="<key>"
eap-cluster-demo-2 ansible_host="<IP2>" ansible_user=root ansible_ssh_private_key_file="<key>"
eap-cluster-demo-3 ansible_host="<IP3>" ansible_user=root ansible_ssh_private_key_file="<key>"
```

## Variables

Key variables in [`vars.yml`](vars.yml):

| Variable | Value | Description |
|---|---|---|
| `eap_version` | `8.1.0` | EAP baseline version to install |
| `eap_config_base` | `standalone-full-ha.xml` | Base server configuration (full profile + HA) |
| `eap_offline_install` | `true` | Install from local archive (set to `false` for online install) |
| `eap_enable_yml_config` | `true` | Enable YAML configuration extension (required for TCPPING) |
| `eap_firewalld_enabled` | `true` | Configure firewalld rules automatically via `eap_firewalld` role |
| `app.name` | `helloworld.war` | Demo application deployed by `playbook.yml` |

## Usage

```bash
# Install, validate, and deploy
ansible-playbook -i inventory playbook.yml

# Validate only
ansible-playbook -i inventory validate.yml

# Upgrade 8.1.0 to 8.1.5 and validate
ansible-playbook -i inventory upgrade.yml
```

The upgrade playbook sets `eap_prospero_update: true`, which uses JBoss Installation Manager to apply available updates in the `eap:8.1.0` channel, bringing the server to 8.1.5.

## Cluster Configuration

The [`templates/eap_ymlconfig.yml.j2`](templates/eap_ymlconfig.yml.j2) template configures:

- A `JBOSS_ID` system property per node (set to `inventory_hostname`)
- Remote outbound socket bindings for each cluster member (dynamically looped over `eap_cluster_nodes`)
- JGroups `tcp` stack with `TCPPING` protocol replacing the default `MPING`
- JGroups `ee` channel pointed at the `tcp` stack

## Author

**Harsha Cherukuri** — Red Hat  
[hcheruku@redhat.com](mailto:hcheruku@redhat.com)  
GitHub: [hcherukuri](https://github.com/hcherukuri)
