# Semaphore – Ansible Automation with CI/CD

This repository documents the installation and configuration of Semaphore running via Docker, as well as its integration with GitHub Actions for the automatic execution of Ansible playbooks.

The goal is to allow changes on GitHub to trigger the controlled and automated execution of Ansible tasks.


## Prerequisites

To replicate this setup it is necessary:

- A Linux machine (VM or bare metal)
- Docker installed
- Access to a GitHub repository
- Basic knowledge:
  - Ansible
  - Docker
  - Git/GitHub


## Semaphore Docker Installation

Semaphore is executed locally through Docker, using SQLite as a database to simplify the setup.

Execute the following command (Note: Weak credentials are only for testing purpose):


```bash
sudo docker run --name semaphore \
--restart unless-stopped \
-p 3000:3000 \
-v semaphore_data:/var/lib/semaphore \
-e SEMAPHORE_DB_DIALECT=sqlite \
-e SEMAPHORE_ADMIN=admin \
-e SEMAPHORE_ADMIN_PASSWORD=Temporario@2024 \
-e SEMAPHORE_ADMIN_NAME="Admin" \
-e SEMAPHORE_ADMIN_EMAIL=admin@localhost \
-d public.ecr.aws/semaphore/pro/server:v2.16.47
```

After the container starts, Semaphore is available in:

http://<TARGET_IP>:3000


## Initial Login

Initial Credentials:

- Username: admin
- Password: Temporario@2024

It is recommended to change the password after first login.

## Ansible Playbooks Repository

The Ansible playbooks are not included in this repository.

Semaphore uses the following repository:

- https://github.com/nunof86/ansible-playbooks

The Ansible playbook used was the following:

- debian-based/system_administration/teste.yml

## Semaphore Configuration Overview

The following elements were configured in Semaphore:

- Repository: GitHub repository with Ansible playbooks
- Inventory: SSH inventory pointing to the target VM
- Key Store: SSH key for authentication
- Task Template: Ansible template that executes the update playbook

### Repository

- Name: Ansible_Playbooks
- URL: https://github.com/nunof86/ansible-playbooks.git
- git: HTTPS
- Main: main
- Access Key: None

### Key Store

- Key Name: Devops_User
- Type: SSH Key
- Username (Optional): devops
- Private Key: content of the private key file (/home/devops/.ssh/id_ed25519 in this case)

### Inventory

- Name: Servers
- User_Credentials: Devops_User
- Type: Static YAML:
  ```bash
  [servers]
  <TARGET_IP> ansible_user=devops
  ```

### Task Template

- Name: System_Update
- Path to playbook file: debian-based/system_administration/system_update.yml
- Inventory: Servers
- Repository: Ansible_Playbooks
- Variable Group: Empty

## Pre-flight Checklist & Troubleshooting

Before executing any Ansible playbook through Semaphore, the following checks must pass successfully on the target environment.

These steps ensure that SSH access, privilege escalation, and Ansible configuration are correctly set.

### 1. SSH Key Authentication to Target VM

<strong>Goal:</strong> Ensure the Ansible controller can connect to the target host via SSH without password prompts.

<strong>Verify SSH Access</strong>

```bash
ssh devops@<TARGET_IP>
```

Expected behavior:
- login succeeds without asking for a password
- User is logged in as `devops`

If this fails:
- Ensure the public SSH key is present in:
```bash
/home/devops/.ssh/authorized_keys
```

- Ensure correct permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> **Note:** Even if the controller and target are the same VM (`localhost`-like scenario), SSH key authentication is still required because Ansible uses SSH by default.


### 2. Ansible Can Reach the Inventory Host 

