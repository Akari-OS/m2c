# M2C Context Supply Protocol v0.2

> Status: Draft
> Date: 2026-04-01

## 1. Overview

The Context Supply Protocol defines how structured context produced by the M2C analysis pipeline is delivered to AI consumers. It standardizes the path from raw `MediaContext` JSON to optimally sized, task-appropriate context windows.

The protocol is organized into four independent stages:

```
MediaContext JSON
      │
      ▼
┌───────────┐     ┌───────────┐     ┌────────────┐     ┌───────────┐
│   Write    │ ──▶ │  Select   │ ──▶ │  Compress  │ ──▶ │  Isolate  │
│ (Analyze → │     │ (JIT      │     │ (Reduce    │     │ (Partition │
│  Structure)│     │  Supply)  │     │  Tokens)   │     │  Context)  │
└───────────┘     └───────────┘     └────────────┘     └───────────┘
                                                              │
                                                              ▼
                                                      AI Consumer
                                                      (LLM Prompt)
```

### 1.1 Design Principles

| Principle | Description |
|-----------|-------------|
| **Stage Independence** | Each stage is a standalone process. Implementations MAY swap any stage with a different provider. |
| **Token Budget Aware** | Every stage respects a `maxTokens` constraint. Context MUST fit within the consumer's budget. |
| **Task Adaptive** | The supply pipeline adapts its behavior based on the downstream task type. |
| **Confidence Gated** | Low-confidence context is filtered before it reaches the consumer. |
| **Multi-Media** | The pipeline handles context from multiple media sources in a single request. |

### 1.2 Terminology

| Term | Definition |
|------|-----------|
| **MediaContext** | Structured JSON output from the M2C analysis pipeline (see v0.1 protocol spec). |
| **Context Level** | Granularity tier (L0–L3) that determines how much detail is included. |
| **Token Budget** | Maximum number of tokens allocated for context in an AI prompt. |
| **Context Rot** | Degradation of AI performance caused by stale, irrelevant, or low-quality context. |
| **Context Policy** | Metadata that controls how a context fragment is distributed to consumers. |
| **Consumer** | Any AI system (LLM, agent, tool) that receives context from the supply pipeline. |

---

## 2. Write Stage (Analyze → Structure)

The Write stage transforms raw analysis outputs into the `MediaContext` JSON schema. This stage is defined in the v0.1 protocol spec and is included here for completeness.

### 2.1 Three-Layer Structure

Context is written at three granularity levels:

```
Frame (finest)
  └── Scene (mid)
        └── Media (coarsest)
```

| Layer | Scope | Contents |
|-------|-------|----------|
| **Frame** | Single keyframe / audio segment | Objects, captions, landmarks, word-level transcription |
| **Scene** | Contiguous scene (detected by `scene` module) | Scene summary, aggregated objects, scene-level transcript, mood |
| **Media** | Entire media file | Global summary, tags, speakers, color palette, style profile |

### 2.2 Write Requirements

- Modules MUST write results to the fields defined in the MediaContext schema (see `schema.json`).
- Each written field MUST include a `confidence` score (0.0–1.0).
- Modules MUST NOT overwrite fields written by other modules unless explicitly designed as an aggregator (e.g., `summary`).
- The `analyzedAt` timestamp MUST be set to the UTC time when the Write stage completes.

### 2.3 Incremental Writes

Implementations SHOULD support incremental writes:

1. A new module is executed (e.g., upgrading from Quick to Standard precision).
2. The module writes its output to the existing `MediaContext`.
3. The `analyzedModules` array is updated.
4. The aggregate `confidence` is recalculated.

This avoids re-running the entire pipeline when additional depth is needed.

---

## 3. Select Stage (JIT Supply)

The Select stage determines *which* parts of the `MediaContext` to supply to the consumer. It implements Just-In-Time (JIT) context delivery: supply only what is needed, when it is needed.

### 3.1 Context Levels

| Level | Name | Contents | Token Estimate |
|-------|------|----------|---------------|
| **L0** | Tags | `tags`, `mood` | ~20–50 tokens |
| **L1** | Summary | L0 + `summary`, `colorPalette`, `speakers`, `duration` | ~100–300 tokens |
| **L2** | Scenes | L1 + `scenes[].summary`, `scenes[].objects`, `scenes[].mood` | ~500–2,000 tokens |
| **L3** | Full | All fields including `transcription`, `acoustic`, `visual`, `style` | ~2,000–10,000+ tokens |

Token estimates are approximate and vary with media length and content density.

### 3.2 Task-Adaptive Selection

The selector MUST choose an appropriate context level based on the consumer's task type:

| Task Type | Recommended Level | Rationale |
|-----------|------------------|-----------|
| `search` | L0 (Tags) | Minimal tokens for filtering and ranking |
| `edit_planning` | L1 (Summary) | Enough context to propose an editing plan |
| `scene_editing` | L2 (Scenes) | Scene-level detail for cut decisions |
| `transcription_work` | L3 (Full) | Requires word-level transcript data |

Implementations MAY define additional task types. Unknown task types SHOULD default to L1.

### 3.3 Token Budget Management

When a `maxTokens` budget is specified:

1. The selector MUST NOT produce output exceeding `maxTokens`.
2. If the recommended level exceeds the budget, the selector MUST downgrade to a lower level.
3. If even L0 exceeds the budget, the selector MUST truncate tags and return a minimal context.

### 3.4 Multi-Media Budget Allocation

When selecting context for multiple media sources within a single request, the selector MUST distribute the token budget across sources.

**Default allocation strategy** — proportional to relevance:

```
Given:
  - N media sources
  - Total budget: maxTokens
  - Relevance score r[i] for each source (0.0–1.0)

Allocation:
  budget[i] = maxTokens × (r[i] / Σ r[j])
```

If no relevance scores are available, the selector SHOULD allocate equally:

```
budget[i] = maxTokens / N
```

**Example**: 10 media sources, 1,000 token budget, equal relevance:

- Each source receives ~100 tokens → L0 or truncated L1

### 3.5 ContextSelector Interface

#### TypeScript

```typescript
interface M2CContextSelector {
  /**
   * Select context for one or more media sources.
   *
   * @param request - The context request parameters
   * @returns Selected context, formatted for AI consumption
   */
  select(request: M2CContextRequest): M2CContextResponse;
}

interface M2CContextRequest {
  /** Media IDs to include */
  mediaIds: string[];

  /** Task type that determines default context level */
  taskType: string;

  /** Maximum tokens for the entire response */
  maxTokens?: number;

  /**
   * Minimum confidence threshold.
   * Context with confidence below this value MUST be excluded.
   * Default: 0.4
   */
  minConfidence?: number;

  /**
   * Explicit level override.
   * When set, bypasses task-adaptive selection.
   */
  level?: 'L0' | 'L1' | 'L2' | 'L3';

  /**
   * Per-media relevance scores (0.0–1.0).
   * Keys are media IDs. Missing entries default to 1.0.
   */
  relevance?: Record<string, number>;
}

interface M2CContextResponse {
  /** Selected context per media source */
  contexts: M2CSelectedContext[];

  /** Total estimated tokens consumed */
  totalTokens: number;

  /** The level actually used (may differ from requested if budget-constrained) */
  effectiveLevel: 'L0' | 'L1' | 'L2' | 'L3';
}

interface M2CSelectedContext {
  /** Media ID */
  mediaId: string;

  /** Context level used for this source */
  level: 'L0' | 'L1' | 'L2' | 'L3';

  /** Estimated tokens for this context */
  tokens: number;

  /** The selected context data */
  data: Partial<MediaContext>;
}
```

#### JSON Schema (Request)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/Akari-OS/m2c/schema/v0.2/context-request.json",
  "title": "M2CContextRequest",
  "type": "object",
  "required": ["mediaIds", "taskType"],
  "properties": {
    "mediaIds": {
      "type": "array",
      "items": { "type": "string" },
      "minItems": 1
    },
    "taskType": {
      "type": "string",
      "examples": ["search", "edit_planning", "scene_editing", "transcription_work"]
    },
    "maxTokens": {
      "type": "integer",
      "minimum": 1
    },
    "minConfidence": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "default": 0.4
    },
    "level": {
      "type": "string",
      "enum": ["L0", "L1", "L2", "L3"]
    },
    "relevance": {
      "type": "object",
      "additionalProperties": {
        "type": "number",
        "minimum": 0.0,
        "maximum": 1.0
      }
    }
  }
}
```

---

## 4. Compress Stage (Token Reduction)

The Compress stage reduces context size while preserving information density. Compression is applied *after* selection and *before* isolation.

### 4.1 Compression Strategies

Implementations MUST support at least one compression strategy and SHOULD support all four:

#### 4.1.1 Summary Compression

Replace verbose content with concise summaries.

- **Input**: Full scene descriptions, detailed transcripts
- **Output**: Condensed natural-language summary
- **Method**: LLM-based summarization or extractive summary

```
Before (480 tokens):
  Scene 3: "The host picks up a red bell pepper from the cutting board,
  washes it under the faucet, places it back, then reaches for a knife
  from the magnetic strip on the wall. They begin slicing the pepper
  into thin strips while explaining the technique..."

After (85 tokens):
  Scene 3: "Host prepares red bell pepper: wash, slice into strips.
  Explains cutting technique."
```

#### 4.1.2 Temporal Compression

Remove redundant temporal information.

- **Input**: Frame-by-frame or segment-by-segment data
- **Output**: Consolidated temporal ranges
- **Rules**:
  - Consecutive frames with identical objects and actions MUST be merged into a single range.
  - Silence segments shorter than 0.5 seconds SHOULD be dropped.
  - Repeated BGM mood segments MUST be consolidated.

```
Before:
  Frame 100 (4.0s): person, kitchen, knife
  Frame 101 (4.04s): person, kitchen, knife
  Frame 102 (4.08s): person, kitchen, knife
  Frame 103 (4.12s): person, kitchen, knife, bell_pepper

After:
  4.0s–4.08s: person, kitchen, knife
  4.12s: person, kitchen, knife, bell_pepper (change detected)
```

#### 4.1.3 Modality Filtering

Discard modalities with low information density for the given task.

| Task Type | Keep | Drop |
|-----------|------|------|
| `search` | tags, mood | acoustic, visual, transcription |
| `edit_planning` | summary, scenes, color | acoustic details, word-level transcript |
| `scene_editing` | scenes, visual, color | fine-grained acoustic |
| `transcription_work` | transcription, speakers | visual, color, acoustic |

Implementations MAY define modality priorities for custom task types.

#### 4.1.4 Iterative Summarization

For media with extremely long context (e.g., 60+ minute video):

```
L3 Full Context (10,000+ tokens)
      │
      ▼
Per-Chapter Summary (2,000 tokens)
      │
      ▼
Summary of Summaries (500 tokens)
      │
      ▼
Within Budget ✓
```

Each iteration MUST preserve:

- Key entities (speakers, objects, locations)
- Temporal structure (scene order, act boundaries)
- Confidence scores (propagated as minimum of inputs)

### 4.2 Compression Metadata

Compressed context MUST carry metadata indicating the compression applied:

```json
{
  "compression": {
    "strategy": ["summary", "temporal", "modality_filter"],
    "originalTokens": 8500,
    "compressedTokens": 1200,
    "ratio": 0.14,
    "lossLevel": "moderate"
  }
}
```

The `lossLevel` field indicates information loss:

| Level | Description |
|-------|-------------|
| `none` | No information lost (only formatting changes) |
| `minimal` | Redundancy removed, all key facts preserved |
| `moderate` | Summarized; some details lost but structure preserved |
| `aggressive` | Heavy summarization; only high-level information remains |

---

## 5. Isolate Stage (Context Partitioning)

The Isolate stage partitions context into scoped fragments, ensuring each consumer receives only relevant information. This prevents context pollution across modalities, time ranges, and agent roles.

### 5.1 Three Dimensions of Isolation

#### 5.1.1 Source/Type Isolation

Context MUST be separable by modality and data type:

| Source Type | Fields |
|------------|--------|
| `visual` | `scenes[].objects`, `scenes[].keyframePath`, `visual.*` |
| `transcription` | `transcription[]`, `speakers[]` |
| `acoustic` | `acoustic.*` |
| `color` | `colorPalette`, `scenes[].colorPalette` |
| `summary` | `summary`, `tags`, `mood` |
| `style` | `style.*` |

When a consumer requests only `transcription` context, fields from other source types MUST NOT be included.

#### 5.1.2 Temporal Isolation

Context MUST be filterable by time range:

- When a consumer operates on a specific time range (e.g., 30.0s–60.0s), the Isolate stage MUST:
  1. Include only scenes overlapping the requested range.
  2. Trim transcription segments to the range boundaries.
  3. Include only acoustic events within the range.
  4. Reset any accumulated state from prior time ranges.

- Temporal isolation SHOULD apply a context reset at scene boundaries to prevent information bleed across scenes.

#### 5.1.3 Agent Role Isolation

Context MUST be partitionable by consumer role:

| Role | Typical Context | Rationale |
|------|----------------|-----------|
| `planner` | L1 summary + scene structure | Needs the big picture, not details |
| `worker` | L2–L3 for assigned scene range only | Needs deep detail for a narrow scope |
| `reviewer` | L1 + quality metrics + confidence scores | Needs to assess, not to edit |
| `user` | L0–L1 natural-language summary | Readable presentation |

Implementations MUST support role-based filtering. Consumers SHOULD declare their role in the context request.

### 5.2 Context Policy Schema

Each context fragment MAY carry a `contextPolicy` metadata object that declares its isolation constraints:

```json
{
  "contextPolicy": {
    "audience": ["planner", "worker", "user"],
    "temporalScope": {
      "start": 0,
      "end": 45.2
    },
    "modalities": ["visual", "transcription"],
    "priority": 0.8
  }
}
```

#### Field Definitions

| Field | Type | Description |
|-------|------|-------------|
| `audience` | `string[]` | Roles that SHOULD receive this context. Empty array means all roles. |
| `temporalScope` | `object` | Time range this context covers. `start` and `end` in seconds. |
| `temporalScope.start` | `number` | Start time in seconds (inclusive). |
| `temporalScope.end` | `number` | End time in seconds (inclusive). |
| `modalities` | `string[]` | Source types included. See Section 5.1.1 for valid values. |
| `priority` | `number` | Priority weight (0.0–1.0). Higher-priority fragments are kept when budget is tight. |

#### JSON Schema (Context Policy)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/Akari-OS/m2c/schema/v0.2/context-policy.json",
  "title": "M2CContextPolicy",
  "type": "object",
  "properties": {
    "audience": {
      "type": "array",
      "items": {
        "type": "string",
        "examples": ["planner", "worker", "reviewer", "user"]
      },
      "default": []
    },
    "temporalScope": {
      "type": "object",
      "required": ["start", "end"],
      "properties": {
        "start": { "type": "number", "minimum": 0 },
        "end": { "type": "number", "minimum": 0 }
      }
    },
    "modalities": {
      "type": "array",
      "items": {
        "type": "string",
        "enum": ["visual", "transcription", "acoustic", "color", "summary", "style"]
      }
    },
    "priority": {
      "type": "number",
      "minimum": 0.0,
      "maximum": 1.0,
      "default": 0.5
    }
  }
}
```

### 5.3 Isolation Enforcement

The Isolate stage MUST enforce the following rules:

1. A context fragment with `audience: ["planner"]` MUST NOT be delivered to a `worker` consumer.
2. A context fragment with `temporalScope: { start: 0, end: 30 }` MUST NOT include data from timestamps beyond 30 seconds.
3. A context fragment with `modalities: ["transcription"]` MUST NOT include visual or acoustic data.
4. When budget constraints require dropping fragments, lower-priority fragments (by `priority` field) MUST be dropped first.

