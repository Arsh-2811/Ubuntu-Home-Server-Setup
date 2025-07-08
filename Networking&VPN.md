**Workflow: Securely Exposing Netdata over Your Tailnet Using Split‑DNS, mkcert, and Nginx**

### 1. Establish an Encrypted Mesh with Tailscale

1. **Purpose:** Create a private, end‑to‑end encrypted network (the “Tailnet”) that connects your server and client devices, bypassing firewalls and NAT.
2. **How it Works:**

   * You install the Tailscale agent on your Ubuntu server and on each device (Windows, macOS, Linux, iOS, Android).
   * Each participant authenticates with your account and is assigned a stable “100.x.y.z” address on your private mesh.
   * All traffic between devices travels over WireGuard, ensuring confidentiality and integrity without any manual port‑forwarding.

---

### 2. Implement Split‑DNS via Pi‑hole

1. **Purpose:** Make custom hostnames (e.g. `netdata.homeserver`) resolve to your server’s Tailnet address—and only inside your Tailnet.
2. **How it Works:**

   * You run Pi‑hole as your Tailnet’s DNS server.
   * In the Tailscale Admin Console you declare a “Split‑DNS domain” (e.g. `homeserver`) and point it to your Pi‑hole’s Tailnet IP. From then on, any DNS query for `*.homeserver` is sent to Pi‑hole rather than the public Internet.
   * In Pi‑hole’s Local DNS Records, you map `netdata.homeserver → 100.x.y.z`.
   * As a result, when any Tailnet‑connected device asks “what is `netdata.homeserver`?”, Pi‑hole answers with your server’s private address.

---

### 3. Create a Trusted Internal CA with mkcert

1. **Purpose:** Obtain browser‑trusted TLS certificates for your internal hostnames—without buying a domain or using a public CA.
2. **How it Works:**

   * You install mkcert on each machine (server and clients). On first run, mkcert generates its own private CA certificate and key, and **automatically adds** that root certificate into the operating system’s or browser’s trust store.
   * Next, you ask mkcert to produce a certificate for “netdata.homeserver.” mkcert signs that certificate with your locally trusted CA.
   * Because your devices already trust the mkcert root, any HTTPS connection to `https://netdata.homeserver` will present a certificate that the browser accepts without warnings.

---

### 4. Configure Nginx as a Reverse‑Proxy with TLS Termination

1. **Purpose:** Expose Netdata’s internal web interface on standard HTTPS ports (443) under the friendly hostname, hiding the actual port (19999) and applying your mkcert‑issued certificate.
2. **How it Works:**

   * Nginx listens on port 443 for incoming TLS connections to `netdata.homeserver`.
   * It uses the certificate and key files that mkcert generated for that hostname. TLS handshake happens here, ensuring the browser sees a valid, trusted certificate.
   * Once the secure connection is established, Nginx inspects the HTTP “Host:” header, matches it to the Netdata block, and internally forwards the request to Netdata’s true listening port (19999) on localhost.
   * Nginx then relays Netdata’s responses back over the encrypted TLS channel to your browser.

---

## ✅ **Workflow: Securely Serving Netdata in a Private Tailnet**

### **1. Components & What They Do**

| Component     | Role in System                             | Default Port(s)                          |
| ------------- | ------------------------------------------ | ---------------------------------------- |
| **Tailscale** | Creates private VPN between devices        | No specific port (uses WireGuard tunnel) |
| **Pi‑hole**   | DNS server (ad-blocker + custom hostnames) | `53` (DNS)                               |
| **Nginx**     | Accepts HTTPS traffic, forwards to Netdata | `80`, `443`                              |
| **Netdata**   | Real-time dashboard for system metrics     | `19999` (internal only)                  |
| **mkcert**    | Creates trusted HTTPS certificates locally | —                                        |

---

### 5. End-to-End Flow: What Happens When You Visit `https://netdata.homeserver`**

#### 📲 Step-by-step Journey

1. **User opens browser** and types:

   ```
   https://netdata.homeserver
   ```

2. **DNS Lookup** happens automatically:

   * Tailscale sees `.homeserver` and sends the DNS request to Pi‑hole (on your home server).
   * Pi‑hole says: "`netdata.homeserver` lives at 100.x.y.z (your server’s Tailnet IP)".

3. **Browser connects to 100.x.y.z on port 443**

   * The browser makes a secure connection (HTTPS) to **port 443**, which goes to Nginx.
   * Nginx presents the TLS certificate created by **mkcert** for `netdata.homeserver`.

4. **TLS Handshake succeeds**

   * No warnings. You see 🔒 lock icon because mkcert's CA is trusted on your device.

5. **Nginx reverse-proxies the request**

   * It looks at the host (`netdata.homeserver`), and internally forwards the request to **localhost:19999** (Netdata).

6. **Netdata serves the dashboard**

   * Netdata responds.
   * Nginx passes that back to your browser—still over the same secure HTTPS connection.

---

### **6. Bonus: How Ad-Blocking Works**

* When any device inside your Tailnet asks for ads (like `ads.google.com`), that DNS request is also sent to **Pi-hole** on port 53.
* Pi-hole checks its blocklist and returns a dummy IP (`0.0.0.0`), blocking the ad.
* Works system-wide for any device using Pi‑hole as DNS via Tailscale config.

---

### 7. Why This Rocks**

* ✅ **No ports open to the internet** → fully private.
* ✅ **Custom hostnames** → no more IPs or ports to remember.
* ✅ **HTTPS with zero cost** → mkcert makes it trusted locally.
* ✅ **Ad‑blocking + DNS resolution in one place** → Pi‑hole.
* ✅ **Runs all on one Ubuntu laptop** → simple, secure home server.

---

**Summary**
By combining Tailscale’s VPN mesh, Pi‑hole’s split‑DNS, mkcert’s zero‑config CA, and Nginx’s TLS‑terminating reverse proxy, you create a seamless, secure, and user‑friendly way to serve Netdata (and other tools) over HTTPS—accessible only to you and your authorized Tailnet devices.