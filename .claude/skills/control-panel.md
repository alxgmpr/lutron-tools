# `/control-panel` - Web Control Panel Development

Develop the React web frontend and Bun backend for the CCA packet control panel.

## Usage

```
/control-panel              # Show status and available commands
/control-panel build        # Build production (Turbo: web + backend)
/control-panel dev          # Start backend + frontend (unified flow)
/control-panel backend      # Start backend only
/control-panel frontend     # Start frontend dev server only
```

---

## Quick Start

**Unified flow:** From repo root, one command starts everything:

```bash
npm run dev
```

This runs Turbo and starts both backend (port 5001) and Vite (port 5173). Open http://localhost:5173 (Vite proxies /api to backend).

Based on the arguments provided, take the appropriate action:

### No arguments
Show the current status:
1. Check if dependencies are installed (run `npm install` from root)
2. Show running processes (backend, dev server)
3. List available commands

### "build"
Build production (web + backend):
```bash
npm run build
```

### "dev"
Start backend and frontend (unified flow):
```bash
npm run dev
```

### "backend"
Start backend only (e.g. for API-only use):
```bash
cd backend && bun run src/server.ts
```

### "frontend"
Start frontend dev server only (requires backend on 5001 for /api):
```bash
cd web && npm run dev
```

### "full"
Same as "dev" — use `npm run dev` from root.

---

## Proactive Usage

Use this skill automatically when:
- User asks to modify the web UI
- User mentions React components or frontend
- User wants to start/restart servers
- User is debugging API or SSE issues
- Editing files in `web/` or `backend/` directories

---

## Architecture

```
Browser <-- HTTP/SSE --> Backend Server (port 5001)
                              |
                              +-- UDP <-- ESP32 (port 5000)
                              +-- SQLite DB (packets.db)
```

**Frontend (React):**
- Real-time packet display via SSE
- Device control panels (Pico buttons, dimmer levels)
- Pairing workflows (Bridge, Pico, Vive)

**Backend (Bun):**
- UDP listener for ESP32 packets
- SSE broadcaster for web clients
- REST API for device control
- SQLite for packet history

---

## Critical Files

### Frontend
| File | Purpose |
|------|---------|
| `web/src/App.tsx` | Main application component |
| `web/src/components/controls/*.tsx` | Control panel components |
| `web/src/components/display/*.tsx` | Packet display components |
| `web/src/generated/protocol.ts` | Protocol definitions (hand-maintained) |

### Backend
| File | Purpose |
|------|---------|
| `backend/src/server.ts` | Main server (UDP, SSE, REST) |

---

## API Endpoints

### Packets
```bash
# Fetch recent packets
curl http://localhost:5001/api/packets?limit=100

# Stream live packets (SSE)
curl -N http://localhost:5001/api/packets/stream
```

### Device Control
```bash
# Send button press
curl -X POST http://localhost:5001/api/send \
  -H "Content-Type: application/json" \
  -d '{"device":"0x0595E68D","button":"0x02"}'

# Set dimmer level
curl -X POST http://localhost:5001/api/level \
  -H "Content-Type: application/json" \
  -d '{"bridge":"0x002C90AD","target":"0x06FDEFF4","level":50}'
```

### Pairing
```bash
# Start Vive pairing
curl -X POST http://localhost:5001/api/vive-pair \
  -H "Content-Type: application/json" \
  -d '{"deviceId":"0x0595E68D","roomId":1}'
```

---

## Component Patterns

### SSE Connection
```typescript
useEffect(() => {
  const eventSource = new EventSource('/api/packets/stream');
  eventSource.onmessage = (e) => {
    const packet = JSON.parse(e.data);
    setPackets(prev => [packet, ...prev].slice(0, 100));
  };
  return () => eventSource.close();
}, []);
```

### API Calls
```typescript
const sendButton = async (device: string, button: number) => {
  await fetch('/api/send', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ device, button: `0x${button.toString(16).padStart(2, '0')}` })
  });
};
```

---

## Debugging Tips

1. **Check backend is running:**
   ```bash
   curl http://localhost:5001/api/packets?limit=1
   ```

2. **View SSE stream:**
   ```bash
   curl -N http://localhost:5001/api/packets/stream
   ```

3. **Check for build errors:**
   ```bash
   cd web && npm run build 2>&1 | head -50
   ```

4. **View backend logs:** Backend logs to stdout with timestamps

---

## Protocol Reference

When working with packets, reference:
- `protocol/cca.yaml` - Packet type definitions
- `web/src/generated/protocol.ts` - Frontend protocol utils (hand-maintained, keep in sync with cca.yaml)
