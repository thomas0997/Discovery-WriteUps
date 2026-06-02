# Homelab Phase 2: Pi-hole DNS Ad Blocker

A full documentation of adding Pi-hole to an existing Nextcloud homelab server — covering installation failures, Docker networking, and the final working setup.

---

## Project Goal

Extend the existing Nextcloud homelab server to also run Pi-hole, a DNS-based ad blocker that filters ads and trackers at the network level. Rather than configuring it router-wide, the goal was to point only specific personal devices at Pi-hole, since the PC and homelab are always powered on together.

This approach avoids touching router settings and keeps the change isolated to the devices that need it.

---

## Hardware and Software

| Component | Details |
|---|---|
| Primary Server | Asus Vivobook S15 S530U (2018, Intel Core i5-8250U, 8GB RAM) |
| Storage | WD Blue 1TB 2.5" SATA HDD |
| Operating System | Ubuntu Server 22.04 LTS |
| Existing Stack | Apache 2.4, MySQL, PHP 8.2, Nextcloud (bare metal) |
| Pi-hole Method | Docker container (final working approach) |
| Static IP | 192.168.1.100 (configured via Netplan) |

---

## Project Phases

- [x] Phase 1: Verify Nextcloud auto-starts on boot
- [x] Phase 2: Install Pi-hole and web dashboard
- [x] Phase 3: Set up thermal shutdown at 85°C
- [x] Phase 4: Point PC and Mac DNS to Pi-hole
- [ ] Phase 5: Jellyfin media server (planned)
- [ ] Phase 6: Homelab dashboard GUI (planned)

---

## Pre-Installation: Confirming Nextcloud Auto-start

Before adding Pi-hole, the first step was confirming that Nextcloud would survive a cold boot without manual intervention. The homelab runs on a single extension cord switch, meaning everything powers on together. Nextcloud must be available the moment the server comes up.

```bash
sudo systemctl is-enabled apache2
sudo systemctl is-enabled mysql
```

Both returned `enabled`, confirming that Nextcloud would start automatically on every boot. No changes were needed.

---

## Troubleshooting Log

This project ran into a significant number of issues across two installation attempts before a working solution was found. Each is documented below with its root cause and resolution.

---

### Issue 1: Pi-hole Installer Did Not Install Lighttpd

**Problem:** The standard Pi-hole installation script completed without errors, but navigating to the dashboard URL returned a "page not found" error. Lighttpd, which Pi-hole uses to serve its web dashboard, was not installed despite being selected during the interactive setup.

**Diagnosis:** Running `pihole -v` revealed the problem immediately:

```
Web Version is N/A (Latest: v6.4.2)
```

The web interface component was simply never installed. Only the DNS engine (pihole-FTL) was running.

**Root cause:** An installer bug or conflict with the existing environment caused the web interface component to be silently skipped.

**Resolution:** Switched to Docker, which bundles the web interface correctly as part of the container image.

**Lesson learned:** Always verify a full installation by checking the version output of each component, not just whether the installer finished without errors.

---

### Issue 2: Port 80 Conflict Between Apache and Lighttpd

**Problem:** When lighttpd was manually installed to serve the Pi-hole dashboard, it failed to start. Both Apache (running Nextcloud) and lighttpd defaulted to port 80, and only one can listen on a port at a time.

**Root cause:** Both web servers claim port 80 by default. The existing Apache installation already owned it.

**Resolution:** Configured lighttpd to use port 8080 instead:

```bash
sudo nano /etc/lighttpd/lighttpd.conf
# Change: server.port = 80
# To:     server.port = 8080
sudo systemctl restart lighttpd
```

In the final Docker setup, this conflict is handled cleanly via an environment variable rather than manual config editing.

---

### Issue 3: Pi-hole Dashboard Returning 403 Forbidden

**Problem:** Even after getting lighttpd running on port 8080 and confirming that Pi-hole's admin files existed in `/var/www/html/admin/`, the dashboard returned 403 Forbidden.

