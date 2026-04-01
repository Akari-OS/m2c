# M2C — Media to Context Protocol

> Transform any media into structured, AI-ready context.

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

> Coming Soon

| Language | Status |
|----------|--------|
| TypeScript | Planned |
| Python | Planned |

## Known Implementations

| Project | Status | Description |
|---------|--------|-------------|
| *Your project here* | — | [Submit an implementation](https://github.com/) |

## License

[Apache 2.0](LICENSE)

## Links

- [Protocol Spec (v0.2)](spec/v0.2/protocol.md)
- [Context Supply Spec](spec/v0.2/supply.md)
- [Interoperability Spec](spec/v0.2/interop.md)

### Inspiration

- [TwelveLabs: Context Engineering for Video](https://www.twelvelabs.io/blog/context-engineering-for-video-understanding)
- [Anthropic: Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [MCP (Model Context Protocol)](https://modelcontextprotocol.io)
- [UniVA: Universal Video Agent](https://github.com/univa-agent/univa)
