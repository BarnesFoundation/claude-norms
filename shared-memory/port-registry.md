---
name: Local Development Port Registry
description: Canonical port assignments across all of Steve's local dev services — check before binding, update after adding
---

# Port Registry

All projects MUST check this registry before binding a port and update it when adding a new service.

| Port | Project | Service | Runtime | Notes |
|------|---------|---------|---------|-------|
| 8000 | Focus-Image-Resolver | uvicorn (FastAPI) | Python 3.13 (.venv313) | Image recognition API + /debug, /compare, /manage dashboards |

## Rules
- Before starting a new service, check this table for conflicts
- Assign ports in logical ranges if possible (e.g., 8000s for Python, 3000s for Node)
- Update this file AND commit when adding/changing a port
- If a port conflict is detected at runtime, check here first
