# Docker-Based Virtual Infrastructure with Ansible Automation and Hardened Nginx Reverse Proxy

## Overview

This project implements a containerized DevOps infrastructure using Docker, Ansible, SSH hardening, UFW firewall rules, Nginx reverse proxy, HTTPS, load balancing, and automatic backend failover.

The infrastructure consists of three Ubuntu containers connected through a custom Docker network.

## Architecture

```text
                         Windows Host
                              |
                       Local Hosts File
                              |
                    public.vm1.local
                              |
                       HTTPS :18443
                              |
                         +---------+
                         |   vm1   |
                         |  Nginx  |
                         +----+----+
                              |
                +-------------+-------------+
                |                           |
             /vm2 /vm3                   /app
                |                           |
                |                    Load Balancing
                |                           |
          +-----+-----+              +------+------+
          |           |              |             |
       +--+--+     +--+--+        +--+--+       +--+--+
       | vm2 |     | vm3 |        | vm2 |       | vm3 |
       |Nginx|     |Nginx|        |Backend|     |Backend|
       +-----+     +-----+        +------+       +------+

                 Docker Network: devops_net
                    172.20.0.0/24
```

## Infrastructure

| Container | IP Address  | Role                                      |
| --------- | ----------- | ----------------------------------------- |
| vm1       | 172.20.0.10 | Nginx reverse proxy, HTTPS, load balancer |
| vm2       | 172.20.0.20 | Backend web server                        |
| vm3       | 172.20.0.30 | Backend web server                        |

All three containers use the custom Docker network:

```text
devops_net
Subnet: 172.20.0.0/24
Gateway: 172.20.0.1
```

## Technologies Used

* Docker
* Ubuntu 24.04
* Ansible
* Nginx
* OpenSSH
* UFW
* TLS/HTTPS
* Git
* GitHub
* Bash

# Security Implementation

## SSH Hardening

SSH access is configured using public-key authentication.

The following SSH security settings are applied:

```text
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
```

This prevents password-based SSH authentication and direct root login.

SSH access is tested using the `devops` and `ansible_user` accounts with SSH keys.

## Firewall Configuration

UFW is enabled on all three containers.

The default inbound policy is deny:

```text
Default: deny (incoming)
Default: allow (outgoing)
```

### vm1

vm1 allows the ports required for the reverse proxy and HTTPS service:

```text
22/tcp
80/tcp
443/tcp
```

### vm2 and vm3

The backend servers are protected from external access.

SSH is allowed for administration, while HTTP access is restricted to vm1:

```text
22/tcp
80/tcp from 172.20.0.10
```

Therefore, vm2 and vm3 are not directly exposed through host HTTP/HTTPS ports.

## Secrets Protection

Ansible Vault is used to encrypt sensitive variables.

The Vault file is stored as:

```text
ansible/group_vars/all/vault.yml
```

The encrypted Vault file is committed to the repository, while private SSH keys and other sensitive files are excluded through `.gitignore`.

# Nginx HTTPS Configuration

Nginx is configured on vm1 as the main entry point.

A self-signed TLS certificate is created for:

```text
public.vm1.local
```

HTTP requests are redirected to HTTPS:

```text
HTTP :80
   |
   v
301 Redirect
   |
   v
HTTPS :443
```

The local HTTPS service is accessed through the host mapping:

```text
public.vm1.local
```

# Nginx Reverse Proxy

vm1 acts as a reverse proxy for the backend containers.

### vm2

Requests to:

```text
https://public.vm1.local:18443/vm2/
```

are forwarded to:

```text
172.20.0.20:80
```

### vm3

Requests to:

```text
https://public.vm1.local:18443/vm3/
```

are forwarded to:

```text
172.20.0.30:80
```

This allows vm1 to provide a single entry point to the backend servers.

# Nginx Load Balancing

The `/app/` endpoint uses an Nginx upstream group:

```nginx
upstream backend_pool {
    server 172.20.0.20:80 max_fails=3 fail_timeout=5s;
    server 172.20.0.30:80 max_fails=3 fail_timeout=5s;
}
```

Requests to:

```text
https://public.vm1.local:18443/app/
```

are distributed between vm2 and vm3.

The upstream configuration allows Nginx to detect backend failures and temporarily stop sending traffic to an unhealthy backend.

# Load Balancing Failover

The infrastructure supports backend failover without manually changing the Nginx configuration.

### Normal Operation

```text
Client
  |
  v
vm1 Nginx
  |
  +------> vm2
  |
  +------> vm3
```

Traffic is distributed across both backend servers.

### vm2 Failure

When vm2 is stopped:

```text
Client
  |
  v
vm1 Nginx
  |
  X------> vm2
  |
  +------> vm3
```

Nginx detects connection failures from vm2 and continues serving `/app/` through vm3.

### vm2 Recovery

After vm2 is restarted:

```text
Client
  |
  v
vm1 Nginx
  |
  +------> vm2
  |
  +------> vm3
```

vm2 becomes available again without requiring a manual Nginx configuration change or reload.

## Failover Verification

The failover process is tested by:

1. Sending repeated requests to `/app/`.
2. Stopping vm2.
3. Confirming that requests continue through vm3.
4. Restarting vm2.
5. Confirming that vm2 becomes available again.
6. Verifying that no manual Nginx reload is required.

# Ansible Automation

Ansible is used to configure all three containers.

The project uses reusable roles:

```text
common
ssh
nginx
```

The `common` role handles:

* Package installation
* `ansible_user` creation
* Passwordless sudo
* SSH public key configuration

The `ssh` role handles SSH hardening.

The `nginx` role handles:

* Nginx installation
* Nginx service
* Dynamic web pages
* Reverse proxy configuration on vm1

## Ansible Vault

Sensitive variables are stored using Ansible Vault:

```text
ansible/group_vars/all/vault.yml
```

The Vault file is encrypted and is not stored as plaintext.

## Idempotence

The playbook is designed to be idempotent.

Running the playbook again after the configuration is already applied should result in no unnecessary configuration changes.

# Local Domain Configuration

The following local domains are used:

```text
public.vm1.local
public.vm2.local
public.vm3.local
```

They are mapped through the Windows hosts file.

The containers also use their Docker network addresses for internal communication.

# Project Structure

```text
docker-ansible-nginx-assessment/
├── ansible/
│   ├── ansible.cfg
│   ├── group_vars/
│   ├── inventory/
│   ├── roles/
│   │   ├── common/
│   │   ├── nginx/
│   │   └── ssh/
│   └── site.yml
├── docker/
│   └── Dockerfile
├── nginx/
├── scripts/
├── .gitignore
└── README.md
```

# Verification

The implementation verifies:

* Docker custom network
* Container IP addresses
* Container restart policies
* SSH key-only authentication
* Disabled password authentication
* Disabled root SSH login
* UFW firewall rules
* Local domain resolution
* HTTPS
* HTTP to HTTPS redirection
* Ansible connectivity
* Ansible Vault encryption
* Ansible idempotence
* Nginx dynamic pages
* Nginx reverse proxy
* Nginx load balancing
* Backend health/failure detection
* vm2 to vm3 failover
* Backend recovery and rejoining

# Conclusion

This project demonstrates a containerized DevOps environment with automated configuration management, security hardening, HTTPS, reverse proxying, load balancing, and backend failover using Docker, Ansible, and Nginx.

## Author

**Adhithya P S**

