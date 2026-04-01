# M2C Interoperability Specification v0.2

> Status: Draft
> Date: 2026-04-01

## 1. Overview

M2C (Media to Context) fills a gap in the existing media metadata ecosystem. While current standards address AI-generated content labeling and provenance tracking, no standard provides a schema for **AI analysis results** — the structured understanding of what is *inside* a media file.

### 1.1 Three Standards, One Complete Picture

```
┌─────────────────────────────────────────────────────────┐
│              AI Media Metadata Ecosystem                 │
│                                                         │
│  ┌───────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │  IPTC 2025.1  │  │    C2PA      │  │     M2C      │ │
│  │               │  │              │  │              │ │
│  │  AI GENERATED │  │ AI PROVENANCE│  │ AI ANALYSIS  │ │
│  │  "Was this    │  │ "Who made    │  │ "What is     │ │
│  │   made by AI?"│  │  this, when, │  │  inside this │ │
│  │               │  │  and how?"   │  │  media?"     │ │
│  └───────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│          │                 │                  │         │
│          └─────────┬───────┘                  │         │
│                    │                          │         │
│          Existing Standards            NEW: M2C v0.2   │
│          (creation metadata)       (understanding meta) │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Complementary Relationships

| Standard | Question Answered | Relationship to M2C |
|----------|-------------------|---------------------|
| **IPTC 2025.1** | "Was this created/modified by AI?" | M2C analysis engine metadata maps to `AI System Used`. M2C tags/summary can populate IPTC descriptive fields. |
| **C2PA** | "What is the provenance chain?" | M2C analysis is recorded as a C2PA action assertion. MediaContext MAY be embedded as a custom assertion. |
| **Schema.org** | "How should search engines understand this?" | M2C fields map to Schema.org vocabulary for structured data / JSON-LD. |
| **MPEG-7** | "How is multimedia content described?" | M2C is the AI-native successor to MPEG-7 descriptors. Conceptual mapping exists but no runtime interop. |

### 1.3 Conformance Language

The key words "MUST", "MUST NOT", "SHOULD", "SHOULD NOT", and "MAY" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

---

## 2. XMP Custom Namespace

### 2.1 Namespace Definition

M2C defines a custom XMP namespace for embedding analysis results in media files.

| Property | Value |
|----------|-------|
| **Namespace URI** | `https://m2c-protocol.org/ns/v0.2/` |
| **Preferred Prefix** | `m2c` |
| **Schema File** | `https://m2c-protocol.org/schema/v0.2/m2c.xmp` |

### 2.2 XMP Property Mapping

The following table maps MediaContext fields to XMP properties. Implementations MUST use these property names when writing M2C data to XMP.

| MediaContext Field | XMP Property | XMP Type | Notes |
|-------------------|-------------|----------|-------|
| `schemaVersion` | `m2c:SchemaVersion` | Integer | MUST be present |
| `mediaType` | `m2c:MediaType` | Closed Choice (`video`, `image`, `audio`) | MUST be present |
| `summary` | `m2c:Summary` | Lang Alt | Multilingual support via `xml:lang` |
| `tags` | `m2c:Tags` | Bag Text | Unordered set |
| `mood` | `m2c:Mood` | Text | |
| `colorPalette` | `m2c:ColorPalette` | Seq Text | Ordered list of hex codes |
| `duration` | `m2c:Duration` | Real | Seconds |
| `confidence` | `m2c:Confidence` | Real | 0.0–1.0 |
| `precisionLevel` | `m2c:PrecisionLevel` | Closed Choice (`quick`, `standard`, `deep`) | |
| `analyzedAt` | `m2c:AnalyzedAt` | Date | ISO 8601 |
| `analyzedModules` | `m2c:AnalyzedModules` | Bag Text | Module IDs |
| `speakers` | `m2c:Speakers` | Bag Struct | See Section 2.3 |
| `scenes` | `m2c:Scenes` | Seq Struct | See Section 2.3 |

Complex fields (`transcription`, `acoustic`, `visual`, `style`) SHOULD NOT be embedded in XMP due to size constraints. These fields MUST be stored in the sidecar JSON file (see Section 7).

### 2.3 Struct Definitions