---

## 6. Context Rot Prevention

Context rot occurs when stale, irrelevant, or low-quality context degrades AI performance. The supply pipeline MUST implement the following countermeasures:

### 6.1 Confidence Score Filtering

- Context with confidence below `minConfidence` (default: 0.4) MUST be excluded at the Select stage.
- Per-field confidence SHOULD be checked: if a scene's confidence is below threshold, that scene MUST be excluded even if the overall media confidence is above threshold.
- Implementations SHOULD log excluded context for debugging.

### 6.2 Layer Selection as Rot Prevention

Supplying more context than needed is itself a form of rot — it dilutes relevant information with noise.

| Symptom | Cause | Remedy |
|---------|-------|--------|
| AI ignores important context | Context window too large | Downgrade level (L3 → L2 → L1) |
| AI hallucinates media content | Low-confidence fields present | Raise `minConfidence` threshold |
| AI confuses sources | Multiple media contexts mixed | Apply source/type isolation |
| AI loses temporal coherence | Full timeline supplied for local task | Apply temporal isolation |

### 6.3 Compaction

For long-running sessions where context accumulates:

1. Implementations SHOULD periodically re-select and re-compress context.
2. Stale context (e.g., from media no longer in the active workspace) SHOULD be evicted.
3. Compaction MUST preserve the highest-confidence context first.

### 6.4 Freshness Metadata

Context SHOULD carry freshness information:

```json
{
  "freshness": {
    "analyzedAt": "2026-04-01T10:30:00Z",
    "suppliedAt": "2026-04-01T10:31:15Z",
    "ttlSeconds": 3600
  }
}
```

- `analyzedAt`: When the analysis was performed.
- `suppliedAt`: When the context was last supplied to a consumer.
- `ttlSeconds`: Recommended time-to-live. After expiry, consumers SHOULD request fresh context.

---

## 7. Interface Definitions

### 7.1 M2CContextSelector (TypeScript)

Complete interface including all stages:

```typescript
interface M2CContextSelector {
  /**
   * Select, compress, and isolate context for AI consumption.
   *
   * This is the primary entry point. It orchestrates Select → Compress → Isolate.
   *
   * @param request - Context request parameters
   * @returns Context response with selected, compressed, isolated fragments
   */
  select(request: M2CContextRequest): M2CContextResponse;
}

interface M2CContextRequest {
  /** One or more media IDs to retrieve context for */
  mediaIds: string[];

  /** Task type for adaptive level selection */
  taskType: string;

  /** Maximum token budget for the entire response */
  maxTokens?: number;

  /** Minimum confidence threshold (default: 0.4) */
  minConfidence?: number;

  /** Explicit level override (bypasses task-adaptive selection) */
  level?: 'L0' | 'L1' | 'L2' | 'L3';

  /** Per-media relevance scores for budget allocation */
  relevance?: Record<string, number>;

  /** Consumer role for isolation (default: all roles) */
  role?: 'planner' | 'worker' | 'reviewer' | 'user';

  /** Temporal scope filter */
  temporalScope?: {
    start: number;
    end: number;
  };

  /** Modality filter */
  modalities?: ('visual' | 'transcription' | 'acoustic' | 'color' | 'summary' | 'style')[];

  /** Compression preferences */
  compression?: {
    /** Allowed strategies (default: all) */
    strategies?: ('summary' | 'temporal' | 'modality_filter' | 'iterative')[];
    /** Maximum acceptable loss level */
    maxLossLevel?: 'none' | 'minimal' | 'moderate' | 'aggressive';
  };
}

interface M2CContextResponse {
  /** Selected context fragments */
  contexts: M2CSelectedContext[];

  /** Total estimated tokens */
  totalTokens: number;

  /** Effective level after budget adjustment */
  effectiveLevel: 'L0' | 'L1' | 'L2' | 'L3';

  /** Compression metadata (if compression was applied) */
  compression?: {
    strategy: string[];
    originalTokens: number;
    compressedTokens: number;
    ratio: number;
    lossLevel: 'none' | 'minimal' | 'moderate' | 'aggressive';
  };

  /** Freshness metadata */
  freshness: {
    analyzedAt: string;
    suppliedAt: string;
    ttlSeconds: number;
  };
}

interface M2CSelectedContext {
  /** Media ID */
  mediaId: string;

  /** Context level used */
  level: 'L0' | 'L1' | 'L2' | 'L3';

  /** Estimated tokens */
  tokens: number;

  /** Selected and possibly compressed context data */
  data: Partial<MediaContext>;

  /** Context policy for this fragment */
  contextPolicy?: {
    audience: string[];
    temporalScope?: { start: number; end: number };
    modalities: string[];
    priority: number;
  };
}
```

