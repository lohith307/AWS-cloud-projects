# Secure Windows Virtual Machine Deployment with CMS on AWS

##  Objective

Deploy a **Windows Virtual Machine on AWS** and install a popular **Content Management System (CMS)** like **WordPress** or **Magento**, following **security best practices** to avoid any violations.

---

##  Architecture Components

- **Amazon EC2 (Windows Server)**
- **Amazon RDS (Optional)**
- **Amazon S3 (Optional for media)**
- **AWS Systems Manager**
- **IAM Roles and Policies**
- **CloudWatch Monitoring**

---

##  Architecture Diagram

```
                   +----------------------------+
                   |    AWS Management Console  |
                   +----------------------------+
                              |
                              v
+---------------------+    +-------------------+
|   Amazon RDS (DB)   |<--> |   EC2 Windows VM   |
+---------------------+    +-------------------+
                                  |
                         +----------------+
                         |    IIS + CMS    |
                         | (WordPress/etc) |
                         +----------------+
                                  |
                        Public Access (HTTP/HTTPS)
```

---

##  Step-by-Step Deployment

### 1. Launch a Secure Windows EC2 Instance




- Go to **EC2 > Launch Instance**
- **AMI**: Microsoft Windows Server 2019/2022 Base
- **Instance Type**: t3.medium or higher
- **Key Pair**: Create or select existing `.pem` key
- **Networking**:
  - VPC: Default or custom
  - Subnet: Public
  - Public IP: Enabled
- **Security Group**:
  - Inbound Rules:
    - RDP (3389) – YOUR IP only
    - HTTP (80) – All
    - HTTPS (443) – All
  - Outbound Rules: Allow all
- **Storage**:
  - Minimum 50 GB (more for Magento)
  - Enable encryption

### 2. Harden the EC2 Windows Instance

```powershell
# Change admin password manually after login
# Run Windows Update
sconfig
# Enable Firewall (should be ON by default)
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```

### 3. Install and Configure IIS Web Server

```powershell
# Open PowerShell as Administrator
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Install PHP using WebPI
Start-Process "C:\Program Files\Microsoft\Web Platform Installer\WebPlatformInstaller.exe"
```

Validate setup:

- Open browser in the VM: `http://localhost`
- Add a `phpinfo.php` file:

```php
<?php
phpinfo();
?>
```

Place this in `C:\inetpub\wwwroot` and visit it in browser.

### 4. Set Up Database

#### Option A: Install MySQL Locally

- Download installer: [https://dev.mysql.com/downloads/installer/](https://dev.mysql.com/downloads/installer/)
- Configure root password
- Create CMS-specific database and user

```sql
CREATE DATABASE cms_db;
CREATE USER 'cms_user'@'localhost' IDENTIFIED BY 'StrongPassword!';
GRANT ALL PRIVILEGES ON cms_db.* TO 'cms_user'@'localhost';
FLUSH PRIVILEGES;
```

#### Option B: Use Amazon RDS

- Launch RDS MySQL/MariaDB instance
- Use a security group allowing access only from EC2

### 5. Install CMS (WordPress or Magento)

#### WordPress

```powershell
Invoke-WebRequest -Uri https://wordpress.org/latest.zip -OutFile wordpress.zip
Expand-Archive wordpress.zip -DestinationPath C:\inetpub\wwwroot\wordpress
```

- Configure `wp-config.php`
- Visit `http://<EC2-Public-IP>/wordpress` to start setup

#### Magento (heavier)

```powershell
# Prerequisites
choco install composer
choco install nodejs
# PHP extensions should be manually enabled via php.ini
```

Follow official Magento setup for Windows: [https://developer.adobe.com/commerce/](https://developer.adobe.com/commerce/)

### 6. Secure the Deployment

```powershell
# Download win-acme for SSL
Invoke-WebRequest -Uri https://github.com/win-acme/win-acme/releases/latest/download/win-acme.v2.1.20.1185.x64.trimmed.zip -OutFile win-acme.zip
Expand-Archive win-acme.zip -DestinationPath C:\win-acme
cd C:\win-acme
wacs.exe
```

- Follow prompts to bind a certificate to your IIS site

Disable directory listing:

```powershell
Set-WebConfigurationProperty -Filter "/system.webServer/directoryBrowse" -Name "enabled" -Value "False" -PSPath "IIS:\"
```

### 7. IAM and Access Management

- Create IAM role with:
  - `AmazonSSMManagedInstanceCore`
  - `CloudWatchAgentServerPolicy`
- Attach role to EC2 instance
- Use IAM users with MFA for all access

### 8. Optional: Use AWS Systems Manager

- Enable SSM Agent (preinstalled on most Windows AMIs)
- Connect securely via **Session Manager** instead of RDP

### 9. Monitoring & Logging

```powershell
# Install CloudWatch Agent
Invoke-WebRequest https://s3.amazonaws.com/amazoncloudwatch-agent/windows/amd64/latest/amazon-cloudwatch-agent.msi -OutFile cloudwatch-agent.msi
msiexec /i cloudwatch-agent.msi /qn
```

Configure and start the agent with a config file for metrics/logs.

---

##  Summary

With this setup, you've securely deployed a Windows VM on AWS with a CMS, hardened the environment, and followed AWS security best practices.

---

##  Repository Suggestions

- `docs/`: All configuration and setup screenshots
- `scripts/`: Any setup automation scripts (PowerShell, SSM, etc.)
- `README.md`: This document
- `backups/`: EBS or DB backup details (redacted)

---

## 📎 References

- [WordPress Official](https://wordpress.org)
- [Magento Official](https://magento.com)
- [AWS EC2 Windows Guide](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/)
- [AWS RDS Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/)
- [win-acme for Windows SSL](https://github.com/win-acme/win-acme)

---