**Diagnosis:** The files were present and had correct ownership (`www-data`). The real problem was that lighttpd had no PHP module configured. Pi-hole's dashboard is built in PHP, and lighttpd was refusing to execute `.php` files because it had no way to process them.

**Root cause:** A bare lighttpd installation has no PHP integration out of the box. Setting it up correctly requires installing and enabling PHP-FPM or a similar module, then linking it to lighttpd.

**Resolution:** Rather than configuring PHP manually in lighttpd, this was the point where the manual installation approach was abandoned entirely in favour of Docker. The Docker image pre-configures PHP and the web server correctly inside the container.

**Lesson learned:** A 403 on a file that exists and has correct permissions almost always points to the web server being unable to process the file type, not a permissions issue.

---

### Issue 4: Docker DNS Unreachable from External Devices

**Problem:** After installing Pi-hole via Docker with standard port mapping, DNS worked from the server itself (`127.0.0.1`) but timed out when queried from other devices on the network (`192.168.1.100`).

```bash
# Worked from the server:
nslookup google.com 127.0.0.1

# Timed out from the server and from other devices:
nslookup google.com 192.168.1.100
```

**Root cause:** Docker's default bridge networking creates a virtual network interface and maps ports through it. In some configurations, this translation layer prevents the container from being reachable via the host machine's actual external IP address, particularly for UDP-based protocols like DNS.

**Resolution:** Switched to Docker host network mode. Instead of Docker managing a virtual network, the container shares the host machine's network stack directly. This makes Pi-hole's services immediately accessible on all of the server's network interfaces.

```bash
sudo docker rm -f pihole
sudo docker run -d --name pihole --network host \
  -e TZ=Asia/Manila \
  -e WEBPASSWORD=yourpassword \
  -v /etc/pihole:/etc/pihole \
  -v /etc/dnsmasq.d:/etc/dnsmasq.d \
  --restart unless-stopped \
  pihole/pihole:latest
```

**Lesson learned:** Docker bridge networking works reliably for HTTP services but can cause issues for low-level protocols like DNS. When a container needs to function as a network service reachable by other devices, host network mode removes the translation layer entirely.

---

### Issue 5: Dashboard Inaccessible After Switching to Host Network Mode

**Problem:** After switching to `--network host`, the `-p` port mapping flags used in the previous Docker command no longer have any effect. Pi-hole's web server defaulted back to port 80, which put it in direct conflict with Apache again.

**Root cause:** Port mapping (`-p`) is a Docker bridge networking feature. It does not apply in host network mode. Without it, Pi-hole defaults to port 80 for its dashboard.

**Resolution:** Passed a Pi-hole-specific environment variable to set the web server port directly:

```bash
-e FTLCONF_webserver_port=8080
```

This tells Pi-hole's internal web server to listen on port 8080, keeping it clear of Apache without needing Docker's port mapping.

---

## Final Working Installation

### Command

```bash
sudo docker run -d --name pihole \
  --network host \
  -e TZ=Asia/Manila \
  -e WEBPASSWORD=yourpassword \
  -e FTLCONF_webserver_port=8080 \
  -v /etc/pihole:/etc/pihole \
  -v /etc/dnsmasq.d:/etc/dnsmasq.d \
  --restart unless-stopped \
  pihole/pihole:latest
```

### Set Dashboard Password

```bash
sudo docker exec -it pihole bash
pihole setpassword
exit
```

### Final Stack

| Service | How It Runs | Port | Access |
|---|---|---|---|
| Nextcloud | Bare metal (Apache) | 80 | http://192.168.1.100 |
| Pi-hole DNS | Docker (host network) | 53 | DNS queries only |
| Pi-hole Dashboard | Docker (host network) | 8080 | http://192.168.1.100:8080/admin |
| MySQL | Bare metal | 3306 | Internal only |

---

## Thermal Shutdown Script

The Asus Vivobook runs without its battery as a permanently-on server. Under sustained load, the i5-8250U can overheat. A lightweight monitoring script was added to shut the server down automatically if the CPU temperature exceeds a safe threshold.