> **Note:** Change or create (if doesn't exists) the `inventories/ssh.ini` file to match the IP of the Target VM.

<strong>Goal:</strong> Verify that Ansible can connect to the inventory hosts using SSH. 

<strong>Test connectivity with Ansible ping</strong>

```bash
ansible -i inventories/ssh.ini all -m ping -vv
```

Expected output:

```bash
SUCCESS => "pong"
```

If this fails:
- Verify the inventory file:

```bash
[servers]
<TARGET_IP> ansible_user=devops
```

- Confirm SSH works manually (`ssh devops@<TARGET_IP>`)
- Ensure the correct SSH user is defined

### 3. Passwordless Sudo Configuration

<strong>Goal:</strong> Ensure Ansible can escalate privileges without prompting for a sudo password.

<strong>Configure passwordless sudo</strong>

```bash
echo 'devops ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/devops-nopasswd
sudo chmod 440 /etc/sudoers.d/devops-nopasswd
sudo visudo -cf /etc/sudoers.d/devops-nopasswd
```

Expected output:

```bash
/etc/sudoers.d/devops-nopasswd: parsed OK
```

If this fails:
- Do <strong>not</strong> continue
- Fix sudo configuration first to avoid playbook failures

### 4. Validate Ansible Privilege Escalation

<strong>Goal:</strong> Confirm Ansible can run commands as `root` using `--become`.

<strong>Test sudo with Ansible</strong>

```bash
ansible -i inventories/ssh.ini all -m command -a "whoami" --become -vv
```

Expected output:

```bash
root
```

If you see:

```bash
Missing sudo password
```

Then:
- Passwordless sudo is not correctly configured
- Recheck Step 3

### 5. Execute the Ansible Playbook Locally

<strong>Goal:</strong> Ensure the playbook itself works <strong>before running it in Semaphore</strong>.

```bash
ansible-playbook -i inventories/ssh.ini debian-based/system_administration/system_update.yml
```

Expected result:
- All tasks complete successfully
- No SSH or sudo errors
- No unreachable hosts

### Why This Checklist Matters

If <strong>any</strong> of the steps above fail:
- Semaphore will also fail
- Semaphore error messages may be misleading
- Troubleshooting becomes harder

> **Note:** Rule of thumb: <strong>If does not work locally with Ansible, it will not work in Semaphore.</strong>

## CI/CD - GitHub Actions Triggering Semaphore

This section configures a self-hosted runner and a GitHub Actions workflow that triggers a Semaphore task via API.

### API Token (Semaphore)

Create an API token in Semaphore UI (Admin user) and validate it locally:

```bash
export SEMAPHORE_URL="http://<TARGET_IP>:3000"
export SEMAPHORE_TOKEN="<TOKEN>"

curl -sS -o /dev/null -w "HTTP=%{http_code}\n" \
  -H "Authorization: Bearer ${SEMAPHORE_TOKEN}" \
  -H "Accept: application/json" \
  "${SEMAPHORE_URL}/api/projects"
```

Expected:
- `HTTP=200`

### GitHub Secrets

Create the following <strong>Repository Secrets:</strong>
- `SEMAPHORE_URL` = `http://<TARGET_IP>:3000`
- `SEMAPHORE_TOKEN` = Semaphore API token (no quotes)
- `SEMAPHORE_PROJECT_ID` = `<PROJECT_ID>`
- `SEMAPHORE_TEMPLATE_ID` = `<TEMPLATE_ID>`

### Self-hosted Runner (Homelab)

In GitHub:
Settings -> Actions -> Runners -> New self-hosted runner

Follow GitHub's instructions on the VM.

Recommended (service mode):

```bash
sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```

Runner should appear as <strong>Idle</strong> in Github.

### GitHub Actions Workflow

Create the file:

`.github/workflows/semaphore-cd.yml`

#### Workflow content:

```bash
name: Trigger Semaphore

on:
  push:
    branches: [ "main" ]
  workflow_dispatch: {}

jobs:
  trigger:
    runs-on: [self-hosted, Linux, X64]

    steps:
      - name: Trigger Semaphore task
        shell: bash
        env:
          SEMAPHORE_URL: ${{ secrets.SEMAPHORE_URL }}
          SEMAPHORE_TOKEN: ${{ secrets.SEMAPHORE_TOKEN }}
          PROJECT_ID: ${{ secrets.SEMAPHORE_PROJECT_ID }}
          TEMPLATE_ID: ${{ secrets.SEMAPHORE_TEMPLATE_ID }}
        run: |
          set -euo pipefail

          echo "Checking Semaphore authentication..."
          auth_code="$(curl -sS -o /dev/null -w "%{http_code}" \
            -H "Authorization: Bearer ${SEMAPHORE_TOKEN}" \
            -H "Accept: application/json" \
            "${SEMAPHORE_URL}/api/projects")"

          if [[ "${auth_code}" != "200" ]]; then
            echo "Semaphore authentication failed"
            exit 1
          fi

          echo "Creating Semaphore task..."
          curl -sS -X POST \
            -H "Authorization: Bearer ${SEMAPHORE_TOKEN}" \
            -H "Content-Type: application/json" \
            -d "{\"template_id\": ${TEMPLATE_ID}}" \
            "${SEMAPHORE_URL}/api/project/${PROJECT_ID}/tasks"
```

Expected result:
- GitHub Action -> <strong>green (passed)</strong>
- Semaphore -> new task visible and executed
- Playbook runs exactly as tested locally

#### Common Failures

401/403 -> Wrong token in GitHub Secrets
404 -> Wrong URL or missing `http://`
Runner idle -> Workflow misconfigured
Semaphore task unreachable -> SSH/sudo not validated locally


## Final Commit and CI/CD Validation

After completing all the configuration steps (Semaphore, Ansible, SSH, sudo, self-hosted runner and GitHub Actions Workflow), a final commit is required to validate the full CI/CD pipeline.

### 1. Commit the Changes

From the repository root directory, stage and commit the changes:

```
git add .
git commit -m "Finalize Semaphore CI/CD setup documentation"
```

Expected result:
- A clean commit is created on the `main` branch
- No Git configuration or identity errors occur

### 2. Push to GitHub

Push the commit to the remote repository:

```bash
git push origin main
```
This action automatically triggers the GitHub Actions workflow.

### 3. Validate GitHub Actions Execution

In the GitHub repository:
1. Go to Actions
2. Open the latest workflow run
3. Confirm:
    - The self-hosted runner is selected
    - All steps complete successfully (green checkmark)

If the workflow fails:
- Review the logs in the Trigger Semaphore task step
- Confirm that all secrets are correctly configured.

### 4. Validate Semaphore Task Execution

In the Semaphore UI:
1. Open the configured project
2. Navigate to <strong>Tasks</strong>
3. Confirm:
    - A new task was created automatically
    - Task status is `Success`
    - The Ansible playbook executed without errors.

### End-to-End Validation Summary

This project implements a **fully automated and validated CI/CD pipeline**.

The delivery flow is as follows:

- Source code changes are committed and pushed to GitHub
- A **GitHub Actions self-hosted runner** is triggered
- The runner communicates with the **Semaphore API**
- A **Semaphore task** is executed
- Infrastructure is configured via **Ansible playbooks**
- Changes are applied to the **target virtual machine**

✔ All stages have been validated successfully, ensuring a reliable and production-ready CI/CD workflow.