**m2c:Speaker**

| Field | XMP Property | Type |
|-------|-------------|------|
| `id` | `m2c:SpeakerId` | Text |
| `label` | `m2c:SpeakerLabel` | Text |
| `ratio` | `m2c:SpeakerRatio` | Real |

**m2c:Scene**

| Field | XMP Property | Type |
|-------|-------------|------|
| `index` | `m2c:SceneIndex` | Integer |
| `start` | `m2c:SceneStart` | Real |
| `end` | `m2c:SceneEnd` | Real |
| `summary` | `m2c:SceneSummary` | Text |
| `mood` | `m2c:SceneMood` | Text |
| `confidence` | `m2c:SceneConfidence` | Real |

### 2.4 File Embedding Patterns

| Format | Method | Support Level |
|--------|--------|---------------|
| **JPEG** | Embed in XMP packet within APP1 marker | Full — MUST support |
| **PNG** | Embed in `iTXt` chunk with keyword `XML:com.adobe.xmp` | Full — MUST support |
| **PDF** | Embed in XMP metadata stream | Full — SHOULD support |
| **MP4** | Embed in `uuid` box (XMP UUID: `BE7ACFCB97A942E89C71999491E3AFAC`) | Partial — see Section 2.5 |
| **WebM** | Not supported by XMP specification | None — use sidecar only |
| **WAV** | Embed in `_PMX` chunk | Partial — SHOULD support |
| **FLAC** | Embed in `VORBIS_COMMENT` block | Partial — MAY support |

### 2.5 MP4 / Video Constraints

Embedding XMP in video files has known limitations:

1. **Non-standard XMP box**: The ISO 14496-12 (MP4) specification does not define a standard XMP container. The `uuid` box approach is an Adobe convention, not universally supported.
2. **File size impact**: Video files are large; adding XMP has negligible relative impact. However, re-muxing the container to insert XMP MAY require rewriting the entire file.
3. **Streaming incompatibility**: XMP embedded in MP4 is not accessible during HTTP streaming (HLS/DASH).
4. **Tool support**: ExifTool supports reading/writing XMP in MP4. FFmpeg does not natively preserve XMP.

**Recommendation**: For video and audio files, implementations SHOULD prefer sidecar JSON (Section 7) and treat XMP embedding as optional.

### 2.6 Example: XMP Packet

