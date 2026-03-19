# Deploying .Comms (Incus + Cloudflare Tunnel)

This guide matches a typical setup where:

- The stack runs on an **Incus** instance with LAN IP **`10.106.171.2`** (adjust if yours differs).
- Public HTTPS is **`https://comm.sanic.one`** via **cloudflared** (Cloudflare Tunnel).

## 1. DNS & tunnel

1. In Cloudflare DNS, create a proxied hostname `comm` under `sanic.one` (or CNAME as you prefer).
2. In **cloudflared**, map `comm.sanic.one` → your origin. If cloudflared runs **on the same host** as Docker, origin is often `https://127.0.0.1` or `http://127.0.0.1:443` depending on how you terminate TLS.
3. If **Caddy** in compose terminates TLS on 443, point the tunnel to `https://10.106.171.2:443` (from the machine running cloudflared) or bind Caddy to the tunnel connector’s loopback—use the address cloudflared can reach.

## 2. Generate config for `comm.sanic.one`

From this directory:

```bash
cp secrets.env.example secrets.env
# Edit secrets.env — set at minimum:
#   LIVEKIT_NODE_IP='10.106.171.2'
./generate_config.sh comm.sanic.one
```

Answer the prompts (reverse proxy vs direct, video on/off). If Caddy is not the public entrypoint (tunnel in front), choose **behind another reverse proxy** only if you intentionally need a different port binding; many setups still use Caddy on 80/443 internally.

After the first run, to change domain or LiveKit IP:

```bash
./generate_config.sh --overwrite comm.sanic.one
```

## 3. WebRTC / LiveKit vs Cloudflare Tunnel

**Cloudflare Tunnel forwards HTTP(S) and WebSockets; it does not replace UDP media paths for WebRTC.**

- LiveKit needs clients to reach **`rtc.node_ip`** on **TCP 7881** and **UDP 50000–50100** (or your configured range), **or** you must run **TURN** and enable it in `livekit.yml`.
- Setting **`LIVEKIT_NODE_IP=10.106.171.2`** makes LiveKit advertise that address for ICE. Clients must be able to open UDP to that IP (same LAN, VPN, or port-forward from the public Internet).
- If users are only on the public Internet and you have **no** UDP to the Incus IP, configure **TURN** (e.g. coturn or a hosted TURN service) and set `turn.enabled: true` in `livekit.yml` per [LiveKit docs](https://docs.livekit.io/).

Symptoms without TURN or reachable UDP: browser shows **ICE failed** / **add a TURN server** in `about:webrtc`.

## 4. Firewall (host or Incus profile)

Allow at least:

- **80/tcp**, **443/tcp** — web (if exposed directly or for ACME).
- **7881/tcp** — LiveKit RTC TCP.
- **50000–50100/udp** — LiveKit RTP (default range in generated `livekit.yml`).

## 5. Build & run

```bash
docker compose up -d --build
```

Rebuild the web image after URL changes:

```bash
docker compose up -d --build web
```

## 6. Checklist

| Item | Value |
|------|--------|
| Public URL | `https://comm.sanic.one` |
| API (OpenAPI / clients) | `https://comm.sanic.one/api` |
| WebSocket | `wss://comm.sanic.one/ws` |
| LiveKit (signaling) | `wss://comm.sanic.one/livekit` |
| `LIVEKIT_NODE_IP` | Your reachable host/Incus IP (e.g. `10.106.171.2`) |

Support, privacy, and marketing links may still point at **sanic.one**; the **chat app** defaults in the client are **`comm.sanic.one`**.
