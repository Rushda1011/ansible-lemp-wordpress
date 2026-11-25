# Automated Production LEMP Stack & WordPress Infrastructure with Ansible

![Ansible](https://img.shields.io/badge/Ansible-2.15+-EE0000?style=for-the-badge&logo=ansible&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04%20|%2024.04%20LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-1.28+-009639?style=for-the-badge&logo=nginx&logoColor=white)
![MariaDB](https://img.shields.io/badge/MariaDB-10.6+-003545?style=for-the-badge&logo=mariadb&logoColor=white)
![PHP](https://img.shields.io/badge/PHP-8.2%20|%208.5-777BB4?style=for-the-badge&logo=php&logoColor=white)
![WordPress](https://img.shields.io/badge/WordPress-6.0+-21759B?style=for-the-badge&logo=wordpress&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazon-aws&logoColor=white)

An idempotent, production-ready **Infrastructure as Code (IaC)** Ansible automation project that provisions and hardens a complete **LEMP** web hosting environment on an AWS EC2 instance. 

The playbook automates user isolation with custom PHP-FPM pools, hardened SFTP access, database provisioning with least privilege access, unattended phpMyAdmin installation, WP-CLI automated WordPress deployment, and Let's Encrypt SSL configuration.

---

## 🏗️ Architecture Overview

```text
                               +------------------------------------------------+
                               |              AWS EC2 Ubuntu Host              |
                               |                                                |
[Browser: Port 80/443]  -----> |  Nginx Web Server                              |
                               |    ├── studentlab2026.shop (WordPress Proxy)   |
[Browser: Port 8080]    -----> |    └── Port 8080 (phpMyAdmin Web UI)           |
                               |           │                                    |
                               |           ▼ FastCGI Unix Socket                |
                               |  Isolated PHP-FPM Pool (/run/php/webuser.sock) |
                               |  Running as: 'webuser' (Non-root user)         |
                               |           │                                    |
                               |           ▼ Local Socket                       |
                               |  MariaDB Database Engine (wordpress_db)        |
                               |                                                |
[SFTP Client: Port 22]  -----> |  sshd (Chroot/Match User 'webuser')            |
                               |  Web Root: /home/webuser/studentlab/public     |
                               +------------------------------------------------+
```

---

## ✨ Key Features & Engineering Highlights

* **Full LEMP Stack Automation:** Seamless installation and lifecycle management for Nginx, MariaDB, and PHP with all required WordPress extensions.
* **Security & User Segregation (Least Privilege):**
  * Website files are owned by an unprivileged system user (`webuser`).
  * Dedicated PHP-FPM process pool executes PHP scripts under `webuser:www-data`, preventing web compromise from elevating to root.
  * Dedicated MariaDB database user (`wp_user`) with scoped privileges strictly limited to `wordpress_db.*`. Anonymous users and test databases are completely purged.
* **Hardened SFTP Access:** Configured OpenSSH `Match User` directive enabling password-authenticated SFTP while explicitly disabling TCP and X11 tunneling.
* **Automated WP-CLI Setup:** Bypasses manual web-based setup wizards by running unattended headless WordPress installations (`wp core install`) and configuring SEO-friendly permalinks directly via the command line.
* **Database Management Web Interface:** Provisions secured phpMyAdmin on a dedicated administrative port (`8080`).
* **SSL & Security Hardening:** Automated certificate generation and renewal cron jobs using Certbot (Let's Encrypt), coupled with local UFW firewall policies.
* **DRY Architecture:** Modular design separating play logic (`playbook.yml`), dynamic Jinja2 templates (`templates/`), and sensitive parameters (`vars.yml`).

---

## 📁 Repository Structure

```text
├── ansible.cfg                 # Global Ansible settings, privilege escalation & defaults
├── inventory.ini.example       # Template inventory file (local or remote SSH hosts)
├── vars.yml.example            # Centralized parameters and credentials template
├── playbook.yml                # Main Ansible orchestrator (8 distinct stages)
├── templates/                  # Modular Jinja2 configuration templates
│   ├── nginx-wordpress.conf.j2 # Nginx Virtual Host for WordPress (ports 80 & 443)
│   ├── phpmyadmin-nginx.conf.j2# Dedicated Nginx Virtual Host for phpMyAdmin (port 8080)
│   ├── php-fpm-pool.conf.j2    # Custom PHP-FPM process pool running under website user
│   └── wp-config.php.j2        # Dynamic wp-config.php with salts & database parameters
├── .gitignore                  # Prevents accidental commit of keys and secrets
└── README.md                   # Project documentation
```

---

## 🚀 Quickstart & Deployment Guide

### Prerequisites
1. An **AWS EC2 Ubuntu instance** (Ubuntu 22.04 LTS or 24.04 LTS recommended).
2. **Inbound Security Group Rules:**
   * `Port 22` (SSH / SFTP)
   * `Port 80` (HTTP)
   * `Port 443` (HTTPS)
   * `Port 8080` (phpMyAdmin)
3. **Ansible 2.15+** installed on your controller machine or target server.

---

### Step 1: Clone the Repository
```bash
git clone https://github.com/Rushda1011/ansible-lemp-wordpress.git
cd ansible-lemp-wordpress
```

### Step 2: Configure Variables & Inventory
Copy the example files and update them with your environment settings:

```bash
cp vars.yml.example vars.yml
cp inventory.ini.example inventory.ini
```

Edit `vars.yml` with your domain, desired passwords, and public IP:
```yaml
domain_name: "example.com"
server_ip: "YOUR_EC2_PUBLIC_IP"
site_user: "webuser"
site_user_password: "YourSecurePassword!"
db_password: "YourDbPassword!"
wp_admin_password: "YourAdminPassword!"
```

### Step 3: Run the Playbook

#### Option A: Running from a Remote Controller Machine (via SSH)
Ensure your private key (`.pem`) has correct permissions (`chmod 400 key.pem`), update `inventory.ini`, and run:
```bash
ansible-playbook -i inventory.ini playbook.yml
```

#### Option B: Running Directly on the Target EC2 Server (Local Execution)
If executing inside the server itself:
```bash
ansible-playbook -i "localhost," -e "ansible_connection=local" playbook.yml
```

---

## 📊 Deployment Verification

Once execution completes with `failed=0`, verify the endpoints:

| Service | Access Endpoint | Authentication / Details |
| :--- | :--- | :--- |
| **WordPress Public Site** | `http://<SERVER_IP>` | Public Frontend |
| **WordPress Admin Dashboard** | `http://<SERVER_IP>/wp-admin` | User: `admin` |
| **phpMyAdmin Console** | `http://<SERVER_IP>:8080` | User: `wp_user` or `root` |
| **SFTP File Access** | `<SERVER_IP>:22` | User: `webuser` (Web root: `/home/webuser/studentlab/public`) |

---

## 🔒 Security Best Practices Implemented
- **No Secret Leakage:** `.gitignore` blocks all private `.pem` keys, local configs, and real variable files.
- **Process Isolation:** The web server runs as `www-data`, but PHP execution is restricted to unprivileged `webuser`.
- **Database Hardening:** Remote root login is restricted to local socket connections; database permissions follow strict least privilege.
- **Automated SSL:** Automatic 90-day renewal handled via cron.

---

## 👤 Author
Developed as part of the Cloud Infrastructure & Automation Internship project.
* **GitHub:** [@Rushda1011](https://github.com/Rushda1011)
* **Project Repository:** [ansible-lemp-wordpress](https://github.com/Rushda1011/ansible-lemp-wordpress)
