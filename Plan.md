**Home Server Implementation Documentation**

This document outlines the design and configuration steps for turning an old laptop into a self‑managed, encrypted, headless home server accessible over a private VPN, with clean HTTPS‑secured URLs for each service. It covers power management, storage encryption, networking, DNS, reverse‑proxying, and service deployment.

---

## 1. Hardware & Base OS

* **Device**: Repurposed laptop running Ubuntu Server 22.04
* **Boot Settings**:

  * UEFI configured to **Power On** automatically when AC power returns (BIOS: "AC Power Loss" → "Power On").
  * `/etc/systemd/logind.conf`: set `HandleLidSwitch=ignore` and `HandleLidSwitchDocked=ignore` to keep the system running with the lid closed.

## 2. Power Management

* **Auto‑Power‑On After Outage**:

  1. Mains out → laptop runs on battery → battery dies → machine powers off.
  2. Mains returns → BIOS automatically powers on.
* **Graceful Low‑Battery Shutdown**:

  * `/etc/UPower/UPower.conf`:

    ```ini
    PercentageLow=10
    PercentageCritical=3
    PercentageAction=2
    CriticalPowerAction=PowerOff
    ```
  * UPower warns at 10% and initiates a clean shutdown at the action threshold.

## 3. Storage & Encryption

* **ZFS with Passphrase Encryption**:

  * Root and datasets encrypted at rest with a manual passphrase.
* **Automatic Unlock via USB**:

  * Generate a raw keyfile (`dd if=/dev/urandom bs=32 count=1 of=/root/zfs-key.bin`) and copy to USB stick.
  * Initramfs hook script (`/etc/initramfs-tools/scripts/local-top/zfs-unlock`) checks for the USB:

    ```bash
    if USB detected; then
      mount USB → zfs load-key -L file:///mnt/zfs-key.bin rpool/ROOT/ubuntu_xxx
    else
      zfs load-key -L prompt rpool/ROOT/ubuntu_xxx
    fi
    ```
  * Rebuild initramfs: `update-initramfs -u`.
  * Fallback to on‑screen passphrase if USB missing.

## 4. Networking & VPN

* **Tailscale**:

  * Install on server and clients; forms a WireGuard‑based mesh VPN (Tailnet).
  * Handles NAT traversal; assigns each node a `100.x.y.z` address.
* **Split‑DNS via Pi‑hole**:

  1. Install Pi‑hole on the server.
  2. In Tailscale Admin → DNS → Restricted Nameservers:

     * Domain: `arsh.net`
     * Nameserver: server’s Tailscale IP (`100.x.y.z`)
     * Enable “Override local DNS”.
  3. In Pi‑hole UI → Local DNS Records:

     ```text
     homeserver.arsh.net         → 100.x.y.z
     netdata.homeserver.arsh.net → 100.x.y.z
     cockpit.homeserver.arsh.net → 100.x.y.z
     pihole.homeserver.arsh.net  → 100.x.y.z
     ```

## 5. Domain & HTTPS

* **Domain Registration**:

  * Purchase `arsh.net` from a domain registrar.
* **Let’s Encrypt Wildcard Certificate** (DNS‑01 Challenge):

  1. Install Certbot: `apt install certbot`
  2. Run manual/DNS‑API challenge:

     ```bash
     certbot certonly --manual --preferred-challenges dns \
       -d homeserver.arsh.net -d '*.homeserver.arsh.net'
     ```
  3. Add required `_acme-challenge` TXT records in registrar DNS.
  4. Certbot issues wildcard cert; automate renewals via DNS API plugin.

## 6. Reverse‑Proxy Configuration

* **Nginx** on Ubuntu Server:

  * Listens on ports 80 and 443; redirects HTTP → HTTPS.
  * Virtual hosts based on `server_name`:

    ```nginx
    server { listen 80; server_name *.homeserver.arsh.net; return 301 https://$host$request_uri; }
    server {
      listen 443 ssl http2;
      server_name netdata.homeserver.arsh.net;
      ssl_certificate     /etc/letsencrypt/live/homeserver.arsh.net/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/homeserver.arsh.net/privkey.pem;
      location / { proxy_pass http://127.0.0.1:19999; }
    }
    # …similar blocks for cockpit (9090) and pihole UI (80)
    ```
  * Reload: `nginx -t && systemctl reload nginx`.

## 7. Service Endpoints

* **Netdata**: `http://localhost:19999` → `https://netdata.homeserver.arsh.net/`
* **Cockpit**: `https://localhost:9090` → `https://cockpit.homeserver.arsh.net/`
* **Pi‑hole**: `http://localhost/admin` → `https://pihole.homeserver.arsh.net/`

## 8. Benefits & Use Cases

* **Unattended Recovery**: Automatic boot and ZFS unlock via USB after power outages.
* **Headless Operation**: Runs with lid closed; remote rebootable without physical presence.
* **Secure Networking**: Tailscale VPN for internal traffic; Pi‑hole for DNS-level ad‑blocking.
* **Clean HTTPS Access**: Memorable, port‑less URLs secured by Let’s Encrypt wildcard cert.
* **Centralized Management**: Nginx reverse proxy routes traffic; no open public ports.

