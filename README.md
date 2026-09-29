# DevOps Technical Assessment

## Docker-Based Virtual Infrastructure with Ansible Automation and Hardened Nginx Reverse Proxy

### Project Overview

This project implements a small virtual infrastructure using Docker containers as virtual-machine stand-ins.

The infrastructure contains three Ubuntu-based containers:

* `vm1` — Nginx reverse proxy and HTTPS entry point
* `vm2` — Backend web server
* `vm3` — Backend web server

Ansible is used to automate server configuration, while Nginx provides reverse proxying, routing, load balancing, and failover.

The project also implements SSH hardening, firewall rules, local DNS-style hostnames, self-signed TLS certificates, and automated configuration management.

---

## Architecture

```text
                         Local Machine
                              |
                       Docker Network
                         vm_net
                    172.28.0.0/16
                              |
                    +---------+---------+
                    |                   |
                   vm1                vm2/vm3
              172.28.0.11        172.28.0.12
                    |             172.28.0.13
                    |
              Nginx Reverse Proxy
                    |
             +------+------+
             |             |
            /vm2          /vm3
             |             |
            vm2           vm3
             \             /
              \           /
                /app
             Load Balancing
```

---

## Technologies Used

* Docker
* Ubuntu
* Linux
* OpenSSH
* UFW / iptables
* Ansible
* Ansible Vault
* Nginx
* TLS / HTTPS
* Git
* GitHub

---

# 1. Docker Infrastructure

Three Ubuntu containers are created and treated as virtual machines.

| Container | IP Address  | Purpose             |
| --------- | ----------- | ------------------- |
| vm1       | 172.28.0.11 | Nginx reverse proxy |
| vm2       | 172.28.0.12 | Backend web server  |
| vm3       | 172.28.0.13 | Backend web server  |

All containers use the custom Docker network:

```text
Network: vm_net
```

The containers use fixed IP addresses and automatic restart configuration.

---

# 2. Docker Network

The containers are connected to the same custom Docker network.

Example network:

```bash
docker network inspect vm_net
```

The network uses a defined subnet rather than Docker's default automatically assigned network.

---

# 3. Creating the Containers

## VM1

VM1 exposes the ports required for the reverse-proxy entry point and SSH access.

```bash
docker run -d \
  --name vm1 \
  --net vm_net \
  --ip 172.28.0.11 \
  --restart always \
  -p 80:80 \
  -p 443:443 \
  -p 2221:22 \
  ubuntu-ssh
```

## VM2

```bash
docker run -d \
  --name vm2 \
  --net vm_net \
  --ip 172.28.0.12 \
  --restart always \
  ubuntu-ssh
```

## VM3

```bash
docker run -d \
  --name vm3 \
  --net vm_net \
  --ip 172.28.0.13 \
  --restart always \
  ubuntu-ssh
```

VM2 and VM3 are not published directly to host ports. They are accessed through the Docker network and Nginx reverse proxy.

---

# 4. Verify Containers

Check running containers:

```bash
docker ps
```

Check container IP addresses:

```bash
docker inspect -f '{{.Name}} -> {{(index .NetworkSettings.Networks "vm_net").IPAddress}}' vm1 vm2 vm3
```

```text
/vm1 -> 172.28.0.11
/vm2 -> 172.28.0.12
/vm3 -> 172.28.0.13
```

Check restart policy:

```bash
docker inspect -f '{{.Name}} -> {{.HostConfig.RestartPolicy.Name}}' vm1 vm2 vm3
```

Expected:

```text
/vm1 -> always
/vm2 -> always
/vm3 -> always
```