```xml
<?xpacket begin="﻿" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/">
  <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
    <rdf:Description
      xmlns:m2c="https://m2c-protocol.org/ns/v0.2/"
      m2c:SchemaVersion="1"
      m2c:MediaType="video"
      m2c:Summary="Cooking vlog. 3 recipes demonstrated in a bright kitchen."
      m2c:Mood="bright, casual"
      m2c:Duration="342.5"
      m2c:Confidence="0.92"
      m2c:PrecisionLevel="standard"
      m2c:AnalyzedAt="2026-04-01T12:00:00Z">
      <m2c:Tags>
        <rdf:Bag>
          <rdf:li>cooking</rdf:li>
          <rdf:li>vlog</rdf:li>
          <rdf:li>recipe</rdf:li>
        </rdf:Bag>
      </m2c:Tags>
      <m2c:ColorPalette>
        <rdf:Seq>
          <rdf:li>#F5E6D3</rdf:li>
          <rdf:li>#2C5F2D</rdf:li>
          <rdf:li>#E8A87C</rdf:li>
        </rdf:Seq>
      </m2c:ColorPalette>
      <m2c:AnalyzedModules>
        <rdf:Bag>
          <rdf:li>scene</rdf:li>
          <rdf:li>transcription</rdf:li>
          <rdf:li>visual</rdf:li>
          <rdf:li>color</rdf:li>
          <rdf:li>summary</rdf:li>
        </rdf:Bag>
      </m2c:AnalyzedModules>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

---

## 3. IPTC Mapping

### 3.1 IPTC Core / Extension Property Mapping

M2C descriptive fields can be mapped to IPTC photo metadata properties. This allows M2C-analyzed media to be discoverable by IPTC-aware tools (DAM systems, stock photo platforms, CMS).

| MediaContext Field | IPTC Property | IPTC Schema | Mapping Quality | Notes |
|-------------------|--------------|-------------|-----------------|-------|
| `summary` | `Description` (dc:description) | IPTC Core | Direct | M2C summary → IPTC Description. Implementations SHOULD prefix with `[M2C]` to indicate AI-generated origin. |
| `tags` | `Keywords` (dc:subject) | IPTC Core | Direct | 1:1 mapping. |
| `mood` | — | — | No mapping | IPTC has no mood field. Use M2C XMP extension (Section 2). |
| `speakers[].label` | `Person Shown in the Image` (Iptc4xmpExt:PersonInImage) | IPTC Extension | Partial | Only maps when speaker has a human-readable label. Speaker ID alone is insufficient. |
| `scenes[].summary` | — | — | No mapping | IPTC has no scene-level structure. Use M2C XMP. |
| `colorPalette` | — | — | No mapping | IPTC has no color metadata. Use M2C XMP. |
| `analyzedAt` | `Date Created` (photoshop:DateCreated) | IPTC Core | Semantic mismatch | `analyzedAt` is the analysis timestamp, not the media creation date. Implementations MUST NOT overwrite existing `Date Created`. Use `m2c:AnalyzedAt` instead. |
| `mediaType` | — | — | Implicit | IPTC determines media type from the file format, not from metadata. |

### 3.2 IPTC 2025.1 AI Properties

IPTC 2025.1 introduced properties for AI-generated and AI-modified content. M2C analysis metadata maps to these properties as follows:

| IPTC 2025.1 Property | M2C Mapping | Notes |
|----------------------|-------------|-------|
| `digitalsourcetype` | `trainedAlgorithmicMedia` or `compositeSynthetic` | Describes the *source media*, not M2C analysis. M2C does not set this field — it is the responsibility of the content creator. |
| `AI System Used` (Iptc4xmpExt:AISystemUsed) | M2C analysis engine identifier | Implementations SHOULD record the M2C module chain (e.g., `M2C/v0.2 scene:pyscenedetect/1.0 summary:llm/gpt-4o`). |
| `AI Training Data` | — | M2C analysis does not constitute training. Not applicable. |

**Important distinction**: IPTC 2025.1 labels whether content was *created or modified* by AI. M2C records that content was *analyzed* by AI. These are orthogonal concerns. A photograph taken by a human (IPTC: `digitalCapture`) MAY have M2C analysis metadata (M2C: `analyzedModules: [visual, color, summary]`).

### 3.3 IPTC Video Metadata Hub

The IPTC Video Metadata Hub (VMH) defines metadata for video assets. M2C fields relate to VMH as follows:

| VMH Property | M2C Field | Notes |
|-------------|-----------|-------|
| `headline` | `summary` (truncated) | VMH headline is short; use first sentence of M2C summary. |
| `description` | `summary` | Full M2C summary. |
| `keyword` | `tags` | Direct mapping. |
| `videoLength` | `duration` | M2C stores seconds; VMH uses `HH:MM:SS:FF`. Conversion required. |
| `personHeard` | `speakers[].label` | Partial — only when label is available. |
| `personShown` | (from `visual` module) | Requires `visual.objectFrequency["person"]` and `pose` module data. |
| `shotType` | (from `visual` module) | Future M2C module (`framing`) could provide this. Not in v0.1. |

---

## 4. C2PA Integration

### 4.1 M2C Analysis as a C2PA Action

When a media file is analyzed by M2C, the analysis event SHOULD be recorded as a [C2PA Action Assertion](https://c2pa.org/specifications/specifications/2.1/specs/C2PA_Specification.html#_actions) in the file's C2PA manifest.

**Action Definition**:

```json
{
  "action": "m2c.analyzed",
  "softwareAgent": "M2C/v0.2",
  "when": "2026-04-01T12:00:00Z",
  "parameters": {
    "m2c.moduleId": ["scene", "transcription", "visual", "color", "summary"],
    "m2c.precisionLevel": "standard",
    "m2c.confidence": 0.92,
    "m2c.schemaVersion": 1
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `m2c.moduleId` | array of strings | MUST | Module IDs that were executed |
| `m2c.precisionLevel` | string | MUST | `quick`, `standard`, or `deep` |
| `m2c.confidence` | number | MUST | Aggregate confidence (0.0–1.0) |
| `m2c.schemaVersion` | integer | MUST | MediaContext schema version |

### 4.2 Custom Assertion: MediaContext Embedding

For applications that need the full analysis result within the C2PA manifest, M2C defines a custom assertion:

**Assertion Label**: `m2c.context`

**Assertion Content**: The complete MediaContext JSON object, CBOR-encoded per C2PA requirements.

```
Assertion label: "m2c.context"
Content type:    application/cbor
Content:         CBOR-encoded MediaContext JSON
```

Implementations MAY embed the full MediaContext or a subset (e.g., L0 or L1 only) depending on size constraints. The `m2c:PrecisionLevel` in the assertion MUST match the depth of the embedded data.

**Size guidance**:
- L0 (tags + mood): ~200 bytes — SHOULD embed
- L1 (summary + tags + palette + speakers): ~1 KB — SHOULD embed
- L2 (+ scenes): ~5–50 KB — MAY embed
- L3 (full): ~50 KB–5 MB — SHOULD NOT embed; use sidecar reference

### 4.3 Provenance Chain Recording

M2C analysis creates a new step in the provenance chain. The relationship between analysis and subsequent edits MUST be preserved:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Original    │     │  M2C         │     │  AI-Assisted  │
│  Media       │────→│  Analysis    │────→│  Edit         │
│              │     │              │     │              │
│  c2pa:       │     │  c2pa:       │     │  c2pa:       │
│  created     │     │  m2c.analyzed│     │  edited      │
│              │     │  m2c.context │     │  (uses M2C   │
│              │     │              │     │   context)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

Each C2PA manifest in the chain MUST include:
1. A `parentOf` relationship to the previous manifest
2. The `m2c.analyzed` action when M2C analysis occurs
3. Subsequent editing actions that MAY reference M2C context data

### 4.4 Training / Data Mining Assertion

C2PA defines assertions for declaring training and data mining permissions. M2C analysis results have specific considerations:

| C2PA Assertion | M2C Relationship |
|---------------|-----------------|
| `c2pa.training-mining` | M2C analysis output (MediaContext) is a *derivative work*. If the original media prohibits training (`c2pa.training-mining.allowed = false`), the M2C analysis result SHOULD carry the same restriction. |
| `c2pa.ai_generated` | M2C analysis output is AI-generated metadata, not AI-generated media. Implementations SHOULD distinguish between these cases. |

**Rule**: When an M2C module analyzes media that has a `c2pa.training-mining` assertion with `allowed = false`, the resulting MediaContext JSON MUST include a `rights` field:

```json
{
  "rights": {
    "trainingMiningAllowed": false,
    "source": "c2pa:inherited"
  }
}
```

---

## 5. Schema.org Mapping

### 5.1 MediaContext to Schema.org Vocabulary

M2C fields map to [Schema.org](https://schema.org/) types and properties, enabling structured data for search engines and knowledge graphs.

| MediaContext Field | Schema.org Type / Property | Notes |
|-------------------|---------------------------|-------|
| (root, `mediaType: "video"`) | `VideoObject` | |
| (root, `mediaType: "image"`) | `ImageObject` | |
| (root, `mediaType: "audio"`) | `AudioObject` | |
| `summary` | `description` | |
| `duration` | `duration` | Convert to ISO 8601 duration (e.g., `PT5M42S`) |
| `tags` | `keywords` | Join as comma-separated string |
| `mood` | `additionalProperty` | Use `PropertyValue` with `name: "mood"` |
| `colorPalette` | `additionalProperty` | Use `PropertyValue` with `name: "colorPalette"` |
| `speakers[].label` | `actor` → `Person.name` | Only when label is a person name |
| `scenes[]` | `hasPart` → `Clip` | Each scene maps to a `Clip` with `startOffset` and `endOffset` |
| `scenes[].summary` | `Clip.description` | |
| `transcription` | `transcript` | Concatenated plain text |
| `confidence` | `additionalProperty` | Use `PropertyValue` with `name: "m2cConfidence"` |
| `analyzedAt` | `dateModified` | The date the analysis was performed |
| `precisionLevel` | `additionalProperty` | Use `PropertyValue` with `name: "m2cPrecisionLevel"` |

### 5.2 JSON-LD Example

```json
{
  "@context": [
    "https://schema.org",
    {
      "m2c": "https://m2c-protocol.org/ns/v0.2/"
    }
  ],
  "@type": "VideoObject",
  "name": "Cooking Vlog — 3 Easy Recipes",
  "description": "Cooking vlog. 3 recipes demonstrated in a bright kitchen.",
  "duration": "PT5M42S",
  "keywords": "cooking, vlog, recipe",
  "dateModified": "2026-04-01T12:00:00Z",
  "actor": [
    {
      "@type": "Person",
      "name": "Host"
    }
  ],
  "hasPart": [
    {
      "@type": "Clip",
      "name": "Scene 1",
      "description": "Opening. Host greets viewers.",
      "startOffset": 0.0,
      "endOffset": 45.2
    },
    {
      "@type": "Clip",
      "name": "Scene 2",
      "description": "Pasta recipe demonstration.",
      "startOffset": 45.2,
      "endOffset": 120.0
    }
  ],
  "additionalProperty": [
    {
      "@type": "PropertyValue",
      "name": "m2cConfidence",
      "value": 0.92
    },
    {
      "@type": "PropertyValue",
      "name": "m2cPrecisionLevel",
      "value": "standard"
    },
    {
      "@type": "PropertyValue",
      "name": "mood",
      "value": "bright, casual"
    }
  ],
  "m2c:schemaVersion": 1,
  "m2c:analyzedModules": ["scene", "transcription", "visual", "color", "summary"]
}
```

### 5.3 Embedding in HTML

Implementations that serve media on the web SHOULD embed M2C-derived Schema.org data in the page:

```html
<script type="application/ld+json">
  { ... JSON-LD from Section 5.2 ... }
</script>
```

This enables search engines to index AI-analyzed media with rich structured data.

---

## 6. MPEG-7 Relationship

### 6.1 Historical Context

MPEG-7 (ISO/IEC 15938), formally known as "Multimedia Content Description Interface," was standardized in 2001 as the definitive framework for describing multimedia content. It defines Description Schemes (DS), Descriptors (D), and a Description Definition Language (DDL).

M2C occupies a conceptually similar space but differs fundamentally:

| Aspect | MPEG-7 | M2C |
|--------|--------|-----|
| **Era** | Pre-AI (2001) | AI-native (2026) |
| **Descriptors** | Hand-crafted feature extractors | AI model outputs |
| **Schema** | XML-based DDL | JSON Schema |
| **Confidence** | Not inherent | First-class concept |
| **Modularity** | Monolithic standard | Independent pluggable modules |
| **Adoption** | Low (too complex) | Designed for simplicity |

### 6.2 Conceptual Mapping

The following table maps MPEG-7 descriptors to M2C equivalents. This is provided as reference for implementors migrating from MPEG-7 systems. No runtime interoperability is defined.

| MPEG-7 Descriptor / DS | M2C Equivalent | Notes |
|------------------------|---------------|-------|
| `ColorLayout` | `color` module → `colorPalette` | MPEG-7 uses DCT coefficients; M2C uses hex color values |
| `DominantColor` | `color` module → `dominantColor` | |
| `ScalableColor` | `color` module → `palette` (full) | |
| `EdgeHistogram` | (no equivalent) | M2C relies on AI captioning, not edge features |
| `AudioSpectrumBasis` | `acoustic` module → `bgmSegments` | MPEG-7 is low-level; M2C is semantic |
| `AudioTempoType` | (future `rhythm` module) | Not in v0.1 |
| `SpokenContent` | `transcription` module | MPEG-7 defined word lattices; M2C uses timestamped segments |
| `FaceRecognition` | `pose` module → `face` | MPEG-7 uses eigenfaces; M2C uses deep learning landmarks |
| `MovingRegion` | `scene` module → scenes | MPEG-7 tracks arbitrary regions; M2C tracks scene boundaries |
| `SemanticBase` | `summary` module | MPEG-7 defined formal semantics; M2C uses LLM-generated natural language |
| `TextAnnotation` | `summary`, `tags` | |

### 6.3 Migration Guidance

Systems migrating from MPEG-7 to M2C SHOULD:

1. Map existing MPEG-7 descriptors to the closest M2C module output
2. Re-analyze media using M2C modules for higher-quality AI-native results
3. Retain MPEG-7 data as archival metadata (do not discard)
4. Use the dual-storage pattern (Section 7) to maintain both representations during transition

---

## 7. Dual Storage Pattern

### 7.1 Architecture

M2C supports two complementary storage locations for analysis results:

```
┌─────────────────────┐     ┌──────────────────────────────┐
│  In-File (XMP)      │     │  Sidecar (JSON)              │
│                     │     │                              │
│  ✓ L0–L1 fields    │     │  ✓ Full MediaContext (L0–L3) │
│  ✓ Travels with    │     │  ✓ No file modification      │
│    the file         │     │  ✓ Unlimited size            │
│  ✗ Size-limited    │     │  ✗ Detachable from media     │
│  ✗ Not all formats │     │  ✗ Requires hash binding     │
└────────┬────────────┘     └──────────────┬───────────────┘
         │                                 │
         └──────────┬──────────────────────┘
                    │
          Content Hash Binding
          (SHA-256 of media file)
```

### 7.2 Sidecar File Convention

The sidecar JSON file MUST follow this naming convention:

```
<original-filename>.m2c.json
```

Examples:
- `cooking-vlog.mp4` → `cooking-vlog.mp4.m2c.json`
- `portrait.jpg` → `portrait.jpg.m2c.json`

The sidecar file MUST include a `source` field for binding:

```json
{
  "source": {
    "filename": "cooking-vlog.mp4",
    "sha256": "a1b2c3d4e5f6...64-char-hex-digest",
    "size": 104857600,
    "mimeType": "video/mp4"
  },
  "schemaVersion": 1,
  "mediaType": "video",
  "summary": "...",
  ...
}
```

### 7.3 Content Hash Binding

The `source.sha256` field MUST be the SHA-256 hash of the original media file at the time of analysis.

**Verification process**:

1. Compute SHA-256 of the media file
2. Compare with `source.sha256` in the sidecar
3. If match: context is valid for this file
4. If mismatch: context is stale — re-analysis SHOULD be triggered

**Edge cases**:

| Scenario | Behavior |
|----------|----------|
| Media file modified after analysis | Hash mismatch → stale context. Flag for re-analysis. |
| Media file renamed | Hash still matches → context is valid. Update `source.filename`. |
| Media file transcoded (e.g., H.264 → H.265) | Hash mismatch → re-analysis required. Some context (summary, tags) MAY be carried over. |
| Sidecar without media file | Orphaned context. Implementations MAY garbage-collect after a configurable period. |

### 7.4 Distribution Metadata Loss

When media files are distributed (uploaded to social platforms, sent via messaging, transcoded by CDNs), embedded XMP metadata is frequently stripped. The dual storage pattern mitigates this:

| Distribution Channel | XMP Preserved? | Mitigation |
|---------------------|---------------|------------|
| Direct file transfer | Yes | No action needed |
| Cloud storage (S3, GCS) | Yes | No action needed |
| Social media upload | Usually No | Re-analyze after download, or maintain server-side sidecar |
| Messaging apps | Usually No | Sidecar file sent alongside media |
| CDN transcoding | No | Store sidecar in origin; re-attach after delivery |
| Email attachment | Usually Yes | No action needed |

**Recommendation**: Implementations MUST always maintain a sidecar JSON as the primary storage. XMP embedding is a convenience layer for interoperability with existing tools.

---

## 8. Compatibility Testing Guidelines

### 8.1 XMP Read/Write Verification

Implementations MUST verify XMP interoperability using [ExifTool](https://exiftool.org/).

**Writing M2C XMP**:

```bash
exiftool \
  -XMP-m2c:SchemaVersion=1 \
  -XMP-m2c:MediaType=video \
  -XMP-m2c:Summary="Cooking vlog. 3 recipes." \
  -XMP-m2c:Mood="bright, casual" \
  -XMP-m2c:Confidence=0.92 \
  -XMP-m2c:PrecisionLevel=standard \
  -XMP-m2c:AnalyzedAt="2026-04-01T12:00:00Z" \
  cooking-vlog.mp4
```

**Reading M2C XMP**:

```bash
exiftool -XMP-m2c:all cooking-vlog.mp4
```

**Expected output**:

```
Schema Version                  : 1
Media Type                      : video
Summary                         : Cooking vlog. 3 recipes.
Mood                            : bright, casual
Confidence                      : 0.92
Precision Level                 : standard
Analyzed At                     : 2026-04-01T12:00:00Z
```

**ExifTool config file** (required for custom namespace):

```perl
# m2c.exiftool.config
%Image::ExifTool::UserDefined = (
    'Image::ExifTool::XMP::Main' => {
        m2c => {
            SubDirectory => {
                TagTable => 'Image::ExifTool::UserDefined::m2c',
            },
        },
    },
);

%Image::ExifTool::UserDefined::m2c = (
    GROUPS => { 0 => 'XMP', 1 => 'XMP-m2c', 2 => 'Other' },
    NAMESPACE => { m2c => 'https://m2c-protocol.org/ns/v0.2/' },
    SchemaVersion => { Writable => 'integer' },
    MediaType     => { Writable => 'string' },
    Summary       => { Writable => 'string' },
    Mood          => { Writable => 'string' },
    Confidence    => { Writable => 'rational64s' },
    PrecisionLevel => { Writable => 'string' },
    AnalyzedAt    => { Writable => 'date' },
    Tags          => { Writable => 'string', List => 'Bag' },
    AnalyzedModules => { Writable => 'string', List => 'Bag' },
    ColorPalette  => { Writable => 'string', List => 'Seq' },
);
```

### 8.2 C2PA Manifest Validation

Implementations that embed M2C data in C2PA manifests MUST verify using the [c2patool](https://github.com/contentauth/c2patool):

**Inspect manifest**:

```bash
c2patool cooking-vlog.mp4 --output manifest.json
```

**Verify M2C action exists**:

```bash
# The manifest.json should contain an action with:
# "action": "m2c.analyzed"
# "parameters" with m2c.moduleId, m2c.confidence, etc.
```

**Verify custom assertion**:

```bash
# Look for assertion with label "m2c.context"
# Decode CBOR content and validate against MediaContext JSON Schema
```

**Validation checklist**:

- [ ] `m2c.analyzed` action is present in the actions assertion
- [ ] All required parameters (`m2c.moduleId`, `m2c.precisionLevel`, `m2c.confidence`, `m2c.schemaVersion`) are present
- [ ] `m2c.context` assertion (if present) decodes to valid MediaContext JSON
- [ ] Decoded MediaContext validates against `schema.json`
- [ ] Provenance chain is intact (parent manifest references are valid)

### 8.3 JSON-LD Validation

Implementations that generate Schema.org JSON-LD from M2C data MUST validate the output:

**Using Google Rich Results Test**:

1. Embed JSON-LD in an HTML page
2. Submit to [Rich Results Test](https://search.google.com/test/rich-results)
3. Verify no errors for `VideoObject` / `ImageObject` / `AudioObject`

**Using JSON-LD Playground**:

1. Paste JSON-LD into [JSON-LD Playground](https://json-ld.org/playground/)
2. Verify the expanded form resolves all `m2c:` prefixed terms
3. Verify `@context` includes both `https://schema.org` and the M2C namespace

**Programmatic validation**:

```bash
# Using jsonld.js CLI or equivalent
jsonld normalize --format application/n-quads input.jsonld
```

The output MUST NOT contain unresolved blank nodes for M2C-specific properties.

### 8.4 Sidecar Integrity Verification

```bash
# 1. Compute media file hash
sha256sum cooking-vlog.mp4
# Output: a1b2c3d4e5f6... cooking-vlog.mp4

# 2. Compare with sidecar
jq -r '.source.sha256' cooking-vlog.mp4.m2c.json
# Output: a1b2c3d4e5f6...

# 3. Validate MediaContext against schema
npx ajv validate -s schema.json -d cooking-vlog.mp4.m2c.json
```

### 8.5 Cross-Standard Consistency

When a media file contains both IPTC metadata and M2C data, implementations SHOULD verify consistency:

| Check | Condition | Severity |
|-------|-----------|----------|
| Keywords match | IPTC `Keywords` ⊇ M2C `tags` | Warning |
| Description present | IPTC `Description` is populated if M2C `summary` exists | Warning |
| AI System recorded | IPTC `AI System Used` includes M2C engine info | Warning |
| C2PA action present | C2PA manifest includes `m2c.analyzed` if M2C sidecar exists | Error |
| Hash binding valid | Sidecar `source.sha256` matches actual file | Error |
| Schema version consistent | XMP `m2c:SchemaVersion` matches sidecar `schemaVersion` | Error |

---

## Appendix A: Namespace Registration

The M2C XMP namespace (`https://m2c-protocol.org/ns/v0.2/`) is not registered with Adobe's XMP namespace registry. Implementations MUST include the namespace declaration in every XMP packet. Registration with Adobe's registry will be pursued when M2C reaches v1.0 stable.

## Appendix B: Future Work

The following interoperability topics are deferred to future versions:

| Topic | Target Version | Description |
|-------|---------------|-------------|
| **Dublin Core mapping** | v0.3 | Formal mapping to DC elements beyond what IPTC covers |
| **EXIF integration** | v0.3 | Reading camera EXIF data as input to M2C analysis modules |
| **EBUCore mapping** | v0.3 | Mapping to European Broadcasting Union metadata standard |
| **ActivityPub / Fediverse** | v0.4 | Distributing M2C context via ActivityPub `attachment` |
| **W3C Web Annotation** | v0.4 | Representing scene-level annotations as W3C Web Annotations |
| **IIIF Presentation API** | v0.4 | Mapping M2C scenes to IIIF Canvas / Annotation structures |
| **Adobe XMP SDK integration** | v1.0 | Native read/write support in Adobe's XMP SDK |

## Appendix C: Reference Table — Field Coverage Across Standards

| MediaContext Field | XMP (M2C) | IPTC | C2PA | Schema.org | MPEG-7 |
|-------------------|-----------|------|------|------------|--------|
| `schemaVersion` | `m2c:SchemaVersion` | — | `m2c.schemaVersion` param | `m2c:schemaVersion` | — |
| `mediaType` | `m2c:MediaType` | (implicit) | — | `@type` | — |
| `summary` | `m2c:Summary` | `dc:description` | `m2c.context` assertion | `description` | `TextAnnotation` |
| `tags` | `m2c:Tags` | `dc:subject` | `m2c.context` assertion | `keywords` | `TextAnnotation` |
| `mood` | `m2c:Mood` | — | `m2c.context` assertion | `additionalProperty` | — |
| `colorPalette` | `m2c:ColorPalette` | — | `m2c.context` assertion | `additionalProperty` | `DominantColor` |
| `duration` | `m2c:Duration` | VMH:`videoLength` | — | `duration` | — |
| `speakers` | `m2c:Speakers` | `PersonInImage` (partial) | `m2c.context` assertion | `actor` | `SpokenContent` |
| `scenes` | `m2c:Scenes` | — | `m2c.context` assertion | `hasPart` → `Clip` | `MovingRegion` |
| `transcription` | (sidecar only) | — | `m2c.context` assertion | `transcript` | `SpokenContent` |
| `acoustic` | (sidecar only) | — | `m2c.context` assertion | — | `AudioSpectrumBasis` |
| `visual` | (sidecar only) | — | `m2c.context` assertion | — | `ColorLayout` |
| `style` | (sidecar only) | — | `m2c.context` assertion | — | — |
| `confidence` | `m2c:Confidence` | — | `m2c.confidence` param | `additionalProperty` | — |
| `analyzedModules` | `m2c:AnalyzedModules` | — | `m2c.moduleId` param | `m2c:analyzedModules` | — |
| `precisionLevel` | `m2c:PrecisionLevel` | — | `m2c.precisionLevel` param | `additionalProperty` | — |
| `analyzedAt` | `m2c:AnalyzedAt` | — | `when` (action) | `dateModified` | — |
