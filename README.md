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
- VPS with public IPv4 (IPv6 optional), Ubuntu 24.04 LTS.
- Domain on Cloudflare; create turn.yourdomain.com as **DNS-only**.
- Open ports: 3478 (UDP/TCP), 5349 (UDP/TCP), and relay range (e.g., 49160–49200).
