**Home Server Implementation Documentation**

This document outlines the design and configuration steps for turning an old laptop into a self‑managed, encrypted, headless home server accessible over a private VPN, with clean HTTPS‑secured URLs for each service. It covers power management, storage encryption, networking, DNS, reverse‑proxying, and service deployment.

* **Unattended Recovery**: Automatic boot and ZFS unlock via USB after power outages.
* **Headless Operation**: Runs with lid closed; remote rebootable without physical presence.
* **Secure Networking**: Tailscale VPN for internal traffic; Pi‑hole for DNS-level ad‑blocking.
* **Clean HTTPS Access**: Memorable, port‑less URLs secured by mkcert.
* **Centralized Management**: Nginx reverse proxy routes traffic; no open public ports.

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

* [ ] **Obtain and install mkcert.**

  * Notes:

* [ ] **Configure Nginx reverse proxy; test each service endpoint.**

  * Notes:

* [ ] **Document backup and recovery procedures for keys and certs.**

  * Notes:
