# M2C Protocol Specification v0.2

> Status: Draft
> Date: 2026-04-01
> Previous: [v0.1](../v0.1/protocol.md)

## 1. Overview

M2C (Media to Context) is an open protocol for transforming raw media files into structured, AI-ready context. It standardizes:

1. **Analysis** — How media is analyzed (modules, dispatch, precision)
2. **Schema** — How analysis results are structured (MediaContext)
3. **Supply** — How context is delivered to AI consumers (JIT, budget, isolation)
4. **Interoperability** — How M2C integrates with existing standards (IPTC, C2PA, XMP)

### 1.1 Design Principles

| Principle | Description |
|-----------|-------------|
| **Context First** | No AI operation without context. Media MUST be analyzed before use. |
| **Modular** | Each analysis capability is an independent, replaceable module. |
| **Layered** | Context is structured in layers (L0–L3) for efficient delivery. |
| **Scored** | Every result has a confidence score. Quality gates prevent bad context. |
| **Cost-Aware** | Analysis costs are estimated upfront. Users choose precision level. |
| **Secure by Default** | Authentication is REQUIRED (not optional) for media data. |
| **Local-First** | The protocol MUST support fully local execution. Cloud is optional. |
| **Standard-Compatible** | Interoperable with IPTC, C2PA, XMP, and Schema.org. |

### 1.2 Terminology

| Term | Definition |
|------|-----------|
| **Media** | Any digital file: video, image, audio, document, 3D model, etc. |
| **Module** | An independent analysis unit (e.g., scene detection, transcription). |
| **Context** | Structured output (JSON) describing what is *in* the media. |
| **Dispatcher** | Orchestrator that selects modules based on media type and precision. |
| **Precision Level** | How deeply to analyze (Quick / Standard / Deep / custom). |
| **Quality Gate** | Confidence threshold check before accepting results. |
| **Consumer** | Any system that uses MediaContext (AI assistant, search engine, editor). |
| **Supply** | The process of delivering context to a consumer. See [supply.md](./supply.md). |

### 1.3 Protocol Version

M2C uses **date-based versioning**: `YYYY-MM-DD` (e.g., `2026-04-01`).

Version negotiation follows the MCP pattern:
1. Consumer sends its supported version
2. Provider responds with the same version (if supported) or its latest
3. If incompatible, the connection is rejected

---

## 2. Media Types

M2C supports any digital media. The `mediaType` field uses an extensible enum:

### 2.1 Core Types

| Type | Description | Examples |
|------|-------------|---------|
| `video` | Moving images with optional audio | .mp4, .mov, .avi, .mkv, .webm |
| `image` | Static visual content (raster) | .jpg, .png, .gif, .webp, .bmp |
| `audio` | Speech, sound effects, ambient | .mp3, .wav, .aac, .flac, .ogg |
| `music` | Musical compositions | .mp3, .wav, .midi, .flac |
| `document` | Text-based documents | .pdf, .docx, .txt, .md, .html |
| `presentation` | Slide-based content | .pptx, .key, .odp |
| `spreadsheet` | Tabular data | .xlsx, .csv, .tsv |
| `vector` | Vector graphics | .svg, .ai, .eps |
| `3d` | 3D models and scenes | .gltf, .obj, .fbx, .usdz |
| `archive` | Containers (recursively analyzed) | .zip, .tar.gz |
| `data` | Structured/semi-structured data | .json, .xml, .yaml |

### 2.2 Custom Types

Implementations MAY define custom types using the `x-` prefix:

```json
{ "mediaType": "x-figma-design" }
```

Custom types MUST still produce a valid `MediaContext` output.

---

## 3. Analysis Pipeline

### 3.1 Flow

```
1. INPUT:    Media URI + precision level
2. NEGOTIATE: Capability negotiation (modules available)
3. DISPATCH:  Media type → module set selection
4. ESTIMATE:  Cost/time estimation → user confirmation
5. EXECUTE:   Run selected modules (parallel where possible)
6. AGGREGATE: Merge module outputs into MediaContext
7. SCORE:     Calculate overall confidence
8. GATE:      Check against quality threshold
9. OUTPUT:    MediaContext JSON
```

