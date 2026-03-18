# Virtual Park

Four browser-based 3D experiences of the same imaginary park. Built with Three.js, N64-era aesthetic. The main version is multiplayer — other players appear as birds in real time.

**Live:** [spac-null.github.io/virtual-park](https://spac-null.github.io/virtual-park/)

---

## Experiences

| # | Name | File | Description |
|---|------|------|-------------|
| 01 | The Walking Version | `virtualPark.html` | Low-poly park, playable bird, gems, zones, multiplayer |
| 02 | The Hovering Version | `parkVR.html` | Bird's-eye view, tap to fly |
| 03 | The Remembering Version | `worldmemory.html` | 3D memory palace, persistent pins |
| 04 | The Storied Version | — | Not yet built |

---

## The Walking Version

### Controls

| Input | Action |
|-------|--------|
| Arrow keys / WASD | Move |
| Space | Jump / fly higher |
| E (in open area) | Leave a trace message |
| TAB | Open scoreboard |
| Hold R | Reset gem progress |
| Mobile D-pad | Move (on-screen buttons) |

### Game Systems

**Zones** — 4 areas with distinct audio ambience, automatic crossfade on entry:
- Festival Plaza (start) — warm bass, stage music
- Food & Bar Village — crowd murmur, clatter
- Cultural Grove — sparse melodic tones
- Outpost Ridge — wind, distance

**Gems** — 10 collectables scattered across zones. Progress saved to `localStorage`. Hold R to reset.

**Generative Music** — Web Audio API synthesis starts on first interaction. Master gain → warm filter → delay → oscillator bank with LFO modulation + sub-bass. Runs independently of zone audio.

**Zone Audio** — Per-zone synthesized ambience crossfades as you move between zones. Quieter at night (`nightFactor: 0.3` between 22:00–06:00 in-game time).

**NPC Birds** — Autonomous flock. Wander and avoid the player.

**Trace System** — Press E to leave a text message (max 100 chars) at your position. Appears as a glowing cylinder. Other players' traces sync over WebSocket in real time.

**Night Events** — Dynamic lighting shifts based on in-game time of day.

**Scoreboard** — TAB key. Tracks gem collection, stored in `localStorage`.

**Multiplayer** — See below.

---

## Multiplayer

Real-time position sync over WebSocket. Other players appear as colored birds with floating name labels.

### How it works

- On load, client connects to `wss://park.jaschablume.nl`
- Each player gets a random `adjective-noun` name (e.g. `swift-bird`), persisted in `localStorage`
- Each player gets a random color from a 12-color palette
- Position broadcasts every 150ms when moving
- Remote birds interpolate smoothly (lerp factor 0.15/frame)
- Auto-reconnects after 5s on disconnect

### Status indicator

Top-right corner shows connection state:

```
● LIVE — 3 online     (green = connected)
You: swift-bird

○ offline             (grey = disconnected / reconnecting)
You: swift-bird
```

### WebSocket Protocol

**Server → Client**

| Message type | When | Payload |
|---|---|---|
| `welcome` | On connect | `{id, color, players: [{id, x, z, rotY, color, name}]}` |
| `join` | New player joins | `{id, x, z, rotY, color, name}` |
| `update` | Player moved | `{id, x, z, rotY, color, name}` |
| `leave` | Player disconnected | `{id}` |
| `trace` | Player left a trace | `{id, x, z, text, color}` |

**Client → Server**

| Message type | When | Payload |
|---|---|---|
| `move` | Position changed | `{type, x, z, rotY, name}` |
| `trace` | E key submitted | `{type, x, z, text}` |

### Infrastructure

| Component | Detail |
|---|---|
| Server | `park-server.py` — Python, `websockets` library |
| Host | trident (jascha@trident.tail630536.ts.net) |
| Port | 8766 (internal) |
| Public endpoint | `wss://park.jaschablume.nl` |
| Tunnel | Cloudflare (`~/.cloudflared/config.yml` on trident) |
| Process manager | systemd — `park-server.service` |

**Check server status (on trident):**
```bash
systemctl is-active park-server
journalctl -u park-server -n 20
```

**Cloudflare tunnel entry** (`~/.cloudflared/config.yml`):
```yaml
- hostname: park.jaschablume.nl
  service: http://localhost:8766
```

---

## Local Development

### Quick start

```bash
./start.sh
```

Opens `http://localhost:8000`. Activates conda env, installs deps if needed.

### Manual

```bash
conda create -y -n virtualpark python=3.10
conda activate virtualpark
pip install flask
python server.py
```

`server.py` is a minimal Flask static file server. The multiplayer client always connects to `wss://park.jaschablume.nl` (the live server), so local and live use the same player pool.

### Requirements

- Python 3.10+
- Miniconda / Anaconda
- Modern browser (Web Audio API, WebGL, WebSocket)

---

## Server Setup (reference)

The park WebSocket server runs as a systemd service on trident.

**Server code:** `/srv/scripts/ops/park-server.py`

**Service file:** `/etc/systemd/system/park-server.service`

```bash
# Restart after code changes
sudo systemctl restart park-server

# Restart Cloudflare tunnel (after config changes)
sudo systemctl restart cloudflared

# Test WebSocket from any machine
npx wscat -c wss://park.jaschablume.nl
```

Expected on connect:
```json
{"type":"welcome","id":"db6d9b55","color":"#6bff6b","players":[...]}
```

---

## Tech Stack

- **Three.js** (via CDN) — 3D rendering, scene graph, materials
- **Web Audio API** — generative music, zone ambience
- **WebSocket** — multiplayer position sync
- **Python + websockets** — server
- **Cloudflare Tunnel** — TLS termination, no open ports
- **GitHub Pages** — static hosting for client
