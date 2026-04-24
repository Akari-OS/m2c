# M2C Protocol Specification v0.1

> Status: Draft
> Date: 2026-04-01

## 1. Overview

M2C (Media to Context) is a protocol for transforming raw media files into structured, AI-ready context. It standardizes the analysis pipeline so that any AI system can understand media content before operating on it.

### 1.1 Design Principles

| Principle | Description |
|-----------|-------------|
| **Context First** | No AI operation without context. Media must be analyzed before use. |
| **Modular** | Each analysis capability is an independent, replaceable module. |
| **Layered** | Context is structured in 3 layers: Frame → Scene → Media. |
| **Scored** | Every result has a confidence score. Quality gates prevent bad context. |
| **Cost-Aware** | Analysis costs are estimated upfront. Users choose precision level. |

### 1.2 Terminology

| Term | Definition |
|------|-----------|
| **Media** | A video, image, or audio file. |
| **Module** | An independent analysis unit (e.g., scene detection, transcription). |
| **Context** | Structured JSON output describing what is *in* the media. |
| **Dispatcher** | The orchestrator that selects modules based on media type. |
| **Precision Level** | How deeply to analyze (Quick / Standard / Deep). |
| **Quality Gate** | Confidence threshold check before accepting results. |

---

## 2. Analysis Pipeline

### 2.1 Flow

```
1. INPUT: Media file + precision level
2. DISPATCH: Media type → module set selection
3. EXECUTE: Run selected modules (parallel where possible)
4. AGGREGATE: Merge module outputs into MediaContext
5. SCORE: Calculate overall confidence
6. GATE: Check against quality threshold
7. OUTPUT: MediaContext JSON
```

### 2.2 Dispatch Rules

The dispatcher selects modules based on media type and precision level.

#### Required Modules by Media Type

| Media Type | Quick | Standard | Deep |
|-----------|-------|----------|------|
| **video** | scene, summary | + transcription, color | + visual, acoustic, pose |
| **image** | visual, summary | + color | + pose |
| **audio** | transcription, summary | + acoustic | + speaker_diarization |

#### Execution Order

Modules MAY execute in parallel unless there are dependencies:

```
scene ──────────────┐
transcription ──────┤
visual ─────────────┤→ summary (depends on all)
color ──────────────┤
acoustic ───────────┘
```

`summary` MUST execute last, as it aggregates all other module outputs.

### 2.3 Precision Levels

| Level | Purpose | Estimated Cost | Estimated Time |
|-------|---------|---------------|----------------|
| **Quick** | Browsing, preview, basic categorization | Low (0-1 AC/min) | < 10s |
| **Standard** | Editing, search, AI proposals | Medium (2-4 AC/min) | 30s-2min |
| **Deep** | AI-driven editing, style analysis, full understanding | High (5-10 AC/min) | 1-5min |

AC = Analysis Credit (implementation-defined unit).

---

## 3. Module Interface

### 3.1 Common Interface

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
  supportedTypes: ('video' | 'image' | 'audio')[];
  /** Dependencies (other module IDs that must run first) */
  dependencies: string[];

  /** Analyze a media file */
  analyze(input: M2CModuleInput): Promise<M2CModuleOutput>;

  /** Estimate cost before execution */
  estimate(input: M2CModuleInput): Promise<M2CEstimate>;
}

interface M2CModuleInput {
  /** Absolute path to media file */
  filePath: string;
  /** Media type */
  mediaType: 'video' | 'image' | 'audio';
  /** Media duration in seconds (0 for images) */
  duration: number;
  /** Module-specific options */
  options?: Record<string, unknown>;
  /** Outputs from dependency modules (if any) */
  dependencyOutputs?: Record<string, M2CModuleOutput>;
}

interface M2CModuleOutput {
  /** Module ID */
  moduleId: string;
  /** Module version */
  moduleVersion: string;
  /** Execution time in milliseconds */
  durationMs: number;
  /** Overall confidence for this module's output (0.0-1.0) */
  confidence: number;
  /** Module-specific structured data */
  data: unknown;
}