### 7.2 JSON Schema (Response)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://github.com/Akari-OS/m2c/schema/v0.2/context-response.json",
  "title": "M2CContextResponse",
  "type": "object",
  "required": ["contexts", "totalTokens", "effectiveLevel", "freshness"],
  "properties": {
    "contexts": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["mediaId", "level", "tokens", "data"],
        "properties": {
          "mediaId": { "type": "string" },
          "level": { "type": "string", "enum": ["L0", "L1", "L2", "L3"] },
          "tokens": { "type": "integer", "minimum": 0 },
          "data": { "type": "object" },
          "contextPolicy": {
            "$ref": "https://github.com/Akari-OS/m2c/schema/v0.2/context-policy.json"
          }
        }
      }
    },
    "totalTokens": {
      "type": "integer",
      "minimum": 0
    },
    "effectiveLevel": {
      "type": "string",
      "enum": ["L0", "L1", "L2", "L3"]
    },
    "compression": {
      "type": "object",
      "properties": {
        "strategy": {
          "type": "array",
          "items": { "type": "string" }
        },
        "originalTokens": { "type": "integer" },
        "compressedTokens": { "type": "integer" },
        "ratio": { "type": "number", "minimum": 0, "maximum": 1 },
        "lossLevel": {
          "type": "string",
          "enum": ["none", "minimal", "moderate", "aggressive"]
        }
      }
    },
    "freshness": {
      "type": "object",
      "required": ["analyzedAt", "suppliedAt", "ttlSeconds"],
      "properties": {
        "analyzedAt": { "type": "string", "format": "date-time" },
        "suppliedAt": { "type": "string", "format": "date-time" },
        "ttlSeconds": { "type": "integer", "minimum": 0 }
      }
    }
  }
}
```

---

## 8. End-to-End Example

A consumer requests context for 3 media sources to plan an edit:

### Request

```json
{
  "mediaIds": ["vid_001", "vid_002", "img_003"],
  "taskType": "edit_planning",
  "maxTokens": 800,
  "minConfidence": 0.5,
  "role": "planner",
  "compression": {
    "maxLossLevel": "moderate"
  }
}
```

### Pipeline Execution

```
1. SELECT
   - taskType "edit_planning" → L1 (Summary)
   - Budget: 800 tokens / 3 sources = ~267 each
   - L1 fits within ~267 tokens ✓

2. COMPRESS
   - vid_001 L1: 250 tokens → no compression needed
   - vid_002 L1: 310 tokens → summary compression → 220 tokens
   - img_003 L1: 120 tokens → no compression needed
   - Total: 590 tokens (within budget)

3. ISOLATE
   - Role: planner → include summary, scene structure; exclude raw transcription
   - No temporal filter → full media range
   - Attach contextPolicy to each fragment