### 3.2 Intent-Driven Dispatch

The dispatcher selects modules based on **three inputs**: media type, precision level, and **user intent**.

#### 3.2.1 Intent (Optional)

An `M2CIntent` object MAY be provided to optimize module selection:

```typescript
interface M2CIntent {
  /** What the user wants to do with the media */
  purpose?: string;  // e.g., "Extract highlights for short-form video"
  /** Known characteristics that skip unnecessary analysis */
  hints?: {
    hasAudio?: boolean;        // false → skip transcription, acoustic
    hasSpeech?: boolean;       // false → skip speaker diarization
    hasMusic?: boolean;        // false → skip music analysis
    cameraType?: 'static' | 'dynamic' | 'multi';  // static → skip scene detection
    speakerCount?: number;     // 1 → skip diarization
    language?: string;         // Pre-set language for transcription
    genre?: string;            // e.g., "tutorial", "vlog", "interview"
  };
  /** Background context for richer summary generation */
  background?: string;  // e.g., "This is part of a cooking series for beginners"
}
```

When intent is provided:
- The dispatcher SHOULD skip modules that are irrelevant (e.g., `hasAudio: false` → skip `transcription` and `acoustic`)
- The `summary` module SHOULD incorporate `purpose` and `background` into the generated summary
- Cost estimation SHOULD reflect the reduced module set

#### 3.2.2 Default Dispatch Table (Without Intent)

When no intent is provided, modules are selected by media type and precision:

| Media Type | Quick | Standard | Deep |
|-----------|-------|----------|------|
| video | scene, summary | + transcription, color | + visual, acoustic, pose |
| image | visual, summary | + color | + pose |
| audio | transcription, summary | + acoustic | + speaker_diarization |
| music | acoustic, summary | + structure | + harmony |
| document | text_extract, summary | + layout | + table_extract |
| presentation | slide_extract, summary | + visual | + layout |
| vector | visual, summary | + color | + structure |
| 3d | visual, summary | + geometry | + material |

#### 3.2.3 Intent-Modified Dispatch Examples

```
Intent: { purpose: "interview highlight reel", hints: { cameraType: "static", speakerCount: 2 } }
Media: video
→ Skip: scene (static camera), pose
→ Prioritize: transcription (speaker diarization), acoustic (silence detection for cuts)
→ Summary: Incorporates "interview" context for relevant highlight suggestions

Intent: { purpose: "add subtitles", hints: { hasMusic: false } }
Media: video
→ Skip: acoustic, color
→ Prioritize: transcription (large model for accuracy)
→ Summary: Focused on speech content and timing

Intent: { hints: { hasAudio: false } }
Media: video (screen recording)
→ Skip: transcription, acoustic, speaker diarization
→ Focus: visual (UI elements, text on screen), scene
```

Implementations MAY extend dispatch rules for custom media types.

### 3.3 Execution Order

Modules MAY execute in parallel unless dependencies exist:

```
independent modules ──┐
                      ├→ summary (depends on all)
independent modules ──┘
```

The `summary` module MUST execute last.

### 3.4 Precision Levels

| Level | Purpose | Cost | Time |
|-------|---------|------|------|
| **Quick** | Browsing, preview, basic categorization | Low | Seconds |
| **Standard** | Editing, search, AI-assisted workflows | Medium | Under 2 min |
| **Deep** | AI-driven editing, style analysis, full understanding | High | Minutes |

Implementations MAY define custom precision levels as strings (e.g., `"ultrafast"`, `"forensic"`).

Cost and time are reported via `M2CEstimate`. Units are implementation-defined.

---

## 4. Module Interface

### 4.1 Common Interface

Every M2C module MUST implement this interface:

```typescript
interface M2CModule {
  /** Unique module identifier */
  id: string;
  /** Human-readable name */
  name: string;
  /** Semantic version */
  version: string;
  /** Supported media types */
  supportedTypes: string[];
  /** Dependencies (module IDs that must run first, or "*" for all) */
  dependencies: string[];

  /** Analyze media */
  analyze(input: M2CModuleInput): Promise<M2CModuleOutput>;

  /** Estimate cost before execution */
  estimate(input: M2CModuleInput): Promise<M2CEstimate>;
}

interface M2CModuleInput {
  /** Media URI (file://, https://, s3://, etc.) */
  mediaUri: string;
  /** Media type */
  mediaType: string;
  /** Duration in seconds (0 for non-temporal media) */
  duration: number;
  /** User intent (purpose, hints, background) — passed to all modules */
  intent?: M2CIntent;
  /** Module-specific options */
  options?: Record<string, unknown>;
  /** Outputs from dependency modules */
  dependencyOutputs?: Record<string, M2CModuleOutput>;
}

interface M2CModuleOutput {
  /** Module ID */
  moduleId: string;
  /** Module version */
  moduleVersion: string;
  /** Execution time in milliseconds */
  durationMs: number;
  /** Confidence score (0.0–1.0) */
  confidence: number;
  /** Module-specific structured data */
  data: unknown;
  /** Output artifact URIs (keyframes, extracted text, etc.) */
  artifacts?: M2CArtifact[];
}

interface M2CArtifact {
  /** Artifact URI */
  uri: string;
  /** MIME type */
  mimeType: string;
  /** Human-readable label */
  label?: string;
}

interface M2CEstimate {
  /** Estimated cost (implementation-defined unit) */
  cost: number;
  /** Cost unit label (e.g., "credits", "USD", "tokens") */
  costUnit: string;
  /** Estimated time in milliseconds */
  timeMs: number;
}
```

Canonical definitions are in [schema.json](./schema.json).
TypeScript interfaces above are informative.

### 4.2 MCP Compatibility

M2C modules MAY be exposed as MCP tool servers. See [interop.md §MCP](./interop.md) for the bridging specification.

### 4.3 Module Registry

See [modules.md §Registry](./modules.md) for the `M2CModuleManifest` specification and module discovery mechanism.

---

## 5. Context Schema (MediaContext)

### 5.1 Four Pillars

Adapted from the Write/Select/Compress/Isolate model. See [supply.md](./supply.md) for the full Context Supply Protocol.

| Pillar | Protocol Component |
|--------|--------------------|
| **Write** | Analysis pipeline → MediaContext JSON |
| **Select** | Context Supply → JIT layer selection |
| **Compress** | Summary module + compression strategies |
| **Isolate** | Context policies for audience/temporal/modality separation |

### 5.2 Three-Layer Structure

```
L0: Tags      — tags, moods                          (~50 tokens)
L1: Summary   — summary, tags, colorPalette, speakers (~200 tokens)
L2: Scenes    — L1 + scenes[].summary, objects        (~500-1000 tokens)
L3: Full      — All fields including transcription     (~2000-5000 tokens)
```

### 5.3 Core Schema

```typescript
interface MediaContext {
  /** Schema version (date-based) */
  schemaVersion: string;  // e.g., "2026-04-01"
  /** Media type */
  mediaType: string;

  // ── L0: Tags ──
  tags: string[];
  moods: string[];  // Recommended vocabulary in appendix

  // ── L1: Summary ──
  summary: string;
  colorPalette?: string[];  // Hex codes
  speakers?: SpeakerInfo[];
  duration: number;

  // ── L2: Scenes (temporal media only) ──
  scenes?: SceneContext[];

  // ── L3: Full (modality-isolated) ──
  transcription?: TranscriptionSegment[];
  acoustic?: AcousticContext;
  visual?: VisualContext;

  // ── Extensions ──
  extensions?: Record<string, unknown>;

  // ── Meta ──
  confidence: number;
  analyzedModules: string[];
  precisionLevel: string;
  analyzedAt: string;  // ISO 8601

  // ── Intent (what the user wanted when analyzing) ──
  intent?: M2CIntent;

  // ── Context Policy (for Supply) ──
  contextPolicy?: ContextPolicy;
}
```

