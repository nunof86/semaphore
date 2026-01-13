# Semaphore – Ansible Automation with CI/CD

This repository documents the installation and configuration of Semaphore running via Docker, as well as its integration with GitHub Actions for the automatic execution of Ansible playbooks.

The goal is to allow changes on GitHub to trigger the controlled and automated execution of Ansible tasks.


## Prerequisites

To replicate this setup it is necessary:

- A Linux machine (VM or bare metal)
- Docker installed
- Acess to a GitHub repository
- Acesso a um repositório GitHub
- Basic knowledge:
  - Ansible
  - Docker
  - Git/GitHub


## Semaphore Docker Installation

Semaphore is executed localy trough Docker, using SQLite as database to simplify the setup.

Execute the following command (Note: Weak credentials is only for testing purpuse):


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

After the container start, the Semaphore is available in:

http://<VM_IP>:3000


## Initial Login

Initials Credentials:

- Username: admin
- Password: Temporario@2024

It is recommended to change the password after first login.

## Ansible Playbooks Repository

The Ansible playbooks are not included in this repository.

Semaphore uses the follow repository:

- https://github.com/nunof86/ansible-playbooks

The Ansible playbooks uses was the follow:

- debian-based/system_administration/teste.yml

## Semaphore Configuration Overview

The following elements were configured in Semaphore:

- Repository: GitHub repository with Ansible playbooks
- Inventory: SSH inventory pointing to the target VM
- Key Store: SSH key for authentication
- Task Template: Ansible template that executes the update playbook
