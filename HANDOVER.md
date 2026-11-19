# Client Handover Manual — GreenField Farmers Co-op Advisory Agent

## 1. System Architecture

The GreenField Farmers Co-op Advisory Agent is a schema-validated, guardrail-protected web application built in Python and Streamlit. Every farmer prompt passes through deterministic and semantic safety gates before invoking any LLM or agricultural tool:

```text
[Farmer Browser (Streamlit UI - app.py)]
                 │
                 ▼
[Layer 1: Sanitization & Regex/Keyword Gate (guardrails.py)]
                 │  ├── Blocks gibberish, prompt injections, hazardous mixes,
                 │  │   extreme dosage requests & out-of-scope queries (0 tokens)
                 ▼
[Layer 2: Isolated LLM Semantic Classifier (guardrails.py)]
                 │  ├── Wraps text in <untrusted_user_input> XML delimiters
                 │  ├── Enforces JSON schema & confidence threshold (>= 0.85)
                 ▼
[Layer 3: Guarded Agentic Loop (agent_loop.py)]
                 │  ├── Calls OpenAI Chat Completions (or DemoFallbackClient)
                 │  ├── Validates tool arguments against JSON Schema (agri_tools.py)
                 ▼
[Layer 4: Post-Tool Dosage Ceiling Gate (guardrails.py)]
                 │  ├── Rejects N > 350, P2O5 > 250, K2O > 400 lbs/acre or > 10,000 acres
                 ▼
[Structured JSON Audit Log (guardrail_events.log) + UI Response]
## 5. Known Limitations & Maintenance Runbook

### Known Limitations
* **Offline Demo Fallback (`DemoFallbackClient` — Non-Production Only):** When `OPENAI_API_KEY` is omitted, `app.py` falls back to `DemoFallbackClient`, which uses deterministic regex extraction rather than an LLM. While its tool calls still pass through `jsonschema` validation and `validate_fertilizer_dosage()`, it cannot handle nuanced multi-turn follow-ups, synthesize custom agronomic advice, or execute the secondary `LLMSemanticClassifier` (Layer 2). Co-op administrators must configure a valid `OPENAI_API_KEY` before exposing the app to farmers.
* **Simulated Weather Data:** `get_simulated_weather` generates deterministic regional weather baselines from ZIP code hashes rather than live meteorological feeds. Before production field spraying, integrate a live weather API (e.g., NOAA/NWS) inside `agri_tools.py`.
* **Simplified Soil Conversion:** Soil ppm-to-lbs/acre conversion uses a standard `2.0x` topsoil depth factor and does not account for soil organic matter, cation exchange capacity (CEC), or legume nitrogen credits.
* **Ephemeral Container Logs:** On free cloud tiers, `guardrail_events.log` resets when the container restarts. For long-term compliance retention, forward `SecurityAuditLogger` events to a persistent database or log sink.

### Routine Maintenance Tasks
1. **Updating Crop Nutrient Targets:** Edit `_CROP_PROFILES` in `agri_tools.py` and add the crop name to `TOOL_SCHEMAS` and `_AGRI_DOMAIN_KEYWORDS` in `guardrails.py`.
2. **Calibrating Classifier Confidence Threshold:** Inspect `confidence` values for `"verdict": "allowed"` vs `"blocked"` entries in `guardrail_events.log`, then adjust `DEFAULT_CONFIDENCE_THRESHOLD` (currently `0.85`) in `guardrails.py`.
3. **Pre-Deployment Regression Verification:** Always run `pytest -v` after any prompt, schema, or guardrail modification before pushing to `main`.