# 🚀 Hosted n8n on Azure cloud

![n8n](https://img.shields.io/badge/Workflow-n8n-ff6d5a?style=for-the-badge&logo=n8n)
![Azure](https://img.shields.io/badge/Cloud-Azure-0078D4?style=for-the-badge&logo=microsoft-azure)
![Docker](https://img.shields.io/badge/Container-Docker-2496ED?style=for-the-badge&logo=docker)
![Nginx](https://img.shields.io/badge/Proxy-Nginx-009639?style=for-the-badge&logo=nginx)
![Ubuntu](https://img.shields.io/badge/OS-Ubuntu_24.04-E95420?style=for-the-badge&logo=ubuntu)

This repository documents the infrastructure and deployment process for a self-hosted instance of **n8n (Workflow Automation Tool)** running on **Microsoft Azure**.

This project transitions from standard SaaS usage to a fully controlled, self-hosted environment. The goal was to leverage custom configurations, unrestricted execution times

---

## 📖 Table of Contents
* [Overview & Resources](#-overview--resources)
* [Architecture & Stack](#%EF%B8%8F-architecture--stack)
* [Infrastructure Setup (Azure)](#%EF%B8%8F-infrastructure-setup-azure-portal)
* [Installation Guide](#%EF%B8%8F-installation--setup-guide)
* [Configuration (Nginx & SSL)](#%EF%B8%8F-configuration-nginx--ssl)
* [Deployment](#-deployment-docker)
* [My Challenges & Solutions](#-challenges--solutions-i-found-while-deploying)
* [Future Roadmap](#-future-roadmap)
---

## 🎓 Overview & Resources

Leveraging the **[GitHub Student Developer Pack](https://education.github.com/pack)**, I utilized industry-standard tools to build this professional infrastructure with **$0 upfront cost**.

* **Microsoft Azure:** Utilized the **Azure for Students** subscription. This includes **$100 in credits** and access to **popular services for free for 12 months**, allowing for a robust, cost-free hosting environment.
* **Name.com:** Used the free domain credit for a custom endpoint (`n8n.your-domain.dev`).
* **Google AI Studio:** Utilized the free tier for Gemini AI integration.

---

## 🏗️ Architecture & Stack

* **Cloud Provider:** Microsoft Azure (Virtual Machine)
* **Operating System:** Ubuntu Server 24.04 LTS
* **Machine Size:** `Standard B2ats v2` (Burstable) - *Selected for optimal cost-to-performance ratio.*
* **Region:** **West US** - *Crucial selection to ensure availability of Google AI Studio/Gemini API services, which are often geo-locked in Asian servers.*
* **Container Runtime:** Docker
* **Reverse Proxy:** Nginx
* **Security (SSL/TLS):** Certbot (Let's Encrypt)

---

## ☁️ Infrastructure Setup (Azure Portal)

Before installing software, the Virtual Machine (VM) must be provisioned correctly in the Microsoft Azure Portal.

### 1. Create Resource
* Search for **"Virtual Machine"** and click **Create**.
* **Subscription:** Select "Azure for Students".
* **Resource Group:** Create new (e.g., `rg-n8n-hosting`).

### 2. Instance Details
* **Virtual Machine Name:** `n8n-server`
* **Region:** `(US) West US` (Required for Gemini API support).
* **Availability Options:** No infrastructure redundancy required.
* **Security Type:** Standard.
* **Image:** `Ubuntu Server 24.04 LTS - x64 Gen2`.
* **Size:** `Standard B2ats v2` (2 vCPUs, 1 GiB memory) or similar burstable size eligible for free credit.

### 3. Administrator Account
* **Authentication type:** SSH Public Key.
* **Username:** `azureuser` (Default).
* **Key Source:** Generate new key pair (Download the `.pem` file when prompted).

### 4. Networking (Crucial Step)
* **Public IP:** Create new (Standard).
* **NIC Network Security Group:** Advanced.
* **Inbound Port Rules:** You must allow traffic on the following ports:
    * `22` (SSH) - For remote connection.
    * `80` (HTTP) - For initial web access.
    * `443` (HTTPS) - For secure SSL connection.

Click **Review + create** to launch the VM.

---

## 🛠️ Installation & Setup Guide

### 1. System Prerequisites
Once the VM is running, open PowerShell on your local machine and connect using the SSH key downloaded earlier:

```powershell
ssh -i "path\to\your-key.pem" azureuser@<VM-Public-IP>
```


Update package lists and upgrade existing packages
```bash
sudo apt update && sudo apt upgrade -y
```
Install essential dependencies
```bash
sudo apt install docker.io nginx certbot python3-certbot-nginx -y
```

### 2. User Permissions
To manage Docker containers without using `sudo` for every command (Standard Security Practice):

```bash
sudo usermod -aG docker $USER
```
**NOTE: You must log out and log back in for this group change to take effect.**

---

## ⚙️ Configuration (Nginx & SSL)

### 1. SSL Setup (HTTPS)
Secure the domain using Certbot. This automatically handles certificate generation and Nginx configuration.

```bash
sudo certbot --nginx -d "your.domain.com"
```

### 2. Nginx Reverse Proxy Optimization
**Critical Step:** Default Nginx settings often time out during long-running AI workflows (e.g., generating long-form text with Gemini).

Edit the configuration file:
`sudo nano /etc/nginx/sites-available/default`

Add/Modify the `location /` block:

```nginx
server {
    server_name your.domain.com;

    location / {
        # Increased timeouts for long AI generation tasks (5 minutes)
        proxy_read_timeout 300s;
        proxy_connect_timeout 300s;
        proxy_send_timeout 300s;

        # WebSocket Support (Required for n8n UI)
        proxy_pass http://localhost:5678;
        proxy_set_header Connection '';
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        
        # Forwarding Real IP
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # ... SSL configuration managed by Certbot ...
}
```

Verify Nginx syntax and reload:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

## 🚀 Deployment (Docker)

The following command launches the n8n container with specific environment variables for timezone and security.

* `--restart unless-stopped`: Ensures n8n auto-starts if the server reboots.
* `N8N_SECURE_COOKIE=false`: Required when running behind a reverse proxy handling SSL.
* `TZ=Asia/Kolkata`: Sets the internal clock to IST for accurate scheduling.

```bash
# Run on Azure VM (Ubuntu)
docker run -d --name n8n \
  -p 5678:5678 \
  -v /home/azureuser/n8n-data:/home/node/.n8n \
  -e N8N_SECURE_COOKIE=false \
  -e N8N_EDITOR_BASE_URL=https://n8n.your-domain.dev \
  -e TZ=Asia/Kolkata \
  --restart unless-stopped \
  n8nio/n8n
```

> **Documentation Reference:** [n8n Docker Environment Variables](https://docs.n8n.io/hosting/environment-variables/)

---

## 🚧 Challenges & Solutions (I found While Deploying)

Deploying this infrastructure revealed several real-world DevOps challenges.

### 1. Google Gemini AI Region Locking
* **Issue:** The free tier of Google AI Studio (Gemini API) returned `404/Service Unavailable` errors when hosted on my initial **East Asia** Azure VM due to geo-restrictions.
* **Solution:** Migrated the entire infrastructure to the **West US** region. This required a full server migration and data restoration.

### 2. Data Migration & Backup
* **Task:** Moving workflow data from the old VM to the new US-based VM without data loss.
* **Method:** Used `tar` to archive the Docker volume and `scp` (Secure Copy Protocol) to transfer data using a local machine as a bridge.

```bash
# 1. Backup on Old Server
tar -czf ~/n8n-data-backup.tgz -C /home/azureuser n8n-data

# 2. Transfer to Local Machine (Local Terminal)
scp azureuser@old-ip:~/n8n-data-backup.tgz .

# 3. Upload to New Server (Local Terminal)
scp n8n-data-backup.tgz azureuser@new-ip:~/
```

### 3. VS Code Remote SSH Stability
* **Issue:** VS Code failed to connect to the VM, throwing "Proxy connection timed out" errors.
* **Fix:** Manually cleared corrupted server files on the VM (`rm -rf ~/.vscode-server`) and pre-installed dependencies (`wget`, `curl`, `tar`) to ensure a clean handshake.

---

## 🔮 Future Roadmap

* [ ] **Automated Backups:** Create a cron job to backup the `n8n-data` volume to Azure Blob Storage daily.
* [ ] **Scaling:** Implement Azure Container Instances (ACI) if workflow load exceeds VM burstable credits.
* [ ] **Advanced AI Agents:** Develop complex agentic workflows using the stable Gemini integration (e.g., auto-reply bots, research agents).

---

### 👤 Author

**Naveed Shaikh**
* Student | Developer | AI Enthusiast
* First Year BCA @ Amity University Online
