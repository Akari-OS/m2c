# M2C — Media to Context Protocol

> Transform any media into structured, AI-ready context.
>
> 📜 **Type: Specification** — This repo holds the protocol spec only. Implementations live in separate repos. See [Known Implementations](#known-implementations).

## What is M2C?

M2C (Media to Context) is an open protocol that standardizes how AI systems analyze and understand media. Not just video — **any media**: documents, presentations, audio, music, 3D models, vector graphics, and more.

It defines:

1. **Analysis** — What to analyze and how (modular pipeline)
2. **Schema** — How results are structured (MediaContext JSON)
3. **Supply** — How context is delivered to AI (JIT, token budgets, isolation)
4. **Interop** — How M2C works with existing standards (IPTC, C2PA, XMP)

```
Any Media → [M2C Pipeline] → Structured Context (JSON) → AI understands
```

## Why?

AI tools today work with media **blindly**. They receive file paths but don't *understand* what's inside.

- **Before M2C**: "Here's a video. Edit it." → AI guesses
- **After M2C**: "12 scenes, 2 speakers, warm palette, cooking content." → AI understands and proposes

### Where M2C fits

```
IPTC 2025.1  → Labels AI-generated content     (generation metadata)
C2PA         → Proves content provenance        (authenticity)
M2C          → Structures AI analysis results   (understanding) ← NEW
```

No existing standard covers **AI analysis results**. M2C fills this gap.

## Core Concepts

### Supported Media Types

| Type | Examples |
|------|---------|
| Video | .mp4, .mov, .mkv |
| Image | .jpg, .png, .webp |
| Audio | .mp3, .wav, .flac |
| Music | .mp3, .midi |
| Document | .pdf, .docx, .md |
| Presentation | .pptx, .key |
| Spreadsheet | .xlsx, .csv |
| Vector | .svg, .ai |
| 3D | .gltf, .obj |
| Custom | `x-` prefix |

### Analysis Modules

Each analysis capability is an independent, replaceable module:

| Module | Function | Example OSS |
|--------|----------|-------------|
| `scene` | Scene boundary detection | PySceneDetect |
| `transcription` | Speech-to-text + speakers | WhisperX |
| `visual` | Image understanding | BLIP / CLIP |
| `color` | Color palette + mood | MovieColorSchemer |
| `acoustic` | Music/sound events | PANNs |
| `text_extract` | Document text/tables | Apache Tika |
| `summary` | Unified summary | Any LLM |

### Precision Levels

| Level | Use Case |
|-------|----------|
| **Quick** | Browsing, preview |
| **Standard** | Editing, search, AI workflows |
| **Deep** | AI-driven editing, full understanding |

### Context Supply (4 Pillars)

| Pillar | What it does |
|--------|-------------|
| **Write** | Analyze → structured JSON |
| **Select** | Pick the right context for the task (JIT) |
| **Compress** | Summarize to fit token budgets |
| **Isolate** | Separate by modality, time, audience |

### Confidence Scoring

Every result includes a confidence score (0.0–1.0). Quality gates prevent bad context from reaching AI.

## Architecture

```
┌─────────────────────────────────────────────────┐
│                M2C Pipeline                      │
│                                                  │
│  Media URI                                       │
│      ↓                                           │
│  Capability Negotiation                          │
│      ↓                                           │
│  Dispatcher (type + precision → modules)         │
│      ↓                                           │
│  Module 1 ─→ Result 1 ─┐                        │
│  Module 2 ─→ Result 2 ─┤→ Aggregator → Context  │
│  Module N ─→ Result N ─┘                        │
│      ↓                                           │
│  Quality Gate (confidence check)                 │
│      ↓                                           │
│  MediaContext JSON                               │
│      ↓                                           │
│  Context Supply (Select → Compress → Isolate)    │
│      ↓                                           │
│  AI Consumer                                     │
└─────────────────────────────────────────────────┘
```

## Specification

```
spec/
├── v0.2/                    ← Current
│   ├── protocol.md          ← Core protocol spec
│   ├── schema.json          ← MediaContext JSON Schema
│   ├── modules.md           ← Module interface + registry
│   ├── supply.md            ← Context Supply Protocol (4 pillars)
│   ├── interop.md           ← IPTC/C2PA/XMP/Schema.org mapping
│   └── ja/                  ← Japanese translations
└── v0.1/                    ← Previous version
```

### JSON Schema Artifacts

Standalone schema files are available at `schema/`:

```
schema/
├── v0.2/
│   ├── media-context.json    ← MediaContext (canonical copy of spec/v0.2/schema.json)
│   ├── module-manifest.json  ← M2CModuleManifest (modules.md §2.1)
│   ├── context-request.json  ← M2CContextRequest  (supply.md §3.5 / §7.1)
│   ├── context-policy.json   ← M2CContextPolicy   (supply.md §5.2)
│   ├── context-response.json ← M2CContextResponse (supply.md §7.2)
│   └── m2c.xmp               ← XMP namespace template (interop.md §2)
└── v0.1/
    └── media-context.json    ← MediaContext v0.1
```

All `$id` URIs resolve under `https://github.com/Akari-OS/m2c/schema/v0.2/<filename>`.
External validators can pin to these URIs directly.

## Design Principles

| Principle | Description |
|-----------|-------------|
| **Context First** | No AI operation without context |
| **Modular** | Replaceable analysis modules |
| **Local-First** | Fully local execution supported |
| **Secure by Default** | Authentication required for media |
| **Standard-Compatible** | Works with IPTC, C2PA, XMP |
| **Cost-Aware** | Estimates upfront, user chooses depth |

## Examples

The [`examples/`](examples/) directory contains sample MediaContext outputs:

- [`video-cooking.mediacontext.json`](examples/video-cooking.mediacontext.json) — A cooking tutorial video analyzed at Standard precision

## SDK

> Planned for Phase 6 and beyond. The spec is stable enough to reference; reference SDKs will follow.

| Language | Status | Notes |
|----------|--------|-------|
| TypeScript | Planned (Phase 6) | Type definitions + lightweight client |
| Python | Planned (Phase 6) | Analyzer authoring kit |
| Rust | **In use** via [Pool](https://github.com/Akari-OS/pool) | Production Analyzer impl (not yet a standalone SDK crate) |

## Known Implementations

| Project | Status | Description |
|---------|--------|-------------|
| [**Pool**](https://github.com/Akari-OS/pool) | Active | Reference Analyzer implementation. Rust + SQLite + MCP. Ships analyzers for video/audio/image/pdf/office/article/code. |
| [**AKARI Video**](https://github.com/Akari-OS/video) | Active | Reference M2C consumer. Loads MediaContext from Pool via MCP. |
| *Your project here* | — | [Submit an implementation](https://github.com/Akari-OS/m2c/discussions) |

## Related Projects — AKARI OS Ecosystem

M2C is the **semantic layer (Layer 2)** of [AKARI OS](https://github.com/Akari-OS), a "personal AI OS" ecosystem.

| Repo | Role | Relation to M2C |
|---|---|---|
| [Akari-OS/.github](https://github.com/Akari-OS/.github) | Org profile / governance (public canonical) | VISION / ROADMAP |
| [pool](https://github.com/Akari-OS/pool) | Universal Knowledge Store (Rust + SQLite + MCP) | **Implements M2C Analyzers**. Stores MediaContext. |
| [amp](https://github.com/Akari-OS/amp) | Agent Memory Protocol | Complementary protocol (M2C = media memory, AMP = experience memory) |
| [video](https://github.com/Akari-OS/video) | Video editor (Tauri + Rust) | M2C consumer |
| [voice](https://github.com/Akari-OS/voice) | Community feedback | — |

See [`docs/architecture.md`](docs/architecture.md) for a detailed integration guide covering the 7 AKARI agents, Shell modules, MCP tool mapping, and LAN-distributed analyzer deployment.

## License

[Apache 2.0](LICENSE)

## Governance

- [Contributing Guide](CONTRIBUTING.md) — spec change process, PR workflow, commit conventions
- [Code of Conduct](CODE_OF_CONDUCT.md) — Contributor Covenant v2.1
- [Changelog](CHANGELOG.md) — version history

## Links

- [Protocol Spec (v0.2)](spec/v0.2/protocol.md)
- [Context Supply Spec](spec/v0.2/supply.md)
- [Interoperability Spec](spec/v0.2/interop.md)
- [AKARI OS Integration Architecture](docs/architecture.md)

### Inspiration

- [TwelveLabs: Context Engineering for Video](https://www.twelvelabs.io/blog/context-engineering-for-video-understanding)
- [Anthropic: Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [MCP (Model Context Protocol)](https://modelcontextprotocol.io)
- [UniVA: Universal Video Agent](https://github.com/univa-agent/univa)