```

### Response

```json
{
  "contexts": [
    {
      "mediaId": "vid_001",
      "level": "L1",
      "tokens": 250,
      "data": {
        "summary": "Interview with product designer. Two speakers discuss UX trends.",
        "tags": ["interview", "design", "UX"],
        "mood": "professional, thoughtful",
        "speakers": [
          { "id": "SPEAKER_01", "ratio": 0.6 },
          { "id": "SPEAKER_02", "ratio": 0.4 }
        ],
        "duration": 720,
        "confidence": 0.88
      },
      "contextPolicy": {
        "audience": ["planner"],
        "temporalScope": { "start": 0, "end": 720 },
        "modalities": ["summary"],
        "priority": 0.8
      }
    },
    {
      "mediaId": "vid_002",
      "level": "L1",
      "tokens": 220,
      "data": {
        "summary": "Product demo. Single presenter walks through 5 features.",
        "tags": ["demo", "product", "tutorial"],
        "mood": "energetic, clear",
        "speakers": [{ "id": "SPEAKER_01", "ratio": 1.0 }],
        "duration": 300,
        "confidence": 0.91
      },
      "contextPolicy": {
        "audience": ["planner"],
        "temporalScope": { "start": 0, "end": 300 },
        "modalities": ["summary"],
        "priority": 0.8
      }
    },
    {
      "mediaId": "img_003",
      "level": "L1",
      "tokens": 120,
      "data": {
        "summary": "Product hero shot on gradient background. Clean typography.",
        "tags": ["product", "hero", "marketing"],
        "mood": "minimal, premium",
        "colorPalette": ["#1A1A2E", "#E94560", "#FFFFFF"],
        "duration": 0,
        "confidence": 0.95
      },
      "contextPolicy": {
        "audience": ["planner"],
        "temporalScope": { "start": 0, "end": 0 },
        "modalities": ["summary", "color"],
        "priority": 0.6
      }
    }
  ],
  "totalTokens": 590,
  "effectiveLevel": "L1",
  "compression": {
    "strategy": ["summary"],
    "originalTokens": 680,
    "compressedTokens": 590,
    "ratio": 0.87,
    "lossLevel": "minimal"
  },
  "freshness": {
    "analyzedAt": "2026-04-01T10:30:00Z",
    "suppliedAt": "2026-04-01T10:45:00Z",
    "ttlSeconds": 3600
  }
}
```

---

## 9. Relationship to v0.1

This specification extends Section 5 ("Context Supply") of the v0.1 protocol spec. The v0.1 `M2CContextSelector` interface is a subset of the v0.2 interface defined here.

| v0.1 Concept | v0.2 Expansion |
|-------------|---------------|
| JIT Supply (Section 5.1) | Select Stage (Section 3) with task-adaptive selection and multi-media budget allocation |
| Context Selector interface (Section 5.2) | Full `M2CContextRequest` / `M2CContextResponse` with compression and isolation support |
| Four Pillars overview (Section 4.1) | Each pillar expanded into a full stage specification (Sections 2–5) |
| Context Levels (L0–L3) | Token estimates, level downgrade rules, and budget enforcement |

v0.2 is backward compatible: a v0.1 `M2CContextSelector.select()` call (with `mediaIds`, `task`, `maxTokens`) is a valid v0.2 `M2CContextRequest`.

---

## 10. Conformance

### 10.1 Conformance Levels

| Level | Requirements |
|-------|-------------|
| **M2C Supply Basic** | MUST implement Select Stage (Section 3) with L0–L3 levels and task-adaptive selection. |
| **M2C Supply Standard** | Basic + MUST implement at least one Compress strategy (Section 4) and confidence filtering (Section 6.1). |
| **M2C Supply Full** | Standard + MUST implement all four Compress strategies, all three Isolate dimensions (Section 5), and freshness metadata (Section 6.4). |

### 10.2 Extension Points

Implementations MAY extend this specification by:

- Adding custom task types for the Select stage.
- Adding custom compression strategies for the Compress stage.
- Adding custom audience roles for the Isolate stage.
- Adding custom modality types for source/type isolation.

Extensions MUST NOT conflict with the behavior defined in this specification for the standard task types, strategies, roles, and modalities listed above.

---

## 11. Versioning

- This specification follows semantic versioning.
- The `schemaVersion` field in `MediaContext` is NOT changed by this specification (it remains as defined in v0.1).
- The supply protocol version is tracked separately via the `$id` in JSON Schema URIs (e.g., `v0.2/context-request.json`).
