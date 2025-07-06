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

### 5. User Experience (Netdata Example)

1. **On Your Mobile or Laptop** (with Tailscale running): you open your browser and type

   ```
   https://netdata.homeserver
   ```
2. **DNS Resolution:** Tailscale intercepts the query for `*.homeserver` and forwards it to Pi‑hole, which returns the Tailnet IP of your server.
3. **TLS Connection:** Your browser connects to that IP on port 443. Nginx presents the mkcert‑signed certificate for `netdata.homeserver`, so the lock icon appears—no warnings.
4. **Proxying:** Nginx passes the request to the real Netdata service on port 19999 and returns the dashboard over the same HTTPS session.
5. **Security Boundaries:**

   * All transport is encrypted twice: first by WireGuard (Tailscale) and second by TLS (Nginx + mkcert).
   * Only devices authenticated into your Tailnet can even resolve or reach `netdata.homeserver`.

---

### 6. Why This Architecture?

* **No Public Exposure:** You never open ports on your home router; only Tailnet members can see or connect.
* **No Paid CA or Public DNS Needed:** mkcert supplies trusted certs, and split‑DNS keeps everything private.
* **Clean, Memorable URLs:** No port numbers; you type a simple hostname under HTTPS.
* **Modular & Extensible:** Repeat the same pattern for Cockpit, Nextcloud, Immich, or any other service—just add DNS entries, mkcert hostnames, and Nginx blocks.

---

**Summary**
By combining Tailscale’s VPN mesh, Pi‑hole’s split‑DNS, mkcert’s zero‑config CA, and Nginx’s TLS‑terminating reverse proxy, you create a seamless, secure, and user‑friendly way to serve Netdata (and other tools) over HTTPS—accessible only to you and your authorized Tailnet devices.