### Installation

```bash
sudo apt install -y lm-sensors
sudo sensors-detect
# Answer yes to all prompts
```

### Script

Saved to `/usr/local/bin/thermal-shutdown.sh`:

```bash
#!/bin/bash
TEMP=$(sensors | grep "Core 0" | awk '{print $3}' | cut -d'+' -f2 | cut -d'.' -f1)
if [ "$TEMP" -gt 85 ]; then
    echo "CPU too hot ($TEMP°C), shutting down..." | logger
    sudo shutdown -h now
fi
```

```bash
sudo chmod +x /usr/local/bin/thermal-shutdown.sh
```

### Cron Job

Added to root crontab (`sudo crontab -e`) to run every minute:

```
* * * * * /usr/local/bin/thermal-shutdown.sh
```

---

## Pointing Client Devices to Pi-hole

Rather than changing the router DNS (which would affect every device), only specific machines were pointed at Pi-hole. This works reliably because the PC and homelab are always on at the same time.

### Windows

1. Settings > Network and Internet > Change adapter options
2. Right-click your connection > Properties
3. Double-click Internet Protocol Version 4 (TCP/IPv4)
4. Select "Use the following DNS server"
5. Set Preferred DNS Server to `192.168.1.100`

### macOS

1. System Settings > Network > Details > DNS tab
2. Click `+` and add `192.168.1.100`
3. Click OK and Apply

---

## Useful Docker Commands

```bash
sudo docker ps                  # see all running containers
sudo docker restart pihole      # restart Pi-hole
sudo docker stop pihole         # stop Pi-hole
sudo docker start pihole        # start Pi-hole
sudo docker logs pihole         # view Pi-hole output and errors
```

---

## Key Lessons Learned

**Know when to abandon a broken path.** The manual Pi-hole installation accumulated broken layers across two attempts: a missing web interface, a port conflict, and unconfigured PHP. Recognising when to step back and switch to a fundamentally different approach saved far more time than continuing to patch a broken foundation.

**Docker solves dependency conflicts.** The core problems with the manual install were port conflicts and missing PHP integration. Docker eliminates both by giving each application its own isolated environment with pre-configured dependencies. Nextcloud runs on bare metal. Pi-hole runs in its own container. They share the same machine but cannot interfere with each other.

**Host network mode is necessary for DNS in Docker.** Docker bridge networking works well for HTTP services but can prevent containers from being reachable via the host's external IP for UDP-based protocols like DNS. When a container needs to act as a network service for other devices, `--network host` removes the translation layer entirely.

**In host network mode, configure ports via environment variables.** The `-p` port mapping flag only works in Docker bridge mode. In host network mode, port settings must be passed through application-specific environment variables like `FTLCONF_webserver_port`.

**A 403 that is not a permissions problem is a processing problem.** When a file exists and has correct ownership but still returns 403, the web server cannot handle the file type. In this case, lighttpd had no PHP module and refused to execute `.php` files.

**Diagnose before retrying.** Commands like `sudo lsof -i :53`, `pihole -v`, and `sudo docker logs pihole` each revealed a specific, actionable problem. Retrying the same installation steps without reading the output never produced a different result.

---

## Next Steps

- [ ] Install Jellyfin media server via Docker
- [ ] Set up a homelab dashboard (Homepage or Heimdall) to manage all services in one place
- [ ] Configure HTTPS with a self-signed or Let's Encrypt certificate for Nextcloud
- [ ] Set up remote access via reverse proxy for Nextcloud
- [ ] Customise Pi-hole blocklists for the home network
- [ ] Review and harden server security before any external exposure

---

## Current Status

Pi-hole is running in Docker alongside Nextcloud and is actively filtering DNS queries for connected devices. The web dashboard is accessible at `http://192.168.1.100:8080/admin`. Both Nextcloud and Pi-hole start automatically when the homelab is powered on. A thermal shutdown script monitors CPU temperature every minute and shuts the server down safely if it exceeds 85°C.