interface M2CEstimate {
  /** Estimated cost in AC */
  cost: number;
  /** Estimated time in milliseconds */
  timeMs: number;
}
```

### 3.2 MCP Compatibility

M2C modules MAY be exposed as MCP (Model Context Protocol) tool servers:

```json
{
  "name": "m2c_scene",
  "description": "Detect scene boundaries in a video file",
  "inputSchema": {
    "type": "object",
    "properties": {
      "filePath": { "type": "string" },
      "mediaType": { "type": "string", "enum": ["video"] },
      "duration": { "type": "number" }
    },
    "required": ["filePath", "mediaType", "duration"]
  }
}
```

This allows M2C modules to be used by any MCP-compatible AI system.

---

## 4. Context Schema (MediaContext)

### 4.1 Four Pillars

Adapted from TwelveLabs' context management model:

| Pillar | M2C Implementation |
|--------|-------------------|
| **Write** | Modules write structured JSON to MediaContext |
| **Select** | Consumers request specific layers (L0-L3) |
| **Compress** | Summary module creates compact representations |
| **Isolate** | Each modality (visual/audio/text) is stored independently |

### 4.2 Three-Layer Structure

```
L0: Tags     — tags, mood (for search/filter)
L1: Summary  — summary, tags, colorPalette, speakers, duration
L2: Scenes   — L1 + scenes[].summary, scenes[].objects
L3: Full     — All fields including transcription segments
```

### 4.3 Schema Definition

See [schema.json](./schema.json) for the full JSON Schema.

Core structure:

```typescript
interface MediaContext {
  /** Schema version */
  schemaVersion: number;
  /** Media type */
  mediaType: 'video' | 'image' | 'audio';

  // ── L0: Tags ──
  tags: string[];
  mood: string;

  // ── L1: Summary ──
  summary: string;
  colorPalette: string[];
  speakers: SpeakerInfo[];
  duration: number;

  // ── L2: Scenes (video only) ──
  scenes: SceneContext[];

  // ── L3: Full (modality-isolated) ──
  transcription: TranscriptionSegment[];
  acoustic: AcousticContext;
  visual: VisualContext;
  style: StyleProfile | null;

  // ── Meta ──
  confidence: number;
  analyzedModules: string[];
  precisionLevel: 'quick' | 'standard' | 'deep';
  analyzedAt: string;
}
```

### 4.4 Confidence Scoring

#### Per-Module Confidence

Each module output includes a `confidence` field (0.0–1.0).

#### Aggregate Confidence

The overall `MediaContext.confidence` is the weighted average of module confidences:

```
confidence = Σ(module_confidence × module_weight) / Σ(module_weight)
```

Default weights:

| Module | Weight |
|--------|--------|
| scene | 1.0 |
| transcription | 1.5 |
| visual | 1.0 |
| color | 0.5 |
| acoustic | 0.8 |
| summary | 2.0 |

#### Quality Gates

| Gate | Threshold | Action |
|------|-----------|--------|
| **Pass** | confidence >= 0.7 | Accept context |
| **Review** | 0.4 <= confidence < 0.7 | Flag for human review |
| **Reject** | confidence < 0.4 | Re-analyze with different module/settings |

---

## 5. Context Supply (RAG Integration)

### 5.1 JIT (Just-In-Time) Supply

Consumers SHOULD NOT load the full MediaContext into every AI prompt. Instead, select the appropriate layer based on the task:

| Task | Recommended Layer | Rationale |
|------|------------------|-----------|
| Search / Browse | L0 (Tags) | Minimal tokens |
| Flow proposal | L1 (Summary) | Enough to propose |
| Scene editing | L2 (Scenes) | Scene-level detail |
| Subtitle work | L3 (Full) | Need transcription |

### 5.2 Context Selector Interface

```typescript
interface M2CContextSelector {
  /** Select context for a specific task */
  select(
    mediaIds: string[],
    task: string,
    maxTokens?: number,
  ): string;
}
```

---

## 6. Analysis Agent

### 6.1 ContextAnalyzer Agent

Implementations SHOULD provide a dedicated analysis agent that orchestrates the pipeline:

```
User: "Analyze this video"
  ↓
ContextAnalyzer Agent
  ├── Estimates cost → shows to user
  ├── User confirms
  ├── Dispatches modules (parallel)
  ├── Aggregates results
  ├── Runs quality gate
  ├── Stores MediaContext
  └── Triggers context graph update (if supported)
```

The ContextAnalyzer is separate from the main AI assistant. It is a specialist agent focused solely on media understanding.

### 6.2 Re-Analysis

If a quality gate fails or a user requests re-analysis:

1. The ContextAnalyzer identifies low-confidence modules
2. Re-runs only those modules (not the full pipeline)
3. Merges new results into existing MediaContext
4. Re-calculates aggregate confidence

---

## 7. Versioning

- Spec version follows semver: `MAJOR.MINOR.PATCH`
- `schemaVersion` in MediaContext tracks schema changes
- Modules declare their own version independently
- Backward compatibility: consumers MUST handle unknown fields gracefully
