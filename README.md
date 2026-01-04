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

Execute the following command:


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