---

## 9. Example Scenarios

### Scenario A: Power Outage and Recovery

1. **Normal Operation:** Services (Netdata, Cockpit, Pi‑hole) are running behind Nginx on the laptop.
2. **Mains Fail:** Laptop switches to battery power and continues serving over Tailscale.
3. **Battery Drains:** At 10% UPower triggers a clean shutdown (saving logs, stopping services).
4. **Mains Restore:** BIOS auto‑powers on the laptop after AC return.
5. **Initramfs Unlock:** USB key is detected, raw ZFS keyfile loaded automatically. Root filesystem mounts.
6. **Service Startup:** Systemd brings up Nginx, Tailscale, Pi‑hole, Netdata, and Cockpit.
7. **Remote Access:** User on mobile types `https://netdata.homeserver.arsh.net/` via Tailscale → Nginx → Netdata UI.

### Scenario B: Headless Remote Reboot and Management

1. **Trigger:** Admin issues `sudo reboot` over SSH or via Cockpit’s web UI.
2. **Shutdown:** Systemd stops services and triggers ZFS scrubs if configured.
3. **Boot:** BIOS powers on automatically, detects no USB, prompts for passphrase if no USB inserted.
4. **Fallback:** If admin has removed USB, they log in locally or via IPMI keyboard to type passphrase.
5. **Services Resume:** Once unlocked, services start normally, reachable at `cockpit.homeserver.arsh.net`.

### Scenario C: Routine Maintenance – Certificate Renewal

1. **Certbot Cron:** Automated DNS‑API plugin renews wildcard cert via DNS‑01 challenge at registrar.
2. **Reload Nginx:** Post-renewal hook runs `systemctl reload nginx`, applying the new certificate without downtime.
3. **Verification:** Admin checks `https://pihole.homeserver.arsh.net/admin/` shows valid HTTPS lock icon.

### Scenario D: Ad‑Blocking Across Devices

1. **Device Connects:** Laptop on coffee‑shop Wi‑Fi; Tailscale tunnels its DNS queries to Pi‑hole.
2. **DNS Requests:** All queries matching `*.arsh.net` go to Pi‑hole; other queries also pass through Pi‑hole for blocking.
3. **Blocking:** Ads and trackers are filtered universally, improving browsing speed and privacy.

## 10. Additional Applications & Access

You can host apps like **Immich** (self‑hosted photo management) and **Nextcloud** (file sync/share) seamlessly within this setup:

1. **Installation**

   * **Immich**: Docker Compose or Podman; listens on port `2283` by default.
   * **Nextcloud**: Docker, Snap, or manual LEMP stack; listens on port `80`/`443` internally.

2. **DNS & Reverse‑Proxy**

   * In Pi‑hole Local DNS Records, add:

     ```text
     immich.homeserver.arsh.net  → 100.x.y.z
     nextcloud.homeserver.arsh.net → 100.x.y.z
     ```
   * In Nginx configuration, add virtual hosts:

     ```nginx
     server {
       listen 443 ssl http2;
       server_name immich.homeserver.arsh.net;
       ssl_certificate     /etc/letsencrypt/live/homeserver.arsh.net/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/homeserver.arsh.net/privkey.pem;
       location / { proxy_pass http://127.0.0.1:2283; }
     }
     server {
       listen 443 ssl http2;
       server_name nextcloud.homeserver.arsh.net;
       ssl_certificate     /etc/letsencrypt/live/homeserver.arsh.net/fullchain.pem;
       ssl_certificate_key /etc/letsencrypt/live/homeserver.arsh.net/privkey.pem;
       location / { proxy_pass http://127.0.0.1:80; }
     }
     ```

3. **Access on Client Devices**

   * **Mobile (iOS/Android)**:

     * Install the **Immich** mobile app, point it to `https://immich.homeserver.arsh.net/`.
     * Install the **Nextcloud** app, configure `https://nextcloud.homeserver.arsh.net/`.
   * **Web Browser**: Navigate to the same URLs in any browser on Tailnet-connected device.
   * **Tailscale**: ensure your mobile is logged in; DNS and routing happen automatically over the Tailnet—no port numbers needed.

4. **User Experience**

   * **Immich App**: automatic camera upload, photo browsing, albums, sharing within your Tailnet.
   * **Nextcloud App**: file sync, document editing, offline access; use WebDAV or built-in features.

## 11. Roadmap (Checklist)

Use this checklist to track each major implementation phase. Leave your personal notes under each item.

* [ ] **Finalize BIOS settings and test auto‑power‑on.**

  * Notes:

* [ ] **Implement and verify UPower low‑battery shutdown.**

  * Notes:

* [ ] **Configure and test USB‑based ZFS unlock.**

  * Notes:

* [ ] **Deploy Tailscale and Pi‑hole; validate split‑DNS resolution.**

  * Notes:

* [ ] **Obtain and install Let’s Encrypt wildcard certificate.**

  * Notes:

* [ ] **Configure Nginx reverse proxy; test each service endpoint.**

  * Notes:

* [ ] **Document backup and recovery procedures for keys and certs.**

  * Notes:
