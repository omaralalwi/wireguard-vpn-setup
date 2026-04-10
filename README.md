# WireGuard VPN Setup (Server & Client)

Secure, production-ready setup for a WireGuard VPN server on any Linux server (Ubuntu/Debian recommended), with full-tunnel routing and real-world troubleshooting.

## 📑 Table of Contents

- [Requirements](#requirements)

- [Server Setup](#-server-setup)
    - [1. System Preparation](#1-system-preparation)
    - [2. Install WireGuard](#2-install-wireguard)
    - [3. Generate Server Keys](#3-generate-server-keys)
    - [4. Enable IP Forwarding](#4-enable-ip-forwarding)
    - [5. Configure Firewall & NAT](#5-configure-firewall--nat)
    - [6. WireGuard Server Config](#6-wireguard-server-config)
    - [7. Start Server](#7-start-server)
    - [8. Add Client](#8-add-client)

- [Client Setup](#-client-setup)
    - [Common Config Template](#common-config-template)
    - [Linux](#linux)
    - [macOS](#macos)
    - [Windows](#windows)
    - [Android](#android)
    - [iOS](#ios)
    - [QR Code (Mobile)](#qr-code-mobile)

- [Testing](#-testing)
- [Debugging](#-debugging)
- [Common Issue: UDP Blocking](#-common-issue-udp-blocking)
- [Security](#-security)
- [Notes](#-notes)
- [License](#license)
- 
---

## Requirements

* Linux server (Ubuntu/Debian recommended)
* Root or sudo access
* Public IP address
* Client device (Linux / macOS / Windows / Android / iOS)

---

## ⚠️ Network Note (Important)

Some networks block UDP (e.g., port `51820`).
To improve compatibility, this guide uses **UDP port 443**.

---

# 🖥️ Server Setup

## 1. System Preparation

### Update system

```bash
sudo apt update && sudo apt upgrade -y
```

### Create a non-root user

```bash
sudo adduser deployer
sudo usermod -aG sudo deployer
```

### Configure SSH access

Run from your **local machine**:

```bash
ssh-copy-id deployer@SERVER_IP
```

> ⚠️ Ensure you can SSH as `deployer` before disabling root login.

### Harden SSH

```bash
sudo nano /etc/ssh/sshd_config
```

```ini
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
```

```bash
sudo systemctl restart ssh
```

### Switch to deployer

```bash
su - deployer
```

---

## 2. Install WireGuard

```bash
sudo apt install wireguard -y
```

---

## 3. Generate Server Keys

```bash
sudo mkdir -p /etc/wireguard
sudo chmod 700 /etc/wireguard

wg genkey | sudo tee /etc/wireguard/server_private.key > /dev/null
sudo cat /etc/wireguard/server_private.key | wg pubkey | sudo tee /etc/wireguard/server_public.key > /dev/null

sudo chmod 600 /etc/wireguard/server_private.key
```

---

## 4. Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

---

## 5. Configure Firewall & NAT

Detect interface:

```bash
INTERFACE=$(ip route get 8.8.8.8 | awk '{print $5}')
```

Apply rules:

```bash
sudo iptables -F
sudo iptables -t nat -F

sudo iptables -A INPUT -p udp --dport 443 -j ACCEPT

sudo iptables -A FORWARD -i wg0 -o $INTERFACE -j ACCEPT
sudo iptables -A FORWARD -i $INTERFACE -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT

sudo iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o $INTERFACE -j MASQUERADE
```

Persist rules:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

---

## 6. WireGuard Server Config

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Interface]
Address = 10.10.0.1/24
ListenPort = 443
PrivateKey = <SERVER_PRIVATE_KEY>

PostUp = iptables -A FORWARD -i wg0 -o $INTERFACE -j ACCEPT; iptables -A FORWARD -i $INTERFACE -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -A POSTROUTING -s 10.10.0.0/24 -o $INTERFACE -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -o $INTERFACE -j ACCEPT; iptables -D FORWARD -i $INTERFACE -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT; iptables -t nat -D POSTROUTING -s 10.10.0.0/24 -o $INTERFACE -j MASQUERADE
```

Insert:

```bash
sudo cat /etc/wireguard/server_private.key
```

---

## 7. Start Server

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
sudo wg
```

Verify:

```bash
ip a show wg0
```

---

## 8. Add Client

On client machine:

```bash
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

Add to server:

```bash
sudo nano /etc/wireguard/wg0.conf
```

```ini
[Peer]
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 10.10.0.2/32
```

Restart:

```bash
sudo systemctl restart wg-quick@wg0
```

Get server public key:

```bash
cat /etc/wireguard/server_public.key
```

---

# 💻 Client Setup

## Common Config Template

```ini
[Interface]
PrivateKey = <CLIENT_PRIVATE_KEY>
Address = 10.10.0.2/24
DNS = 1.1.1.1
# MTU (Maximum Transmission Unit)
# Default WireGuard MTU is usually ~1420
# Use lower values if you experience:
# - slow connections
# - some websites not loading
# - random timeouts
#
# Recommended values:
# 1420 → default (best performance if network is clean)
# 1380 → good for restricted networks / VPN over VPN (recommended)
# 1280 → fallback for very restrictive networks
MTU = 1380

[Peer]
PublicKey = <SERVER_PUBLIC_KEY>
Endpoint = SERVER_IP:443
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

---

## Linux

```bash
sudo apt install wireguard -y
nano ~/wg-client.conf
sudo wg-quick up ~/wg-client.conf
```

---

## macOS

* Install WireGuard from App Store
* Add tunnel → “Add Empty Tunnel”
* Paste config → Activate

---

## Windows

* Install WireGuard from official site
* Import tunnel from file
* Activate

---

## Android

* Install WireGuard app
* Add tunnel → Create from scratch or import
* Activate

---

## iOS

* Install WireGuard app
* Add tunnel → Create from scratch or import
* Activate

---

## QR Code (Mobile)

```bash
sudo apt install qrencode -y
qrencode -t ansiutf8 < client.conf
```

---

# 🧪 Testing

```bash
sudo wg
curl ifconfig.me
ping 8.8.8.8
ping google.com
```

Expected:

* Handshake present
* Public IP = server IP
* Internet works

---

# 🐞 Debugging

## No handshake

```bash
sudo tcpdump -ni any udp port 443
```

* No packets → ISP/network blocking UDP
* Packets present → config issue

---

## Connected but no internet

```bash
sysctl net.ipv4.ip_forward
iptables -t nat -L
```

---

## DNS issues

```bash
ping 8.8.8.8
ping google.com
```

> ⚠️ If DNS fails, your system may not apply `resolvconf` automatically.

---

# ⚠️ Common Issue: UDP Blocking

Symptoms:

* No handshake
* No packets in tcpdump

Solution:

```ini
ListenPort = 443
Endpoint = SERVER_IP:443
```

---

# 🔒 Security

* Never commit private keys
* Use placeholders:

```
<SERVER_PRIVATE_KEY>
<CLIENT_PRIVATE_KEY>
```

* Rotate keys periodically
* Restrict SSH access

---

# 📌 Notes

* Subnet: `10.10.0.0/24`
* Each client: `/32`
* Avoid mixing firewall tools

---

## License

MIT
