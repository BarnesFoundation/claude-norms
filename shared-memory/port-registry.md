---
name: Local Development Port Registry
description: Canonical port assignments across all of Steve's local dev services — check before binding, update after adding
---

# Port Registry

All projects MUST check this registry before binding a port and update it when adding a new service.

| Port | Project | Service | Runtime | Notes |
|------|---------|---------|---------|-------|
| 3001 | Focus-3.0 | react-scripts dev server | Node 22 | React frontend, proxies /api/* to 4006 |
| 3800 | PPT2VXP | Next.js dev server | Node 22 | PPTX→IIIF pipeline, `npm run dev` |
| 4006 | Focus-3.0 | Express server | Node 22 | Full server (prod DB) or local-dev-server |
| 4040 | PPT2VXP | Firebase Emulator UI | Java/Node | `npm run emulators` |
| 8000 | Focus-Image-Resolver | uvicorn (FastAPI) | Python 3.13 (.venv313) | Image recognition API + /debug, /compare, /manage dashboards |
| 8180 | PPT2VXP | Firestore emulator | Java | Part of Firebase emulators |
| 9299 | PPT2VXP | Auth emulator | Node | Part of Firebase emulators |
| 9399 | PPT2VXP | Storage emulator | Node | Part of Firebase emulators |

## ngrok Tunnels (Hobbyist Plan — 3 endpoint max)

| Domain | Local Port | Project |
|--------|-----------|---------|
| `francesco-forgetive-rayden.ngrok-free.dev` | 3800 | PPT2VXP |
| `barnessteve2.ngrok.app` | 3001 | Focus-3.0 |
| `barnessteve3.ngrok.app` | 8000 | Focus-Image-Resolver |

Start each with `ngrok http --url=<domain> <port>`. NEVER start ngrok without `--url` — it will claim the free static domain and conflict with PPT2VXP.

## Rules
- Before starting a new service, check this table for conflicts
- Assign ports in logical ranges if possible (e.g., 8000s for Python, 3000s for Node)
- Update this file AND commit when adding/changing a port
- If a port conflict is detected at runtime, check here first
