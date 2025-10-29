# Secure Home Network: Recursive DNS with Pi-hole, Unbound, and Tailscale

## 1. Project Overview

This project details the design and implementation of a secure, private, and high-performance home network infrastructure. The core of this system is a Raspberry Pi (RPi Zero 2W) acting as a network-wide DNS filter (Pi-hole) and a private, recursive DNS resolver (Unbound).

This setup is augmented with Tailscale to provide secure, zero-trust remote access to internal resources, bypassing the limitations of ISP-level CGNAT.

### Project Goals:
* **Privacy:** Prevent third-party DNS providers (like Google, Cloudflare) and my ISP from logging and monetizing my browsing history.
* **Security:** Block network-wide advertisements, trackers, and malicious domains at the DNS level *before* they reach client devices.
* **Control:** Gain full ownership and visibility of my network's DNS resolution chain, from client request to authoritative answer.
* **Secure Access:** Implement a secure method to access my internal network resources (like the Pi-hole admin panel) from anywhere, despite having no public IP address due to CGNAT.

---

## 2. Technical Architecture

### Core Components:
* **Hardware:** Raspberry Pi Zero 2W
* **DNS Filtering:** Pi-hole
* **DNS Resolution:** Unbound
* **Secure Access:** Tailscale
* **Web Interface Security:** SSL/TLS Certificate (via Tailscale MagicDNS HTTPS / Let's Encrypt)

### Network Traffic Flow: LAN Access

This flow illustrates how local clients resolve DNS queries by using the Pi-hole as the network's only recursive DNS server, ensuring all traffic is filtered before leaving the local network.

1.  **Local Client:** A device on the network requests `google.com`.
2.  **Router (DHCP):** The router's DHCP server assigns the Pi-hole's IP address as the *only* DNS server to the client.
3.  **Pi-hole:** The request hits the Pi-hole.
    * If `google.com` is on a blocklist, Pi-hole returns `0.0.0.0`, and the request stops.
    * If allowed, Pi-hole forwards the request to its *only* upstream resolver: `127.0.0.1#5335` (Unbound).
4.  **Unbound:** Unbound receives the query and performs a **recursive lookup**.
    * It queries the Internet's root DNS servers (`.`).
    * Then queries the `.com` TLD servers.
    * Then queries Google's authoritative nameservers for the IP address.
5.  **Response:** Unbound receives the IP and validates it using **DNSSEC**. It passes the answer back to Pi-hole, which passes it to the client.

### Network Traffic Flow: Tailscale VPN Access

This flow illustrates how remote clients maintain privacy and security by using the home network's DNS services through the encrypted Tailscale tunnel, which is essential for bypassing CGNAT restrictions.

1.  **Remote Client (VPN Connected):** A device (e.g., a laptop or phone outside the home) connects to the **Tailnet** via the Tailscale VPN app.
2.  **DNS Routing (Tailscale MagicDNS):**
    * The Tailscale client detects that the Pi-hole's IP (the machine running the Pi-hole/Unbound stack) is set as the **Global Nameserver** for the Tailnet.
    * The client's OS routes all DNS queries through the **encrypted WireGuard tunnel** to the Pi-hole machine's **Tailscale IP (100.x.x.x)**.
3.  **Pi-hole:** The request arrives at the Pi-hole on its `tailscale0` interface.
    * **Filtering:** Pi-hole processes the request against blocklists (If blocked, returns `0.0.0.0`).
    * **Forwarding:** If allowed, Pi-hole forwards the request to its upstream resolver: `127.0.0.1#5335` (Unbound).
4.  **Unbound:** Unbound performs a **recursive lookup** and **DNSSEC validation** (Steps 4 and 5 of the LAN flow).
5.  **Response:** The valid, secure IP address is passed back through the following path:
    * Unbound $\rightarrow$ Pi-hole $\rightarrow$ **Tailscale Tunnel (Encrypted)** $\rightarrow$ Remote Client.

#### Accessing the Web Interface (HTTPS Secure)

When accessing the Pi-hole admin panel:

1.  **Client Request:** A remote client requests the Admin page using the secure MagicDNS hostname (e.g., `https://pringles.ts.net/admin/`).
2.  **Tailscale Interception:** The **`tailscaled`** process on the Pi-hole intercepts the request on port 443.
3.  **Certificate Handshake:** Tailscale presents the **valid Let's Encrypt certificate** (provisioned via MagicDNS HTTPS) to the browser.
4.  **Proxy:** Tailscale decrypts the traffic and uses the **`tailscale serve`** command to forward the request to the local web server (**Lighttpd** running on unencrypted `http://127.0.0.1:80`).
5.  **Secure Access:** The Pi-hole web interface loads securely in the browser, showing the valid lock icon.

---

## 3. Key Features & Rationale (The "Why")

This section explains the technical decisions made for this project.

### Why Pi-hole? (Network-Wide DNS Filtering)

* **The Problem:** Modern applications and websites are saturated with tracking scripts, advertisements, and telemetry that consume bandwidth, slow down performance, and compromise user privacy. Blocking this on every individual device (e.g., with browser extensions) is inefficient and incomplete.
* **The Solution:** By deploying Pi-hole as the central DNS server for the entire network, I can filter all unwanted traffic at its source.
* **The Value:** This provides a "set it and forget it" solution that protects *all* devices on the network, including IoT devices and smart TVs that cannot run ad-blockers themselves. It enhances privacy, secures the network from malware domains, and visibly improves page load times.

### Why Unbound? (Recursive DNS for Privacy)

* **The Problem:** Using a standard Pi-hole setup means forwarding DNS queries to a third-party service (e.g., Cloudflare at `1.1.1.1` or Google at `8.8.8.8`). This solves ad-blocking but still allows that single provider to see and log every website I visit. I am simply trusting a different corporation with my data instead of my ISP.
* **The Solution:** I implemented **Unbound** as a local, **recursive DNS resolver**. Instead of *forwarding* queries, Unbound resolves them *directly*. It traverses the entire DNS hierarchy from the root servers down to the authoritative nameserver for any given domain.
* **The Value:** My browsing history **never leaves my network**. No third party logs my queries. This provides the ultimate level of DNS privacy. Furthermore, Unbound performs **DNSSEC** validation, which cryptographically verifies that the DNS responses are authentic and have not been tampered with (e.g., by a DNS spoofing or cache poisoning attack).

### Why Tailscale? (Secure Access & CGNAT Traversal)

* **The Problem:** My ISP (Airtel) uses **CGNAT (Carrier-Grade NAT)**, meaning I do not have a unique, public IPv4 address. This makes it impossible to open ports or run a traditional VPN (like WireGuard or OpenVPN) to access my home network from outside.
* **The Solution:** I deployed **Tailscale**, a **zero-config mesh VPN** (or "overlay network"). Tailscale creates an end-to-end encrypted network over the public internet, using WireGuard® as its foundation.
* **The Value:** Tailscale's "magic DNS" and NAT traversal capabilities allow me to connect to my Pi-hole (e.g., `http://pringles`) from my phone or laptop anywhere in the world, *without* any port forwarding. It establishes a direct, secure tunnel, solving the CGNAT problem and allowing me to manage my home network and use my own private DNS resolver securely on the go.

### Why an HTTPS Certificate? (Securing Internal Services)

* **The Problem:** By default, the Pi-hole admin panel is served over unencrypted HTTP. While this is on my internal network, it's bad security practice. Any user (or compromised device) on my Wi-Fi could potentially sniff the login credentials in clear text.
* **The Solution:** I plan to deploy a reverse proxy (like Nginx, Caddy, or Traefik) to manage an SSL/TLS certificate for the Pi-hole web interface.
* **The Value:** This enforces an encrypted HTTPS connection, preventing credential sniffing on the local network. It adheres to a **Zero Trust** security model ("never trust, always verify"), where even internal traffic is secured. It also removes browser security warnings and demonstrates a professional approach to securing all web-based services, not just public-facing ones.

---

## 4. Challenges & Troubleshooting

Building this system provided valuable troubleshooting experience:

* **Initial DNS Leak:** After the initial setup, `dnsleaktest.com` showed that queries were still resolving to Cloudflare.
    * **Root Cause:** The browser's built-in **DNS-over-HTTPS (DoH)** feature was bypassing the OS-level DNS settings and sending queries directly to Cloudflare.
    * **Solution:** I disabled DoH in the browser settings, which forced all queries to correctly route through the Pi-hole -> Unbound chain. This highlighted the conflict between local DNS control and modern application-layer protocols.

* **Unbound Service Failure:** The `unbound.service` repeatedly failed to start.
    * **Root Cause:** Using `journalctl -xeu unbound.service`, I identified a syntax error in `/etc/unbound/unbound.conf.d/pi-hole.conf`. A commented-out line was copied without its leading `#` symbol, causing the parser to fail.
    * **Solution:** I corrected the configuration file and used `unbound-checkconf` to validate the syntax before restarting the service, which resolved the issue.

---

## 5. Future Plans

* [ ] Segregate IoT devices onto a separate VLAN to further isolate them from trusted clients.
* [ ] Monitor Unbound's performance and cache statistics to further optimize resolution times.
