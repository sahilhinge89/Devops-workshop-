# Ubuntu VM Setup with Jenkins on VirtualBox

This project demonstrates how to create an Ubuntu Virtual Machine using VirtualBox and install Jenkins for CI/CD development.

---

# Prerequisites

- Oracle VirtualBox
- Ubuntu Server 22.04 LTS ISO
- Internet Connection
- Minimum 8 GB RAM on Host Machine

---

# VM Configuration

| Setting | Value |
|---------|-------|
| Name | Ubuntu-Server |
| OS | Ubuntu 22.04 LTS |
| RAM | 8 GB |
| CPU | 4 Cores |
| Disk | 50 GB |
| Storage Type | Dynamically Allocated |
| Network | NAT (Port Forwarding) / Bridged Adapter |

---

# Step 1: Create Virtual Machine

1. Open VirtualBox.
2. Click **New**.
3. Enter VM Name.
4. Select Linux → Ubuntu (64-bit).
5. Allocate Memory.
6. Allocate CPUs.
7. Create Virtual Hard Disk.
8. Choose VDI.
9. Select Dynamically Allocated.
10. Allocate 50 GB Storage.

---

# Step 2: Attach Ubuntu ISO

Settings

```
Storage
→ Empty
→ Choose Ubuntu ISO
```

Start the VM.

---

# Step 3: Install Ubuntu Server

Boot the VM using the Ubuntu Server ISO.

---

## 3.1 Select Language

Choose your preferred language.

Recommended:

```
English
```

Press **Enter**.

---

## 3.2 Installer Update

When prompted to update the installer, choose:

```
Continue without updating
```

*(Optional: You can update later after installation.)*

---

## 3.3 Keyboard Configuration

Choose the following:

```
Layout : English (US)
Variant : English (US)
```

Select **Done**.

---

## 3.4 Network Configuration

The installer automatically detects the network.

If using NAT:

```
DHCP
```

Leave the default settings and select **Done**.

---

## 3.5 Configure Proxy

Leave the proxy field empty.

```
Proxy Address:

(blank)
```

Select **Done**.

---

## 3.6 Configure Ubuntu Archive Mirror

Keep the default mirror.

Example:

```
http://archive.ubuntu.com/ubuntu
```

Select **Done**.

---

## 3.7 Guided Storage Configuration

Choose:

```
Use an entire disk
```

Select the Virtual Disk.

Example:

```
VBOX HARDDISK
```

Keep these options enabled:

```
✓ Set up this disk as an LVM group
```

Leave encryption disabled.

Choose:

```
Done
```

Confirm by selecting:

```
Continue
```

---

## 3.8 Profile Setup

Fill in the required details.

Example:

| Field | Example |
|--------|---------|
| Your Name | Sahil Hinge |
| Server Name | jenkins-server |
| Username | sahil |
| Password | ******** |
| Confirm Password | ******** |

Select **Done**.

---

## 3.9 SSH Setup

Select:

```
✓ Install OpenSSH Server
```

Do **not** import SSH keys from GitHub or Launchpad.

Select **Done**.

---

## 3.10 Featured Server Snaps

No additional software is required.

Leave everything unchecked.

Choose:

```
Done
```

---

## 3.11 Installation Complete

Wait for Ubuntu to finish installing.

When prompted:

```
Reboot Now
```

Remove the Ubuntu ISO when VirtualBox asks.

Press **Enter** to boot into Ubuntu.

---

## 3.12 Login

Login using the credentials created during installation.

Example:

```
Username : sahil
Password : ********
```

Verify the installation:

```bash
lsb_release -a
```

Example output:

```
Distributor ID: Ubuntu
Description: Ubuntu 22.04 LTS
```
---

# Step 4: Install OpenSSH Server

During installation select

```
✓ Install OpenSSH Server
```

Do not import GitHub or Launchpad keys.

---

# Step 5: Update Ubuntu

```bash
sudo apt update
sudo apt upgrade -y
```

---

# Step 6: Install Basic Packages

```bash
sudo apt install git curl wget unzip vim net-tools tree htop -y
```

---

# Step 7: Install UFW

```bash
sudo apt install ufw -y
```

Allow SSH

```bash
sudo ufw allow OpenSSH
```

Enable Firewall

```bash
sudo ufw enable
```

Verify

```bash
sudo ufw status
```

---

# Step 8: Install Java

```bash
sudo apt install fontconfig openjdk-21-jdk -y
```

Verify

```bash
java -version
```

---

# Step 9: Install Jenkins

Import Jenkins Key

```bash
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
/usr/share/keyrings/jenkins-keyring.asc > /dev/null
```

Add Repository

```bash
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
```

Update Packages

```bash
sudo apt update
```

Install Jenkins

```bash
sudo apt install jenkins -y
```

---

# Step 10: Start Jenkins

```bash
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

Check Status

```bash
sudo systemctl status jenkins
```

---

# Step 11: Allow Jenkins Port

```bash
sudo ufw allow 8080/tcp
sudo ufw reload
```

---

# Step 12: Configure VirtualBox NAT Port Forwarding

VirtualBox

```
Settings
→ Network
→ NAT
→ Advanced
→ Port Forwarding
```

Add Rule

| Name | Protocol | Host Port | Guest Port |
|------|----------|----------|-----------|
| Jenkins | TCP | 8080 | 8080 |

---

# Step 13: Access Jenkins

Open Browser

```
http://localhost:8080
```

If using Bridged Adapter

```
http://<VM-IP>:8080
```

---

# Step 14: Unlock Jenkins

Retrieve Initial Password

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Paste the password into the Jenkins setup page.

---

# Step 15: Install Suggested Plugins

Click

```
Install Suggested Plugins
```

Create your first administrator account.

---

# Verify Installation

Check Java

```bash
java -version
```

Check Jenkins

```bash
sudo systemctl status jenkins
```

Check Port

```bash
sudo ss -tulpn | grep 8080
```

Open Browser

```
http://localhost:8080
```

---

# Project Structure

```
VirtualBox
│
├── Ubuntu Server 22.04
│
├── OpenSSH
│
├── UFW Firewall
│
├── Java 21
│
└── Jenkins
```

---




# Technologies Used

- Ubuntu Server 22.04
- Oracle VirtualBox
- OpenSSH
- UFW
- OpenJDK 21
- Jenkins
- Git

---

# Author

**Sahil Hinge**

DevOps Engineer | Linux | Docker | Jenkins | Kubernetes | AWS
