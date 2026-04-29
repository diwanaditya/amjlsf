# AMJLSF Architecture

## Pipeline Overview

AMJLSF operates as a four-stage black-box pipeline. "Black-box" means the framework never accesses model weights, activations, gradients, or internal logits — only the text output (and optionally token probabilities where the API exposes them).

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          AMJLSF v3.0 Pipeline                            │
│                                                                          │
│  ┌─────────────────┐    ┌──────────────────────┐    ┌────────────────┐  │
│  │   INPUT MODULE  │───►│  QUESTION DELIVERY   │───►│   RESPONSE     │  │
│  │                 │    │      MODULE           │    │   ANALYSIS     │  │
│  │ • Model ID      │    │                      │    │   MODULE       │  │
│  │ • API Key       │    │ • Load question set  │    │                │  │
│  │ • Endpoint URL  │    │ • Stratified sample  │    │ • LlamaGuard-2 │  │
│  │ • Domain tags   │    │ • Randomized order   │    │   (jailbreak)  │  │
│  │ • Scan level    │    │ • Benign controls    │    │ • Cosine sim.  │  │
│  │ • Regulations   │    │   interleaved        │    │   (hallucin.)  │  │
│  └─────────────────┘    └──────────────────────┘    │ • Rule-based   │  │
│                                                      │   (compliance) │  │
│                                                      │ • Latency      │  │
│                                                      │   capture      │  │
│                                                      └───────┬────────┘  │
│                                                              │           │
│                          ┌───────────────────────────────────┘           │
│                          ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                  SCORING & CERTIFICATION MODULE                  │   │
│  │                                                                  │   │
│  │  PRTS → DS_k → CRS  ·  BCRS  ·  SDI  ·  LRS  ·  RCGS → CES    │   │
│  │                                                                  │   │
│  │  Output: CRS Risk Report  +  Signed Certificate  +  JSON/CSV    │   │
│  └──────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Gradio UI Architecture (Cell 4)

The UI is built with Gradio 4.x and structured as a two-column layout:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  AMJLSF v3.0 ── Autonomous Multi-Jurisdictional LLM Security Framework  │
├──────────────────────┬──────────────────────────────────────────────────┤
│   LEFT PANEL         │   RIGHT PANEL (Tabbed)                           │
│   (Controls)         │                                                  │
│                      │  ┌─────────┬───────────┬──────────┬──────────┐  │
│  • API Key input      │  │ Results │ Dashboard │ Analysis │ History  │  │
│    (provider hint)   │  └─────────┴───────────┴──────────┴──────────┘  │
│                      │                                                  │
│  • Scan level radio  │  Results tab:                                    │
│    Micro/Low/Med/High│    Live status card (streaming)                  │
│                      │    Live trace table (per-question)               │
│  • Regulation        │    Final CRS report (HTML)                      │
│    checkboxes        │                                                  │
│                      │  Dashboard tab:                                  │
│  • JSON uploader     │    Radar chart (5 dimensions)                   │
│    (optional)        │    CRS gauge                                     │
│                      │    Score breakdown table                         │
│  • START AUDIT       │                                                  │
│  • RESET             │  Analysis tab:                                   │
│                      │    Per-question heatmap                          │
│  • Scan size info    │    Severity distribution                         │
│    panel             │    Dimension correlation                         │
│                      │                                                  │
│  • Provider legend   │  History tab:                                   │
│                      │    SDI trend (requires ≥2 runs)                 │
│                      │                                                  │
│                      │  Reports tab:                                    │
│                      │    AI summary (auto-generated)                  │
│                      │    JSON download                                 │
│                      │    CSV download                                  │
└──────────────────────┴──────────────────────────────────────────────────┘
```

---

## Model Loading Strategy

### Local Mode (no API key)
Loads `mistralai/Mistral-7B-Instruct-v0.2` via HuggingFace Transformers with 4-bit quantization (bitsandbytes). Requires a T4 GPU (Colab) or equivalent. Lazy-loaded on first audit run to avoid startup delays.

### API Mode (key provided)
Provider auto-detected from key prefix:
```
sk-ant-  → Anthropic (claude-haiku-20240307)
sk-      → OpenAI (gpt-4o-mini)
gsk_     → Groq (llama3-70b-8192)
other    → Together AI (Mixtral-8x7B)
```

All API calls use streaming-compatible endpoints with a configurable timeout.

---

## Dataset Loading

`audit_questions.json` is resolved in priority order:

1. File uploaded via the Gradio file picker (overrides all)
2. Google Drive path `/content/drive/MyDrive/audit_questions.json`
3. Fallback built-in sample set (20 questions) — clearly labelled in output

---

## Keep-Alive Mechanism

A background daemon thread pings `localhost:7861` every 45 seconds to prevent Google Colab from disconnecting during long audits. This is especially important for Medium (50Q) and High (100Q) scan levels.

---

## Certificate Generation

At the end of each successful scan, AMJLSF generates a certificate dictionary:

```python
{
    "model_id": "<model identifier>",
    "evaluation_date": "<ISO 8601 UTC>",
    "dimension_scores": { "J": 0.12, "H": 0.21, "I": 0.08, "S": 0.15, "C": 0.09 },
    "crs": 0.14,
    "bcrs": 0.05,
    "jurisdictions_assessed": ["EU AI Act", "India DPDP"],
    "certification_level": "FULL",
    "valid_until": "<ISO 8601 UTC, +90 days>",
    "sha256": "<SHA-256 hash of the full result object>"
}
```

The `sha256` field is computed over a canonical JSON serialization of all score fields, providing cryptographic verifiability aligned with EU AI Act Article 12 traceability requirements.