---
![image alt](https://github.com/hanamanttaranal-cpu/internship-v7-ai-assignment/blob/8071de5b5282d3f2ee8294dceefe54fbc98a1bf8/Screenshot%202026-09-28%20203528.png)


# 5. SSH Hardening

OpenSSH Server is configured on the containers.

Security configuration includes:

* Dedicated SSH user
* SSH key-based authentication
* Password authentication disabled
* Direct root SSH login disabled
* SSH access verified from the host
* Password login rejection tested

Example SSH connection:
![image alt](https://github.com/hanamanttaranal-cpu/internship-v7-ai-assignment/blob/10780e2069484e35f3a2961ccbe1eac61fc51fbf/Screenshot%202026-09-29%20111208.png)
```bash
ssh -p 2221 <user>@localhost
```

The SSH configuration is managed through Ansible.

---

# 6. Firewall Configuration

A host-level firewall is configured on each VM.

The firewall uses a default-deny inbound policy.

Only required ports are allowed.

### VM1

Required services include:

```text
SSH     22
HTTP    80
HTTPS   443
```

### VM2 / VM3

Backend services are available through the Docker network.

Their backend ports are not published directly to the host.

Firewall configuration is verified using:

```bash
ufw status verbose
```

The objective is to prevent unnecessary inbound access while allowing only required traffic.

---

# 7. Local Hostnames

Local hostnames are configured for the infrastructure:

```text
public.vm1.local
public.vm2.local
public.vm3.local
```

The local hosts file maps these names to the corresponding container IP addresses.

Example:

```text
172.28.0.11    public.vm1.local
172.28.0.12    public.vm2.local
172.28.0.13    public.vm3.local
```
![image alt](https://github.com/hanamanttaranal-cpu/internship-v7-ai-assignment/blob/f3146d1cc96a19757f0046302640b21f32f4bfa9/Screenshot%202026-09-29%20111744.png)

The hostnames are then used for testing SSH and web access.

---

# 8. TLS / HTTPS

A self-signed TLS certificate is configured for the local environment.

Nginx provides HTTPS access on VM1.

HTTP traffic is redirected to HTTPS.


```text
http://public.vm1.local
        |
        v
     HTTPS
        |
        v
https://public.vm1.local
```

Because this is a local assessment environment, a self-signed certificate is used instead of a publicly trusted certificate.

---

# 9. Ansible Automation

Ansible is used to automate configuration of all three containers.


```text
[servers]
vm1
vm2
vm3
```
![image alt](https://github.com/hanamanttaranal-cpu/internship-v7-ai-assignment/blob/f0f9aecad1c4e1b183757e832faca520a360f730/Screenshot%202026-09-29%20113622.png)

Ansible is configured to connect to the containers using SSH keys.

Connectivity is tested with:

```bash
ansible all -m ping
```

```text
vm1 | SUCCESS
vm2 | SUCCESS
vm3 | SUCCESS
```

---

# 10. Ansible Project Structure

```text
ansible/
├── ansible.cfg
├── inventory
├── site.yml
├── group_vars/
│   └── all/
│       └── vault.yml
└── roles/
    ├── common/
    ├── ssh/
    └── nginx/
```

---

# 11. Ansible Vault

Sensitive information is stored using Ansible Vault.


```bash
ansible-vault create group_vars/all/vault.yml
```

Secrets are not stored as plaintext in the repository.

The repository must never contain:

* Private SSH keys
* Plaintext passwords
* Vault passwords
* Other sensitive credentials
![image alt](https://github.com/hanamanttaranal-cpu/internship-v7-ai-assignment/blob/5374be4b8d05ea393aedecd4d3f79a3f9b1420af/Screenshot%202026-09-29%20114041.png)
---

# 12. Common Server Configuration

Ansible automates common packages and configuration.

The configuration includes required tools such as:

```text
curl
git
docker
```

A dedicated Ansible user is configured with appropriate sudo permissions.

---

# 13. Nginx Web Servers

Nginx is installed and configured on the required containers.

Each backend provides a dynamic page showing information such as:

* Hostname
* Current time
* Backend server identity

This allows load balancing and failover to be demonstrated clearly.

Example:
![image alt](https://github.com/hanamanttaranal-cpu/internship-v7-ai-assignment/blob/576db8cdc67244ee089038c37cb0e36a68d485e9/Screenshot%202026-09-29%20114441.png)

```text
Backend Server: vm2
Hostname: vm2
Current Time: ...
```

and:

```text
Backend Server: vm3
Hostname: vm3
Current Time: ...
```

---

# 14. Nginx Reverse Proxy

VM1 acts as the reverse proxy.

Requests are routed to the backend containers.

### VM2 route

```text
/vm2
   |
   v
vm2
```

### VM3 route

```text
/vm3
   |
   v
vm3
```

This allows the backend services to be accessed through VM1.

---

# 15. Load Balancing

The `/app` endpoint distributes requests between VM2 and VM3.

Example:

```text
                 VM1
            Nginx Reverse Proxy
                    |
                  /app
                    |
             +------+------+
             |             |
            VM2           VM3
```

Repeated requests can show responses from both backend servers.

Example:

```bash
for i in {1..10}; do
    curl -k https://public.vm1.local/app
    echo
done
```

The backend hostname in the response demonstrates which server handled each request.

---

# 16. Health and Failover Test

The infrastructure is tested by stopping one backend server.

### Step 1

Run repeated requests:

```bash
for i in {1..20}; do
    curl -k https://public.vm1.local/app
    sleep 1
done
```

### Step 2

Stop VM2:

```bash
docker stop vm2
```

### Step 3

Continue sending requests.

Traffic should continue to be served by VM3.

### Step 4

Start VM2 again:

```bash
docker start vm2
```

VM2 should become available again without manually reloading the Nginx configuration.

This demonstrates backend failure handling and recovery.

---

# 17. Ansible Idempotency

The Ansible playbook is executed twice.

First execution:

```bash
ansible-playbook site.yml
```

The first run performs the required configuration.

The playbook is then executed again:

```bash
ansible-playbook site.yml
```

The second execution should show no unnecessary changes.

Example:

```text
changed=0
```

This demonstrates Ansible idempotency.

---

# 18. Verification

Important verification commands:

### Docker

```bash
docker ps
```

### Network

```bash
docker network inspect vm_net
```

### Container IPs

```bash
docker inspect -f '{{.Name}} -> {{(index .NetworkSettings.Networks "vm_net").IPAddress}}' vm1 vm2 vm3
```

### Restart policy

```bash
docker inspect -f '{{.Name}} -> {{.HostConfig.RestartPolicy.Name}}' vm1 vm2 vm3
```

### Ansible connectivity

```bash
ansible all -m ping
```

### Firewall

```bash
ufw status verbose
```

### Nginx

```bash
nginx -t
```

### HTTPS

```bash
curl -k https://public.vm1.local
```

### Failover

```bash
docker stop vm2
```

Then continue testing:

```bash
curl -k https://public.vm1.local/app
```

---

# 19. Project Directory

Recommended repository structure:

```text
devops-technical-assessment/
│
├── README.md
│
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
│
├── ansible/
│   ├── ansible.cfg
│   ├── inventory
│   ├── site.yml
│   ├── group_vars/
│   │   └── all/
│   │       └── vault.yml
│   └── roles/
│       ├── common/
│       ├── ssh/
│       └── nginx/
│
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
│
├── scripts/
│   ├── setup.sh
│   └── test-failover.sh
│
└── screenshots/
    ├── docker-containers.png
    ├── network.png
    ├── ssh-security.png
    ├── ansible-ping.png
    ├── ansible-idempotency.png
    ├── nginx.png
    ├── load-balancing.png
    └── failover.png
```

Adjust this structure to match the files actually present in your repository.

---

# 20. Screenshots / Evidence

The following evidence is included in the project documentation:

1. Three Docker containers running
2. Custom Docker network and subnet
3. Fixed container IP addresses
4. Automatic restart
5. SSH key-based login
6. Password login rejection
7. Root SSH login disabled
8. Firewall rules
9. Local `.local` hostname resolution
10. HTTPS and HTTP-to-HTTPS redirect
11. Ansible ping
12. First Ansible playbook execution
13. Second Ansible execution showing no unnecessary changes
14. Nginx backend pages
15. `/vm2` routing
16. `/vm3` routing
17. `/app` load balancing
18. VM2 failure
19. Continued traffic through VM3
20. VM2 recovery

---

# 21. Demo Video

A demonstration video is provided showing the main project functionality.

The video demonstrates:

* Docker infrastructure
* SSH security
* Firewall
* Local domains
* HTTPS
* Ansible automation
* Idempotency
* Nginx reverse proxy
* Load balancing
* Backend failure
* Automatic recovery

Demo Video:

**[Add Google Drive video link here]**

Make sure the Google Drive sharing permission is:

```text
Anyone with the link → Viewer
```

---

# 22. Security Notes

Sensitive information is not committed to GitHub.

Do not commit:

```text
*.pem
*.key
id_rsa
id_ed25519
vault-password
.env
plaintext passwords
```

Use Ansible Vault for secrets.

---

# 23. Conclusion

This project demonstrates a local DevOps infrastructure using Docker containers as virtual-machine stand-ins.

The environment combines:

* Docker networking
* Linux administration
* SSH hardening
* Firewall configuration
* Ansible automation
* Ansible Vault
* Nginx
* Reverse proxy
* HTTPS
* Load balancing
* Health and failover testing
* Infrastructure documentation

The project is designed to demonstrate practical DevOps skills in a zero-cost local environment.
