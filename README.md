# 🛡️ Raspberry Pi 5 Dual-NIC Network Gateway: AdGuard Home & Zapret

A high-performance, low-latency physical edge-gateway and router appliance deployed on a Raspberry Pi 5. This project centralizes network-wide DNS filtering, ad/tracker blocking, and automated Deep Packet Inspection (DPI) circumvention via a dedicated dual-Ethernet interface setup.

---

## 📌 Network Topology & Architecture

The gateway acts as the physical router and DNS sinkhole between the ISP modem and the local Access Point (AP).

```
                                  [ INTERNET / WAN ]
                                           │
                                           ▼
                                ┌─────────────────────┐
                                │   ISP Modem / ONT   │
                                └──────────┬──────────┘
                                           │ (WAN Uplink)
                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Raspberry Pi 5 (Edge Gateway & DPI Bypass Router)                              │
│                                                                                 │
│   [ WAN Interface ] <── USB 3.0 to Gigabit Ethernet Adapter (e.g. eth1 / usb0)  │
│          │                                                                      │
│          ▼                                                                      │
│   ┌──────────────┐     In-line Inspection     ┌────────────────────────┐        │
│   │ Zapret NFQWS │ <────────────────────────> │ iptables/nftables NAT  │        │
│   └──────────────┘     (DPI Auto-Desync)      └───────────┬────────────┘        │
│          ▲                                                │                     │
│          │ Upstream Queries (DoQ / DoH)                   │ IP Forwarding       │
│   ┌──────────────┐                                        │ (192.168.x.1)       │
│   │ AdGuard Home │ <── Port 53 / DoH / DoQ                ▼                     │
│   └──────────────┘                               [ LAN Interface ]              │
│                                                   Onboard 1GbE (eth0)           │
└───────────────────────────────────────────────────────────┬─────────────────────┘
                                                            │
                                                            ▼ (Trunk / Uplink)
                                                 ┌──────────────────────┐
                                                 │   Wi-Fi AP / Switch  │
                                                 └──────────┬───────────┘
                                                            │
                                     ┌──────────────────────┴─────────────────────┐
                                     ▼                                            ▼
                              [ Wi-Fi Clients ]                            [ Wired Clients ]
                           (Phones, Laptops, TVs)                        (PC, Home Server, etc.)
```

### Interface Assignment
* **LAN Interface (`eth0` - Onboard Gigabit):** Dedicated to internal local traffic. Connected to the home Access Point (AP) / Switch; serves DHCP and acts as the default gateway (`192.168.x.1`) and primary DNS server.
* **WAN Interface (`eth1` / `usb0` - 1 Gbps USB 3.0 Ethernet):** Dedicated WAN uplink plugged directly into the ISP modem/ONT. Handles NAT masquerading, external routing, and Zapret packet modification.

---

## 🔧 Hardware & Software Stack

* **Hardware:** Raspberry Pi 5 (Quad-core Cortex-A76 @ 2.4GHz)
* **Networking Hardware:** 
  * Integrated Broadcom Gigabit Ethernet (LAN)
  * Realtek/ASIX USB 3.0 to Gigabit Ethernet Adapter (WAN)
* **OS:** Linux (openwrt)
* **DNS Engine:** [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome)
* **DPI Mitigation Engine:** [Zapret](https://github.com/bol-van/zapret) deployed via [Keift Automated Installer](https://keift.gitbook.io/guides/linux/install-zapret)
* **Packet Forwarding:** Linux IP Routing (`net.ipv4.ip_forward=1`) + `nftables`/`iptables` NAT Masquerade

---

## 🚀 Key Features

* **Dual-NIC Edge Gateway:** Physical traffic separation between LAN and WAN with dedicated Gigabit line rates and wire-speed routing.
* **Network-Wide Ad & Malware Sinkholing:** Comprehensive DNS-level tracker, advertisement, and threat blocking before requests ever leave the internal network.
* **Automated Layer-7 DPI Circumvention:** Transparent auto-mode packet desynchronization via Zapret (`nfqws`), bypassing ISP throttling, SNI filters, and TCP resets without client-side VPNs.
* **Next-Gen Encrypted DNS (DoQ/DoH):** Ultra-fast DNS-over-QUIC and DNS-over-HTTPS upstream pipelines with split-DNS support for local `.lan` resolution.
* **Zero Client Maintenance:** Any device joining the Wi-Fi AP immediately receives filtered, DPI-bypassed connectivity without manual proxy, VPN, or certificate installs.

---

## ⚙️ Configuration & Deployment

### 1. Network Routing & NAT Setup (Kernel Level)
Enable packet forwarding between LAN (`eth0`) and WAN (`eth1`):

```bash
# Enable IPv4 Forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# Setup NAT masquerading on WAN
sudo iptables -t nat -A POSTROUTING -o eth1 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
sudo iptables -A FORWARD -i eth1 -o eth0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

### 2. Zapret Automated Setup (NFQWS)
Zapret is installed and automated using the Keift Linux guide:

* Reference: [Keift Linux Zapret Guide](https://keift.gitbook.io/guides/linux/install-zapret)
* Mode: **Auto Bypass / Automated DPI Circumvention Mode**

```bash
# Automated installer pull & execution
curl -sSL https://raw.githubusercontent.com/bol-van/zapret/master/install_easy.sh -o install_easy.sh
chmod +x install_easy.sh
sudo ./install_easy.sh
```
*Configured to hook outbound WAN traffic (`eth1`) for dynamic TCP payload segmentation and fake SNI injection.*

### 3. AdGuard Home Configuration
Configured as the primary network resolver (`port 53`), listening on LAN:

#### Upstream DNS Resolvers
```text
[/lan/]127.0.0.1:5354
quic://dns.adguard-dns.com
https://dns.cloudflare.com/dns-query
https://dns10.quad9.net/dns-query
8.8.8.8
76.76.2.0
```

#### Upstream Routing Architecture:
* `[/lan/]127.0.0.1:5354`: Local resolver mapping for internal `.lan` hostnames and reverse DNS lookup (PTR).
* `quic://dns.adguard-dns.com`: Ultra-low latency DNS-over-QUIC (DoQ) primary upstream.
* `https://dns.cloudflare.com/dns-query` & `https://dns10.quad9.net/dns-query`: Encrypted Anycast HTTPS resolvers providing fallback redundancy and threat intelligence filtering.
* `8.8.8.8` & `76.76.2.0` (Control D): Public Anycast DNS fallbacks.

---

## 📊 Performance & Real-World Metrics

* **Throughput:** ~940 Mbps symmetrical NAT forwarding over USB 3.0 Gigabit WAN.
* **DNS Resolution Time:** `< 1 ms` (cached), `< 20 ms` (encrypted DoQ/DoH upstream).
* **System Resource Usage:** `< 4%` CPU utilization on RPi 5 Cortex-A76 under multi-client saturated streaming and gaming loads.

---

## 📜 License & Disclaimers
This repository is documented for technical evaluation, educational, and network research purposes.