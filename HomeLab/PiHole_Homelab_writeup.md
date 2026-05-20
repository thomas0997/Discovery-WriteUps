# Pi-hole Setup Documentation
## Homelab Project: Ad Blocking & DNS Integration

---

## Project Goal
Install and configure Pi-hole on the existing Nextcloud homelab server (Asus Vivobook S15 S530U) to block ads, trackers, and malicious domains network-wide. Integrate Pi-hole as the primary DNS server for all devices on the home network.

---

## Hardware & Environment
- **Server:** Asus Vivobook S15 S530U (same machine as Nextcloud)
- **OS:** Ubuntu Server 22.04 LTS
- **Server IP:** 192.168.1.100
- **Router:** PLDT Fibr EchoLife HG8145V5
- **Location:** Philippines (40°C ambient temperature)
- **Power Setup:** Running on AC power with battery connector disconnected from motherboard for heat and safety management

---

## Why Pi-hole on the Same Server?

Rather than using a separate device (e.g. Raspberry Pi), Pi-hole was installed on the existing Nextcloud Vivobook server for the following reasons:

- **Resource efficiency** — Pi-hole uses only ~100MB RAM; Vivobook has 8GB
- **Cost savings** — No additional hardware needed
- **Simplicity** — One server to manage instead of two
- **Practical** — Server is already running 24/7

**Resource allocation:**
- Ubuntu Server: ~0.5GB RAM
- Nextcloud: ~2-3GB RAM
- Pi-hole: ~100MB RAM
- Total: Well within 8GB capacity ✅

---

## Installation Method: Docker

Pi-hole was installed using Docker for the following reasons:
- Easier to manage and update
- Isolated from the rest of the system
- Easy to restart, remove, or reconfigure
- Industry standard for homelab deployments

---

## Installation Steps

### Step 1: Install Docker
```bash
sudo apt install docker.io -y
```

### Step 2: Start and Enable Docker
```bash
sudo systemctl start docker
sudo systemctl enable docker
```
Docker is now set to start automatically on every boot.

### Step 3: Verify Docker Installation
```bash
sudo docker --version
```
Expected output: `Docker version 24.x.x` or similar.

### Step 4: Create Pi-hole Config Directories
```bash
sudo mkdir -p /opt/pihole/etc-pihole /opt/pihole/etc-dnsmasq
```
These directories store Pi-hole's configuration and DNS settings persistently.

### Step 5: Deploy Pi-hole Container
```bash
sudo docker run -d \
  --name pihole \
  --restart unless-stopped \
  -p 53:53/tcp -p 53:53/udp \
  -p 8080:80 \
  -e TZ="Asia/Manila" \
  -e WEBPASSWORD="yourpassword" \
  -v /opt/pihole/etc-pihole:/etc/pihole \
  -v /opt/pihole/etc-dnsmasq:/etc/dnsmasq.d \
  pihole/pihole:latest
```

**Key flags explained:**
- `--restart unless-stopped` — Auto-restarts Pi-hole on every reboot
- `-p 53:53` — DNS port (TCP and UDP)
- `-p 8080:80` — Web admin dashboard port
- `TZ="Asia/Manila"` — Correct timezone for Philippines
- `WEBPASSWORD` — Admin dashboard password

### Step 6: Verify Pi-hole is Running
```bash
sudo docker ps
```
Pi-hole container should show status: **Up** ✅

---

## Accessing Pi-hole Dashboard

Pi-hole admin dashboard is accessible at:
```
http://192.168.1.100:8080/admin
```

Log in with the password set during deployment.

---

## Issues & Troubleshooting

### Issue 1: Wrong Password on Login
**Problem:** Pi-hole dashboard rejected the correct password set during Docker deployment.

**Root Cause:** Password sometimes doesn't set correctly on first Docker run.

**Resolution:** Reset password manually using Docker exec:
```bash
sudo docker exec -it pihole bash
pihole setpassword yournewpassword
exit
```

---

## DNS Integration: Device Configuration

Instead of configuring the PLDT router's DNS settings (which can be restrictive), DNS was configured **per device** — the same method used previously with a VM-based Pi-hole setup.

### iPhone/iPad
1. Settings → WiFi
2. Tap your network name
3. Configure DNS → Manual
4. Add DNS server: `192.168.1.100`

### Mac
1. System Preferences → Network
2. Select active connection → Advanced
3. DNS tab → Add `192.168.1.100`

### Windows
1. Settings → Network & Internet
2. Change adapter options
3. Right-click → Properties → IPv4
4. Set Preferred DNS: `192.168.1.100`
5. Set Alternate DNS: `8.8.8.8` (backup)

---

## Auto-Restart Behavior

Pi-hole automatically restarts on every Vivobook reboot due to the `--restart unless-stopped` Docker flag.

**Startup sequence after Vivobook powers on:**
1. Ubuntu Server boots (~1 minute)
2. Docker starts automatically
3. Pi-hole container starts automatically
4. Nextcloud (Apache + MySQL) starts automatically
5. Everything live — no manual intervention needed ✅

---

## Hardware Safety Notes

### Battery Management
Due to Philippines' 40°C ambient temperature, the following steps were taken to prevent overheating:

- **Battery connector disconnected from motherboard** — Laptop runs purely on AC power
- Battery physically remains inside laptop but is completely inactive
- No charging = no heat generation from battery
- Equivalent to running as a desktop computer

**Result:** Safer long-term operation in high-temperature environment ✅

### Recommended Future Additions
- **Laptop stand** — Improves airflow underneath (~₱200-500)
- **Small desk fan** — Reduces ambient temperature around server (~₱300-500)
- **UPS (Uninterruptible Power Supply)** — Protects against Philippines brownouts (~₱1,500-3,000 at PC Express/Lazada)
  - Recommended brands: APC, CyberPower, Eaton
  - Without UPS: sudden power loss could corrupt Nextcloud data

---

## Current Server Stack

| Service | Status | Port |
|---|---|---|
| Ubuntu Server 22.04 | Running ✅ | — |
| Apache2 | Running ✅ | 80 |
| MySQL | Running ✅ | 3306 |
| Nextcloud | Running ✅ | 80 |
| Docker | Running ✅ | — |
| Pi-hole | Running ✅ | 53, 8080 |

---

## Next Steps
1. Configure blocklists in Pi-hole dashboard (add more ad/tracker lists)
2. Monitor Pi-hole statistics (queries blocked, top blocked domains)
3. Set up UPS for brownout protection
4. Configure HTTPS for Nextcloud (fixes Chrome/Firefox browser issues)
5. Set up remote access (domain + reverse proxy)
6. Install Jellyfin media server

---

## Project Status
**Pi-hole Phase:** Complete ✅
**Overall Homelab Status:** Nextcloud + Pi-hole running locally on Vivobook
**Next Phase:** HTTPS configuration and remote access setup
