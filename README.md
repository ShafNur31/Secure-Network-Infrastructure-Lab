# Secure-Network-Infrastructure-Lab

Proxmox Network Security Lab
> A self-contained enterprise-grade network security environment built on a Proxmox hypervisor, featuring firewall segmentation, encrypted remote access, hardened servers, and centralised SIEM monitoring.
---

Overview
This lab simulates a real-world IT infrastructure by deploying isolated network segments behind a pfSense firewall, provisioning Linux and Windows servers, establishing a WireGuard VPN tunnel for secure remote management, and monitoring the entire environment with a Wazuh SIEM. Every component is intentionally exposed to a hostile WAN to generate real traffic for analysis.
---

Lab Infrastructure
VM ID	Hostname	IP Address	Role
501	firewall-router	10.0.0.1	pfSense Firewall / Router
502	management	10.0.0.100	Management VM
503	container	10.0.50.3	LXC Container (LAN2)
504	check	10.10.30.10 (WAN-side)	WireGuard Remote Client
505	ubuntu-server	10.0.0.10	Ubuntu Web & File Server
506	windows-FS	10.0.0.20	Windows File Server
507	wazuh-siem	10.0.0.30	Wazuh SIEM Manager
---
Network Configuration (pfSense)

Interface Layout
Interface	Bridge	Subnet	Description
WAN	vmbr0	10.10.30.0/24	Hostile upstream
LAN 1	vmbr1	10.0.0.0/24	Management & SIEM net
LAN 2	vmbr1	10.0.50.0/24	Secondary workload net
VPN	wg0	192.168.100.0/24	WireGuard tunnel
Key Configurations
NAT configured to allow internal VMs to reach the internet while remaining hidden from WAN
DHCP enabled on both LAN1 (`10.0.0.100–200`) and LAN2 (`10.0.50.2–101`) for automated IP assignment
Firewall rules created on LAN2 to explicitly permit outbound traffic (pfSense blocks all by default on new interfaces)
ZFS selected as the filesystem for data integrity and snapshot support
GPT partitioning used for modern disk compatibility
---




WireGuard VPN
Deployed for secure remote access into the internal network from the hostile WAN-side VM (504), simulating an admin connecting from outside the perimeter.



Tunnel Details
Parameter	Value
Listen Port	51820
pfSense Tunnel	192.168.100.1/24
Client Tunnel	192.168.100.3/32
Allowed IPs	192.168.100.2/32, 10.0.0.0/24, 10.0.50.0/24


Client Setup (VM504 - Ubuntu)
```bash
# Install WireGuard
sudo apt update && sudo apt install wireguard -y

# Generate key pair
wg genkey | tee privatekey | wg pubkey > publickey

# Client config
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address    = 192.168.100.3/32
DNS        = 10.0.0.1

[Peer]
PublicKey  = <PFSENSE_PUBLIC_KEY>
Endpoint   = 10.10.30.10:51820
AllowedIPs = 10.0.0.0/24, 10.0.50.0/24, 192.168.100.1/32

# Bring up tunnel
sudo wg-quick up wg0
```
Verification
```bash
ping 192.168.100.1   # Firewall VPN address
ping 10.0.0.1        # Firewall LAN1 address (through tunnel)
ping 10.0.0.20       # Windows File Server (through tunnel)
```
---
Server Provisioning
Ubuntu Server (VM505 - 10.0.0.10)
Deployed on LAN1 with a static IP, running NGINX as a web server and Samba for cross-platform file sharing.
```bash
# Core services installed
sudo apt install nginx -y          # Web server
sudo apt install samba -y          # SMB file sharing
sudo apt install openssh-server -y # Remote access
sudo apt install lynx -y           # CLI browser for local verification
```
Custom NGINX landing page created at `/var/www/html/index.html` to confirm connectivity from within the isolated LAN.
Samba share exposed and mounted from the Windows File Server to validate cross-OS file access:
```bash
sudo mkdir -p /mnt/win_share
sudo mount -t cifs -o username=Administrator //10.0.0.20/internshared /mnt/win_share
```
---
Windows Server 2022 (VM506 - 10.0.0.20)
Provisioned with Desktop Experience on a q35 machine type with TPM and EFI disk for Secure Boot support.
Hardware spec:
CPU: 2 cores (x86-64-v2-AES)
RAM: 4 GiB
Disk: 32 GB
NIC: e1000 bound to vmbr1 (LAN1)
File share configured:
Share path: `C:\internshared`
Share name: `internshared` (via Advanced Sharing)
NTFS permissions: Administrator → Full Control
Windows Firewall: File and Printer Sharing enabled on Private profile
Verified via `net share` command
---
Server Hardening
Windows Server — SMBv1 Disabled
SMBv1 is a legacy protocol associated with critical exploits (e.g. EternalBlue / WannaCry) and was disabled via Group Policy:
```
GPMC.msc
└── Computer Configuration
    └── Preferences > Windows Settings > Registry
        └── HKLM\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters
            └── SMB1 = REG_DWORD : 0
```
Ubuntu Server — Hardening Steps
> *(Screenshots and additional steps to be inserted here)*
---
SIEM Deployment — Wazuh (VM507 - 10.0.0.30)
Wazuh was deployed as an all-in-one installation on a dedicated Ubuntu VM and configured to collect logs from both the Windows and Ubuntu servers.
Wazuh Manager Install
```bash
# Import GPG key
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && \
  chmod 644 /usr/share/keyrings/wazuh.gpg

# Run all-in-one installer
curl -sO https://packages.wazuh.com/4.x/wazuh-install.sh
sudo bash wazuh-install.sh -a
```
Agent Registration
Windows File Server (VM506)
Registered via the Wazuh management CLI on the SIEM VM:
```bash
sudo /var/ossec/bin/manage_agents
# A → Add agent → Name: Intern-Windows-FS → IP: 10.0.0.20 → Confirm → Extract key
```
Ubuntu Server (VM505)
```bash
# Add Wazuh repo
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee -a /etc/apt/sources.list.d/wazuh.list

# Install agent pointing to SIEM manager
sudo apt-get update
sudo WAZUH_MANAGER="10.0.0.30" apt-get install wazuh-agent -y

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```
Both agents report back to the Wazuh dashboard, with failed login attempts and system events captured in real time.
---
Technologies Used
Category	Technology
Hypervisor	Proxmox VE
Firewall / Router	pfSense CE 2.8.1
VPN	WireGuard
OS (Linux)	Ubuntu Server 22.04 LTS
OS (Windows)	Windows Server 2022 Standard Eval
Web Server	NGINX
File Sharing	Samba (SMB), Windows File Sharing
SIEM	Wazuh 4.x
Containerisation	Proxmox LXC
---
Key Skills Demonstrated
Network segmentation — multi-LAN architecture with firewall rules, NAT, and DHCP across isolated subnets
Perimeter security — pfSense firewall policy design separating hostile WAN from internal environments
Encrypted remote access — WireGuard VPN tunnel configuration and end-to-end handshake verification
Server administration — Linux and Windows Server provisioning, service deployment, and cross-platform file sharing
Security hardening — SMBv1 disabled, firewall scoping, and principle of least privilege applied to shares
SIEM monitoring — Wazuh all-in-one deployment with multi-agent log collection and real-time alerting
