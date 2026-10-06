# Site24x7 Agent Setup — Infrastructure Configuration Guide

This document describes how Site24x7 monitoring agents are installed and configured on EC2 instances as part of the Cumulus infrastructure post-provisioning pipeline.

---

## Overview

Site24x7 is the server monitoring solution used to track EC2 instance health, performance, and availability. The agent is installed automatically during the post-EC2 provisioning phase via Ansible, with the device key securely managed through AWS SSM Parameter Store.

---

## Architecture

```
Jenkins Pipeline
      │
      ▼
base_postInfra.sh
      │
      ├── Reads SITE24x7_KEY (SSM parameter path) from env
      ├── Fetches device key from AWS SSM Parameter Store (encrypted)
      │
      ▼
Ansible: install_apps.yml
      │
      ▼
Ansible: site24x7-install.yml
      │
      ├── Checks if /opt/site24x7/monagent already exists
      ├── Downloads Site24x7InstallScript.sh from Site24x7 CDN
      ├── Installs agent with device key (-key flag)
      └── Verifies installation
```

---

## Prerequisites

- AWS CLI access with SSM read permissions in the target account
- Ansible installed on the Jenkins build agent
- Site24x7 device key stored in AWS SSM Parameter Store (encrypted `SecureString`)
- EC2 instances reachable via SSH using the configured Ansible key

---

## Configuration

### Environment Variables

| Variable | Description | Required |
|---|---|---|
| `EC2_INSTALL_TAGS` | Comma-separated list of install tags. Must include `site24x7` to trigger agent installation. | Yes |
| `SITE24x7_KEY` | SSM Parameter Store path where the Site24x7 device key is stored (e.g. `/cumulus/site24x7/device-key`) | Yes (if site24x7 tag present) |
| `AWS_REGION` | AWS region to fetch the SSM parameter from | Yes |
| `ALLOW_POST_EC2_SETUP_INSTALL` | Set to `true` to force reinstall even if agent is already present | No (default: false) |

### SSM Parameter Store

The Site24x7 device key is stored as an encrypted `SecureString` in AWS SSM Parameter Store. The path is defined per environment via the `SITE24x7_KEY` environment variable.

To create/update the key:
```bash
aws ssm put-parameter \
  --name "/your/ssm/path/site24x7-key" \
  --value "us_<your_device_key>" \
  --type SecureString \
  --region <aws-region> \
  --overwrite
```

---

## Installation Flow

### 1. Pipeline Entry Point — `base_postInfra.sh`

The `installPackages()` function controls whether Site24x7 is installed:

```bash
# Only runs if EC2_INSTALL_TAGS contains "site24x7" AND SITE24x7_KEY is set
if echo "$EC2_INSTALL_TAGS" | grep -q "site24x7" && [ ! -z "$SITE24x7_KEY" ]; then
    export SITE24_KEY=$(aws ssm get-parameter \
        --name ${SITE24x7_KEY} \
        --region "${AWS_REGION}" \
        --with-decryption \
        --query "Parameter.Value" \
        --output text)
fi
```

The fetched key is then passed to Ansible as an extra variable:
```bash
playBookWithTags \
  "site24x7_key=${SITE24_KEY} ..." \
  "${EC2_INSTALL_TAGS}" \
  install_apps.yml
```

### 2. Ansible Orchestration — `install_apps.yml`

```yaml
- hosts: '{{host_tag}}'
  tasks:
    - name: Install Site24x7
      tags: site24x7
      include_tasks: site24x7-install.yml
```

### 3. Agent Installation — `site24x7-install.yml`

```yaml
# Step 1: Check if agent is already installed
- name: Check linux server monitoring agent exists
  stat:
    path: /opt/site24x7/monagent
  register: installed

# Step 2: Download the install script (skipped if already installed)
- name: Download Site24x7 install script
  get_url:
    url: "https://staticdownloads.site24x7.com/server/Site24x7InstallScript.sh"
    dest: "/tmp/Site24x7InstallScript.sh"
    mode: '0755'
  when: installed.stat.exists == False or override == "true"

# Step 3: Install the agent
- name: Install Site24x7 Agent
  command: "bash /tmp/Site24x7InstallScript.sh -i -key={{ site24x7_key }} -automation=true"
  no_log: true   # prevents key from appearing in logs
  when: installed.stat.exists == False or override == "true"

# Step 4: Verify installation
- name: Verify Site24x7 Installation
  stat:
    path: "/opt/site24x7/monagent"
  register: site24x7_installed

- name: Fail if Site24x7 installation failed
  fail:
    msg: "Site24x7 installation failed."
  when: not site24x7_installed.stat.exists
```

---

## File Reference

| File | Location | Purpose |
|---|---|---|
| `base_postInfra.sh` | `shell/postInfra/` | Fetches SSM key, triggers Ansible |
| `install_apps.yml` | `ansible/postInfra/` | Ansible orchestration playbook |
| `site24x7-install.yml` | `ansible/postInfra/` | Site24x7 agent installation tasks |

---

## How to Enable Site24x7 on a New Environment

1. **Store the device key in SSM**
   ```bash
   aws ssm put-parameter \
     --name "/your-env/site24x7/device-key" \
     --value "us_<key_from_site24x7_portal>" \
     --type SecureString \
     --region <region> \
     --overwrite
   ```

2. **Set Jenkins environment variables** in the pipeline config:
   ```
   EC2_INSTALL_TAGS = site24x7
   SITE24x7_KEY    = /your-env/site24x7/device-key
   AWS_REGION      = us-east-1
   ```

3. **Run the post-infra pipeline** — the agent will be installed automatically on all EC2 instances in that environment.

4. **Verify** by logging into the [Site24x7 portal](https://www.site24x7.com) and confirming the servers appear under **Server Monitor → Monitors**.

---

## Reinstalling the Agent

To force reinstall on instances where the agent is already present, set:
```
ALLOW_POST_EC2_SETUP_INSTALL = true
```
This sets `override=true` in the Ansible run, bypassing the "already installed" check.

---

## Troubleshooting

| Issue | Resolution |
|---|---|
| Agent not appearing in Site24x7 portal | Check `/opt/site24x7/monagent` exists on the EC2; check agent service status: `systemctl status site24x7monagent` |
| SSM fetch failing | Ensure the IAM role on the Jenkins agent has `ssm:GetParameter` permission for the key path |
| Installation skipped silently | Verify `EC2_INSTALL_TAGS` includes `site24x7` and `SITE24x7_KEY` is set |
| Pipeline fails with "installation failed" | Check `/tmp/Site24x7InstallScript.sh` logs on the EC2 instance; verify device key is valid |

---

## Security Notes

- The device key is **never logged** — the Ansible task uses `no_log: true`
- The key is fetched at runtime from SSM with `--with-decryption`, never stored in code or config files
- SSM parameter type must be `SecureString` (KMS-encrypted)