`extensions` is a free-form object for domain-specific data. Implementations SHOULD use namespaced keys (e.g., `"style.cutTempo"`, `"brand.guidelines"`).

### 5.4 Confidence Scoring

#### Per-Module Confidence

Each module output includes `confidence` (0.0–1.0). Calculation method is module-defined but MUST account for input quality.

#### Aggregate Confidence

```
confidence = Σ(module_confidence × module_weight) / Σ(module_weight)
```

Default weights are defined in [modules.md](./modules.md).

#### Quality Gates

| Gate | Threshold | Action |
|------|-----------|--------|
| **Pass** | confidence >= 0.7 | Accept context |
| **Review** | 0.4 <= confidence < 0.7 | Notify consumer for human review |
| **Reject** | confidence < 0.4 | Re-analyze failed modules only |

Implementations SHOULD allow consumers to customize thresholds.

---

## 6. Capability Negotiation

Following the MCP pattern, M2C uses capability negotiation at connection time.

### 6.1 Provider Capabilities

```json
{
  "protocolVersion": "2026-04-01",
  "capabilities": {
    "modules": ["scene", "transcription", "visual", "color", "summary"],
    "mediaTypes": ["video", "image", "audio"],
    "precisionLevels": ["quick", "standard", "deep"],
    "maxDuration": 7200,
    "supply": {
      "contextLevels": ["L0", "L1", "L2", "L3"],
      "compression": true,
      "isolation": true
    },
    "extensions": ["style", "brand"]
  }
}
```

### 6.2 Consumer Request

```json
{
  "protocolVersion": "2026-04-01",
  "requested": {
    "modules": ["scene", "transcription", "summary"],
    "precisionLevel": "standard",
    "contextLevel": "L2",
    "maxTokens": 1000
  }
}
```

### 6.3 Negotiation Flow

1. Consumer sends requested capabilities
2. Provider responds with what it can fulfill
3. If `protocolVersion` mismatch, provider MAY offer an alternative
4. If no agreement, connection is rejected with error

---

## 7. Analysis Agent (ContextAnalyzer)

Implementations SHOULD provide a dedicated analysis agent:

```
User: "Analyze this video"
  ↓
ContextAnalyzer Agent
  ├── Capability negotiation
  ├── Cost estimation → display to user
  ├── User confirms
  ├── Dispatch modules (parallel where possible)
  ├── Aggregate results
  ├── Quality gate
  ├── Store MediaContext
  └── Post-analysis callback (implementation-defined)
```

### 7.1 Re-Analysis

On quality gate failure or user request:

1. Identify low-confidence modules
2. Re-run ONLY those modules
3. Merge new results into existing MediaContext
4. Recalculate aggregate confidence

---

## 8. Security

### 8.1 Principles

Media files often contain private/sensitive content. M2C mandates:

