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

## 5) Configure coturn
  Open `/etc/turnserver.conf` and paste this **minimal**, **hardened** config (adjust comments/values):
  ```bash
  ########################################################################
  # Core identity
  ########################################################################
  # Your public DNS name
  listening-ip=YOUR_SERVER_PUBLIC_IPV4
  # If you have IPv6, also:
  # listening-ip=YOUR_SERVER_PUBLIC_IPV6
  relay-ip=YOUR_SERVER_PUBLIC_IPV4
  
  # Public hostname (must resolve to this box)
  realm=webrtc.yourdomain.com
  server-name=turn.yourdomain.com
  
  ########################################################################
  # Ports
  ########################################################################
  listening-port=3478
  tls-listening-port=5349
  
  # Restrict relay range (open these in firewall!)
  min-port=49160
  max-port=49200
  
  ########################################################################
  # Security / credentials (TURN REST)
  ########################################################################
  fingerprint
  use-auth-secret
  static-auth-secret=PUT_THE_SECRET_FROM_STEP_3_HERE
  # Optional: tighten valid window (default ~1 day). 3600 = 1 hour
  # stale-nonce=600
  # user-quota=12
  total-quota=1200
  no-multicast-peers
  no-loopback-peers
  no-cli
  
  ########################################################################
  # TLS
  ########################################################################
  cert=/etc/letsencrypt/live/turn.yourdomain.com/fullchain.pem
  pkey=/etc/letsencrypt/live/turn.yourdomain.com/privkey.pem
  dh-file=/etc/ssl/certs/dhparam.pem
  
  ########################################################################
  # Misc best practices
  ########################################################################
  log-file=/var/log/turnserver/turnserver.log
  simple-log
  no-stdout-log
  cli-password=$(openssl rand -hex 16)
  ```
  Create the DH params (once):
  ```bash
  sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
  ```
  Edit the file to set:
  - `YOUR_SERVER_PUBLIC_IPV4` (and IPv6 if used),
  - the `static-auth-secret` you generated,
  - `realm` and `server-name` matching your domain(s),
  - cert paths exactly as certbot created.

## 6) Open the firewall
  Using **ufw**:
  ```bash
  sudo ufw allow 22/tcp
  sudo ufw allow 3478/tcp
  sudo ufw allow 3478/udp
  sudo ufw allow 5349/tcp
  sudo ufw allow 5349/udp
  sudo ufw allow 49160:49200/tcp
  sudo ufw allow 49160:49200/udp
  sudo ufw enable
  sudo ufw status
  ```
  > If your cloud provider has a separate security group, open the same ports there.

## 7) Start coturn and check logs
  ```bash
  sudo systemctl restart coturn
  sudo systemctl enable coturn
  sudo systemctl status coturn --no-pager
  sudo tail -f /var/log/turnserver/turnserver.log
  ```
  You should see it binding on 0.0.0.0:3478, 0.0.0.0:5349, etc., and your chosen relay range.

  **Troubleshooting:**
  In this step, I got below error:
  ```bash
  tail: cannot open /var/log/turnserver/turnserver.log for reading: No such file or directory
  tail: no files remaining
  ```
  Here is **how to fix it**:
  That error just means coturn isn’t writing to a file yet. In the config I gave, we set `log-file=/var/log/turnserver/turnserver.log` and `no-stdout-log`. If the directory/file doesn’t exist or isn’t writable by the turnserver user, tail has nothing to read.
  **solution:**
  1. Edit `/etc/turnserver.conf` and comment out these lines:
  ```bash
  # log-file=/var/log/turnserver/turnserver.log
  # no-stdout-log
  ```
  (Keep `simple-log` or add `verbose` temporarily for debugging.)
  Restart and watch logs:
  ```bash
  sudo systemctl restart coturn
  sudo systemctl status coturn --no-pager
  sudo journalctl -u coturn -f --no-pager
  ```
