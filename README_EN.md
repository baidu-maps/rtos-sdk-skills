# Workspace

[中文](README.md)

This repository provides an AI Coding Agent Skill `mapsdk-rtos-app-sdk` for **Baidu Map RTOS SDK (mapsdk-rtos)**, helping developers efficiently integrate **and port** RTOS map applications within intelligent coding tools (e.g. Cursor, Claude Code): deliverable structure, public API header tree, platform porting contract, initialization and authentication, Canvas adapters, map component control, overlay drawing, search and route planning, offline maps, navigation, and scene recognition (mecp).

## Skill Capabilities

- **Initialization and authentication** — License authentication, base configuration, package name/cache path/version info, initialization order, threading model, and integration checklist
- **Platform porting contract and Canvas adapter** — C platform contract `bd_map_*`, C++ adapter classes `*Impl`, platform util free functions, network contract `BNetwork_*`, Canvas contract (71 methods), and `ImageProvider`
- **Minimal adapter skeleton** — An empty implementation skeleton that compiles and links out of the box (6 files, every function to implement, a self-check list, and the replace-order from skeleton to working)
- **Map component control** — `MapViewApi` setup, lifecycle, map state control, render requests, and touch events (static methods + `MapViewHandle`, multi-instance support)
- **Overlay drawing** — Markers, polylines, point lists / GeoJSON, style setup, visibility control, and layer updates
- **Search and route planning** — POI search, reverse geocoding, walking/driving route planning, and route tile preloading
- **Offline maps** — Version requests, downloadable city list, download status, and offline package management
- **Navigation** — `NaviApi` setup, route result integration, navigation startup, and state callbacks
- **Scene recognition (mecp)** — AOI scene recognition (airport / train station / mall / water etc.); data is downloaded online while recognition runs locally, as an independent module

This Skill applies to RTOS map application development that integrates public `*_api.h` headers under `includes/` and implements the `common/` platform porting contract. **All input coordinates must be GCJ02**; other coordinate systems (e.g. device GNSS WGS84) must be converted on the application side first.

## Requirements

- **Platform**: RTOS / desktop simulator (rtos-map-simulator)
- **Language**: C / C++
- **SDK**: mapsdk-rtos (deliverables: `includes/` 54 public headers, `common/` platform porting contract, `libmapsdk.a`)
- **Core APIs**: `MapViewApi`, `AuthLicenseApi`, `SearchApi`, `NaviApi`, `MapOfflineApi`, `BaseApi`, mecp C API (`rtos_mecp.h`)
- **Integration**: Public headers under `includes/` plus the `common/` platform porting contract (`bd_map_*` / `*Impl` / platform util / `BNetwork_*` / Canvas / ImageProvider)

## Directory structure

```
rtos-sdk-skills/
├── README.md                         # Chinese README
├── README_EN.md                      # This file (English)
├── SKILL.md                          # Skill definition (mapsdk-rtos-app-sdk)
└── references/
    ├── init-auth.md                  # Initialization order, threading model, integration checklist, log keywords
    ├── adapter-skeleton.md           # Minimal compilable adapter skeleton
    ├── adapter-build.md              # Platform porting contract and Canvas adapter implementation
    ├── overlay-map-control.md        # Map state control, overlays, touch, route tiles, multi-instance
    ├── search-navi-offline.md        # Search, navigation, and offline maps
    ├── mecp.md                       # Scene recognition (independent module)
    └── demo.md                       # End-to-end runnable examples
```

## How to use

### 1. Clone this repository

```bash
git clone https://github.com/baidu-maps/rtos-sdk-skills.git
cd rtos-sdk-skills
```

### 2. Download from Release (optional)

You can also download the `rtos-sdk-skills.zip` asset from [Releases](https://github.com/baidu-maps/rtos-sdk-skills/releases) and extract it:

```bash
unzip rtos-sdk-skills.zip
cd rtos-sdk-skills
```

### 3. Register the Skill with your AI assistant

Link or copy this repository folder to your environment’s skills directory so the AI can load these docs during conversations.

**Claude Code (local)**

- Skills directory is usually: `~/.claude/skills/`
- Register (symlink, recommended):
  ```bash
  ln -sfn "$(pwd)" ~/.claude/skills/mapsdk-rtos-app-sdk
  ```
- Or copy this repository folder into `~/.claude/skills/mapsdk-rtos-app-sdk`.

**Cursor**

- Skills directory is usually: `~/.cursor/skills-cursor/`
- Register (symlink, recommended):
  ```bash
  ln -sfn "$(pwd)" ~/.cursor/skills-cursor/mapsdk-rtos-app-sdk
  ```
- Or copy this repository folder into `~/.cursor/skills-cursor/mapsdk-rtos-app-sdk`.

### 4. Use in conversation

In a client that supports Skills, when your question involves keywords like “RTOS map SDK”, “mapsdk-rtos”, “port to RTOS”, “MapViewApi”, “AuthLicenseApi”, “SearchApi”, “NaviApi”, “MapOfflineApi”, “BaseApi / coordinate conversion”, “Canvas adapter”, “platform porting contract / common”, “bd_map_* / BNetwork_*”, “WGS84 / GCJ02”, “mecp / scene recognition”, “RTOS overlays”, “RTOS navigation”, “RTOS offline maps”, or “multi-instance map”, the assistant will prefer this repository’s docs and give answers and code that are aligned with Baidu Map RTOS SDK and are compilable, linkable, and portable.

## References

- [SKILL.md](SKILL.md) — Skill trigger guidance, deliverables, compile / `-I` roots, input coordinate system, global hard rules, and quick symptom lookup
- [references/init-auth.md](references/init-auth.md) — Initialization order, threading model, integration checklist, and log keywords
- [references/adapter-skeleton.md](references/adapter-skeleton.md) — Minimal compilable adapter skeleton: 6 files, every function to implement, a self-check list, and the replace-order from skeleton to working
- [references/adapter-build.md](references/adapter-build.md) — Platform porting contract: `bd_map_*`, `*Impl`, platform util, `BNetwork_*`, Canvas (71 methods), and images; `.a` / target environment matching
- [references/overlay-map-control.md](references/overlay-map-control.md) — Map state, markers, polylines, touch, images, route tiles, and multi-instance
- [references/search-navi-offline.md](references/search-navi-offline.md) — Search, navigation, and offline maps
- [references/mecp.md](references/mecp.md) — Scene recognition (independent module)
- [references/demo.md](references/demo.md) — End-to-end runnable examples

## License

This project is an internal Baidu project for authorized use only.
