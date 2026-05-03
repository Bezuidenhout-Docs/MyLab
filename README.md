![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=for-the-badge&logo=proxmox&logoColor=white)
![Debian](https://img.shields.io/badge/Debian-A81D33?style=for-the-badge&logo=debian&logoColor=white)
![Alpine Linux](https://img.shields.io/badge/Alpine_Linux-0D597F?style=for-the-badge&logo=alpine-linux&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![WireGuard](https://img.shields.io/badge/WireGuard-88171A?style=for-the-badge&logo=wireguard&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-242424?style=for-the-badge&logo=tailscale&logoColor=white)
![Cloudflare](https://img.shields.io/badge/Cloudflare-F38020?style=for-the-badge&logo=cloudflare&logoColor=white)

🖥️ Proxmox HomeLab Documentation
Version: 1.0
Last Updated: 2026-05-03
Hostname: pve
Purpose: Self-hosted services, automation, and network utilities lab

1. System Overview
Hardware Specifications
Component	Specification
Platform	Generic x86_64 server
CPU	Intel 3rd Gen Core (Ivy Bridge)
RAM	16GB DDR3
Storage	500GB SATA SSD
NIC	Gigabit Ethernet (100Mbps operational limit)
Boot Mode	UEFI
Software Environment
Component	Version
Hypervisor	Proxmox Virtual Environment 8.x
Kernel	6.17.4-1-pve
OS Base	Debian GNU/Linux 12 (Bookworm)
Storage Backend	LVM-thin + LVM standard
2. Network Architecture
text
                       ┌─────────────────────────────┐
                       │          Internet            │
                       └──────────────┬──────────────┘
                                      │
                                      ▼
                       ┌─────────────────────────────┐
                       │      ISP Router / Gateway    │
                       │        192.168.222.1         │
                       │  Port forwards: 80/443, 51820│
                       └──────────────┬──────────────┘
                                      │
                                      ▼
                       ┌─────────────────────────────┐
                       │   Bridge: vmbr0              │
                       │   Host IP: 192.168.222.100   │
                       │   Subnet: 255.255.255.0      │
                       └──────────────┬──────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
    │  Reverse Proxy  │     │  VPN Endpoint   │     │  DNS Resolver   │
    │    (NPM)        │     │  (WireGuard)    │     │  (AdGuard)      │
    │   101           │     │   112           │     │   109           │
    └─────────────────┘     └─────────────────┘     └─────────────────┘
External Access Methods
Method	Target	Purpose
Port Forwarding (80/443)	NPM (101)	Public web service access
Port Forwarding (51820)	WireGuard (112)	Full VPN tunnel
Tailscale	Mesh network	Zero-trust overlay
Cloudflare Tunnel	HTTP/HTTPS	CDN + DDoS protection + SSL
3. LXC Inventory
All containers are unprivileged with nesting enabled unless noted.

VMID	Hostname	IP Address	vCPU	RAM	Disk	OS	Tags
101	nginxproxymanager	192.168.222.101	2	2GB	8GB	Debian	proxy, reverse-proxy
102	myspeed	192.168.222.102	1	1GB	4GB	Debian	tracking, speedtest
103	upsnap	192.168.222.103	1	512MB	2GB	Debian	network, wol
104	docker	192.168.222.104	2	2GB	4GB	Debian	docker, container-host
105	heimdall-dashboard	192.168.222.105	1	512MB	2GB	Debian	dashboard, launcher
106	n8n	192.168.222.106	2	2GB	10GB	Debian	automation, workflow
107	smokeping	192.168.222.107	1	512MB	2GB	Debian	network, monitoring
108	vaultwarden	192.168.222.108	4	6GB	20GB	Debian	password-manager, security
109	adguard	192.168.222.109	1	512MB	2GB	Debian	adblock, dns
110	glance	192.168.222.110	1	512MB	2GB	Debian	dashboard, rss
111	memos	192.168.222.111	1	1GB	3GB	Debian	notes, snippets
112	wireguard	192.168.222.112	1	512MB	4GB	Debian	vpn, network
114	alpine-it-tools	192.168.222.114	1	256MB	1GB	Alpine	development, utilities
Note: Container 112 (WireGuard) includes /dev/net/tun passthrough for VPN functionality.

4. Resource Allocation Summary
Resource	Allocated	Pool Total	Usage
vCPUs	18	N/A (overcommit)	Light load
RAM	~18GB	16GB	Overcommitted
Disk (LVM-thin)	~38.5GB	338GB	11.4%
VG Free Space	16GB	—	Available
Optimization Notes
Container vaultwarden (108) uses 6GB RAM — can be reduced to 2GB for typical usage

Total allocated RAM exceeds physical RAM — monitor with htop or Proxmox dashboard

Container heimdall-dashboard (105) disk at 93% usage — expand or clean

5. Storage Layout
text
SSD (500GB)
├── Partition 1 (1MB) - BIOS boot
├── Partition 2 (1GB) - EFI (vfat)
└── Partition 3 (~464GB) - LVM Physical Volume
    └── Volume Group "pve"
        ├── swap (7.7GB)
        ├── root (96GB) - ext4, mounted at /
        └── data (338GB) - LVM-thin pool
            ├── Thin volumes for all containers (see inventory above)
            └── Free space: 16GB remaining
Storage Commands
bash
# Check disk usage
lvs

# Expand container disk
lvextend -L +2G /dev/pve/vm-105-disk-0

# Resize filesystem inside container
# (run inside container) resize2fs /dev/sda1
6. Security Configuration
Current State
Aspect	Status
Host firewall	Disabled (all ACCEPT)
Container firewalls	Not configured
VLAN isolation	None (single flat subnet)
Fail2ban	Not installed
SSH hardening	Not applied
External Exposure
Service	Exposed To Internet
Nginx Proxy Manager (101)	Yes (ports 80/443)
WireGuard (112)	Yes (port 51820)
All other containers	No (local only)
Security Recommendations
Enable Proxmox host firewall via datacenter settings

Implement VLAN segmentation (management, services, untrusted)

Install fail2ban on containers exposed to internet

Disable password authentication for SSH (use key-based only)

Regular security updates: apt update && apt upgrade

7. Backup Strategy
Current Backups
Method: Manual (vzdump ad-hoc)

Schedule: None

Retention: Not defined

Offsite: Not configured

⚠️ Risk: No automated backups. Full recovery requires manual rebuild.

Implemented Backup Script
Save as /root/backup.sh and make executable:

bash
#!/bin/bash
BACKUP_DIR="/var/backups/proxmox"
DATE=$(date +%Y%m%d-%H%M%S)
mkdir -p $BACKUP_DIR

# Backup all containers
for vmid in 101 102 103 104 105 106 107 108 109 110 111 112 114; do
    vzdump $vmid --mode snapshot --compress zstd --dumpdir $BACKUP_DIR
done

# Backup host configuration
tar czf $BACKUP_DIR/host-configs-$DATE.tar.gz /etc/pve/ /etc/network/interfaces

# Remove backups older than 30 days
find $BACKUP_DIR -name "*.vma.zst" -mtime +30 -delete
Automate with cron:

bash
# Weekly backup every Sunday at 2 AM
0 2 * * 0 /root/backup.sh
8. Disaster Recovery Procedure
Prerequisites
Proxmox installation ISO

Backup files (from script above)

Network configuration notes

Recovery Steps
Install Proxmox on new storage device

Configure host network (/etc/network/interfaces)

Restore containers:

bash
cd /var/backups/proxmox
for backup in *.vma.zst; do
    vzdump restore $backup $(basename $backup | cut -d- -f3)
done
Restore host configurations:

bash
tar xzf host-configs-*.tar.gz -C /
Verify networking and start containers:

bash
pct start <vmid> --all
9. Maintenance Procedures
Weekly Tasks
bash
# Update container templates
pveam update

# Check disk usage
lvs
df -h

# Review logs for errors
journalctl -xe | grep -i error
Monthly Tasks
bash
# Full system update
apt update && apt upgrade -y

# Reboot if kernel updated
reboot

# Verify backup integrity
find /var/backups/proxmox -name "*.vma.zst" -exec tar tf {} \; > /dev/null
Performance Monitoring
bash
# Quick status
pct list
pvesh get /nodes/localhost/status

# Resource usage
htop
iotop
10. Service Dependency Diagram
text
Internet
    │
    ▼
Gateway (.1)
    │
    ├──► Port 80/443 ──► NPM (101) ──┬──► Heimdall (105)
    │                                ├──► Glance (110)
    │                                ├──► Memos (111)
    │                                └──► MySpeed (102)
    │
    ├──► Port 51820 ──► WireGuard (112) ──► Full LAN Access
    │
    └──► Tailscale/Cloudflare ──► Overlay Network
  
Internal Only:
    ├──► AdGuard (109) ◄── DNS for all containers
    ├──► n8n (106)
    ├──► Docker (104)
    ├──► SmokePing (107)
    ├──► UpSnap (103)
    ├──► Vaultwarden (108)
    └──► Alpine IT Tools (114)
11. Quick Command Reference
Task	Command
List containers	pct list
Enter container console	pct enter <vmid>
Start/shutdown container	pct start/stop <vmid>
Create manual backup	vzdump <vmid> --compress zstd
View storage usage	lvs && vgs
Check container logs	pct log <vmid>
Network status	ip a && bridge link show
Live resource monitor	watch -n 2 'pct list && echo --- && vgs'
12. Known Limitations & Future Improvements
Current Limitations
Network bottleneck: 100Mbps Ethernet limits throughput

RAM overcommit: ~18GB allocated to 16GB physical

No UPS protection: Risk of filesystem corruption

Single point of failure: No redundancy

Planned Improvements
Implement automated backups

Reduce Vaultwarden RAM allocation from 6GB → 2GB

Expand Heimdall disk from 2GB → 4GB

Enable host firewall and VLANs

Add UPS with auto-shutdown via NUT

Migrate to hardware with Gigabit networking

Appendix: Deployment Notes
Initial Setup Commands
All containers deployed using Community Scripts for Proxmox:

bash
# Source: https://community-scripts.org
# Scripts used for each container type
Environment Variables
Setting	Value
Timezone	Africa/Johannesburg
Nameserver	8.8.8.8 (in container 101)
Gateway	192.168.222.1
DNS for containers	AdGuard (109) for ad-blocking
Documentation generated: 2026-05-03
Maintained by: Homelab enthusiast
License: MIT (feel free to use and adapt)
