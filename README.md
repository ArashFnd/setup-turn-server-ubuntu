# Coturn TURN Server on Ubuntu + Cloudflare + WebRTC
This repo documents a production-ready TURN setup for WebRTC using **coturn** on **Ubuntu 24.04 LTS**, fronted by a **Cloudflare** DNS subdomain, plus a tiny Node/Next.js API that issues time-limited TURN REST credentials. It’s the end-to-end notes I wish I had: copy-pasteable commands, sane defaults, and gotchas called out.

---

## Table of Contents
- [What’s Inside](#whats-inside)
- [Prerequisites](#prerequisites)

---

## What’s Inside
- Step-by-step install on Ubuntu (coturn, ufw, certbot).
- Hardened turnserver.conf with UDP/TCP and TLS (turns:) enabled, and a bounded relay port range.
- Cloudflare DNS setup (A/AAAA record, **DNS-only**, no proxy).
- REST credential generation on the server (Node/Next.js) so the browser never sees the shared secret.
- Browser snippet to plug straight into RTCPeerConnection({ iceServers }).
- Troubleshooting playbook (logging, ports, cert renewals, symmetric NAT tips).

---

## Prerequisites
- Use: **Ubuntu 24.04 LTS (Noble)**, 64-bit. It’s free—no need to “buy” Ubuntu (you pay your VPS provider).
- **Minimum VPS:** 1 vCPU, 1 GB RAM, 20 GB disk, **public IPv4** (add IPv6 if you can).
- **Network must allow:** UDP/TCP **3478**, TLS **5349**, and a relay port range (e.g., **49160–49200/UDP,TCP**).
> If you’re behind a NAT or home router, strongly prefer a cloud VPS with a public IP instead.

## 1) Point a Cloudflare subdomain at your server
  1. In Cloudflare → DNS:
  - Create **A** record:
    - **Name**: `turn` (so FQDN is `turn.yourdomain.com`)
    - **IPv4**: your server’s public IP
    - **Proxy status**: **DNS only** (⚪️ **grey cloud**, not orange). TURN needs raw UDP/TCP; Cloudflare’s proxy breaks it.
  - (Optional) Add AAAA for IPv6 with DNS only as well.
  2. Wait for DNS to propagate (usually quick).

## 2) Install coturn and basics on Ubuntu
   ```bash
  sudo apt update
  sudo apt install -y coturn ufw jq
  ```
  Enable the systemd service (Ubuntu disables it by default via a guard file):
  ```bash
  # Enable service startup
  echo "TURNSERVER_ENABLED=1" | sudo tee /etc/default/coturn
  ```

## 3) (Recommended) Use TURN REST credentials (ephemeral)
  This is the safest & most common WebRTC pattern:
  - coturn uses a shared secret to verify HMAC-based, time-limited usernames/credentials your app generates on the fly.
  - Your app creates a `username` like `<unixTimestamp>:<userId>` and an HMAC-SHA1 signature as the `credential`.
  Pick a realm and shared secret:
  ```bash
  TURN_REALM="webrtc.yourdomain.com"   # can be your main domain; must match in config
  TURN_SECRET="$(openssl rand -hex 32)" # save this; you'll put it in your app server too
  echo "$TURN_SECRET"
  ```

## 4) Get TLS certificate (so you can offer turns: on 5349)
  **Use certbot + standalone (quickest)**
  Make sure ports 80 and 443 are free while issuing:
  ```bash
  sudo apt install -y certbot
  sudo systemctl stop coturn || true
  
  sudo certbot certonly --standalone -d turn.yourdomain.com --agree-tos -m you@example.com --non-interactive
  ```
  Certificates land under `/etc/letsencrypt/live/turn.yourdomain.com/`.

  Auto-renew will run via systemd timer; we’ll tell coturn to use these paths.