1. **Authentication REQUIRED**: All M2C endpoints MUST require authentication (unlike MCP where it's optional)
2. **Consent**: Users MUST explicitly trigger analysis (no automatic analysis of user media without consent)
3. **Data Locality**: Implementations MUST declare whether media data leaves the device
4. **Artifact Cleanup**: Temporary artifacts (keyframes, etc.) SHOULD be cleaned up after analysis

### 8.2 Transport Options

M2C supports three deployment topologies:

| Topology | Description | Use Case |
|----------|-------------|----------|
| **Local** | Modules run on the same machine as the consumer | Default. Desktop apps, privacy-sensitive media |
| **LAN Distributed** | Modules run on a different machine on the local network | Offload heavy analysis to a powerful machine while editing on a lightweight device |
| **Cloud** | Modules run on remote servers (API) | When local hardware is insufficient or for specialized models |

#### Local (Default)

```
[Consumer + Modules] — same process or stdio
```

Modules are invoked via function calls, stdio, or unix socket. No network traffic. No authentication required.

#### LAN Distributed

```
[Lightweight PC]              [Powerful PC]
 Editor App     ←── LAN ──→  M2C Analyzer
 (consumer)                    (module server)
```

- Consumer discovers analyzer nodes via **mDNS/Bonjour** (zero-config) or manual IP configuration
- Communication uses **HTTP + JSON-RPC** on the local network (same as MCP Streamable HTTP transport)
- Media files are transferred via the local network or accessed from a shared filesystem (NAS, SMB)
- Authentication: SHOULD use a shared secret or local-only token (not full OAuth)
- Data Locality: media stays within the local network. Implementations MUST NOT relay media to external servers without explicit consent
- The consumer's Capability Negotiation discovers which modules are available on which node

**Typical setup:**
1. Install M2C Analyzer on the powerful machine
2. Analyzer broadcasts availability via mDNS: `_m2c._tcp.local`
3. Consumer auto-discovers and shows: "Analysis server found: Desktop-PC (RTX 4090)"
4. User selects which modules to offload

#### Cloud

```
[Consumer] ←── HTTPS ──→ [Cloud M2C Provider]
```

- HTTPS REQUIRED
- Full authentication (OAuth 2.1 or API key)
- Media is uploaded to the provider
- Implementations MUST declare data locality policy

### 8.3 Transport Security

- **Cloud**: HTTPS REQUIRED
- **LAN Distributed**: HTTPS RECOMMENDED, HTTP acceptable on trusted networks
- **Local**: No transport security needed (stdio, unix socket)
- Media URIs with credentials MUST NOT be logged

---

## 9. Error Handling

### 9.1 Error Model

Two layers, following JSON-RPC conventions:

1. **Protocol errors**: Invalid request, unsupported version, authentication failure
2. **Module errors**: Analysis failure, timeout, unsupported media format

### 9.2 Error Codes

| Code | Meaning |
|------|---------|
| `-32600` | Invalid request |
| `-32601` | Unknown module |
| `-32602` | Invalid parameters |
| `-32603` | Internal error |
| `-32700` | Parse error |
| `-33001` | Media not found (URI unreachable) |
| `-33002` | Unsupported media type |
| `-33003` | Analysis timeout |
| `-33004` | Quality gate failure |
| `-33005` | Cost limit exceeded |

### 9.3 Progress Notification

For long-running analyses, providers SHOULD send progress notifications:

```json
{
  "moduleId": "transcription",
  "progress": 0.65,
  "total": 1.0,
  "message": "Processing audio segments..."
}
```

---

## 10. Versioning

- Protocol version: date-based (`YYYY-MM-DD`)
- Schema version: same date-based format in `MediaContext.schemaVersion`
- Module versions: independent semver
- Extensions: independent versioning
- Backward compatibility: consumers MUST ignore unknown fields

### 10.1 Extension Principles

Following MCP's extension design:

1. **Optional**: Extensions MUST NOT be required for core functionality
2. **Additive**: Extensions MUST only add, never remove or modify core behavior
3. **Composable**: Multiple extensions MUST work independently
4. **Independently versioned**: Each extension has its own version

---

## Appendix A: Recommended Mood Vocabulary

Implementations SHOULD use these terms for the `moods` field:

**Energy**: `energetic`, `calm`, `intense`, `relaxed`, `dynamic`
**Tone**: `bright`, `dark`, `warm`, `cool`, `neutral`
**Emotion**: `happy`, `sad`, `dramatic`, `romantic`, `mysterious`, `humorous`
**Style**: `professional`, `casual`, `cinematic`, `minimal`, `vintage`, `modern`

Custom moods MAY be added with the `x-` prefix: `"x-nostalgic"`.

---

## Appendix B: Related Specifications

| Specification | Relationship |
|--------------|-------------|
| [MCP](https://modelcontextprotocol.io) | Module transport compatibility |
| [IPTC 2025.1](https://iptc.org/standards/photo-metadata/) | AI generation labels (complementary) |
| [C2PA](https://c2pa.org/) | Content provenance (complementary) |
| [XMP](https://developer.adobe.com/xmp/docs/) | File-embedded metadata carrier |
| [Schema.org](https://schema.org/) | Web vocabulary mapping |
| [JSON Schema 2020-12](https://json-schema.org/) | Schema definition format |
