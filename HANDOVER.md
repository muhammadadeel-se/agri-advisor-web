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
* **Offline Demo Fallback (`DemoFallbackClient` — Non-Production Only):** When `OPENAI_API_KEY` is omitted, `app.py` routes completions to `DemoFallbackClient`, which uses deterministic regex extraction instead of an LLM. While its tool calls still pass through `jsonschema` validation and `validate_fertilizer_dosage()`, maintainers should note its exact input boundaries:
  * **Inputs it handles:**
    1. *Weather lookups:* Single-turn prompts containing both a **5-digit US ZIP code** (`\b\d{5}\b`) and at least one weather keyword (`weather`, `rain`, `forecast`, `moisture`, `temperature`, `wind`, `zip`), with optional `metric` / `celsius` unit detection.
    2. *Fertilizer calculations:* Single-turn prompts naming a supported crop (`corn`, `wheat`, `soybean`, `tomato`, `potato`) **plus** explicit numeric values for all three macronutrients (`N=...`, `P=...`, `K=...` in ppm), with optional `acres` (defaults to `1.0`) and `pH` (defaults to `6.5`).
  * **Inputs it drops to a generic or canned reply:**
    1. *Multi-turn follow-ups:* Because `DemoFallbackClient` inspects only the latest message (`messages[-1]`), context-dependent follow-ups (e.g., *"Can I apply that today?"* or *"Recalculate for 50 acres"*) ignore conversation history and drop to the generic capabilities prompt.
    2. *Missing or natural-language parameters:* Naming a crop without all three explicit `N`, `P`, and `K` tags returns a canned format prompt; giving a ZIP code without a weather keyword (or using a city name instead of a ZIP) drops to the generic capabilities message.
    3. *Compound & open-ended agronomy questions:* Asking for weather and fertilizer in one message executes only the weather lookup and ignores the fertilizer request; general farming questions (pH-only lime advice, planting dates, irrigation) drop to the generic message.
    4. *Disabled semantic classifier:* Layer 2 (`LLMSemanticClassifier`) does not run when no API key is configured, leaving only Layer 1 deterministic regex/keyword filtering active.
  * **Production requirement:** Co-op administrators must set a valid `OPENAI_API_KEY` before exposing the app to farmers.
* **Simulated Weather Data:** `get_simulated_weather` generates deterministic regional weather baselines from ZIP code hashes rather than live meteorological feeds. Before production field spraying, integrate a live weather API (e.g., NOAA/NWS) inside `agri_tools.py`.
* **Simplified Soil Conversion:** Soil ppm-to-lbs/acre conversion uses a standard `2.0x` topsoil depth factor and does not account for soil organic matter, cation exchange capacity (CEC), or legume nitrogen credits.
* **Ephemeral Container Logs:** On free cloud tiers, `guardrail_events.log` resets when the container restarts. For long-term compliance retention, forward `SecurityAuditLogger` events to a persistent database or log sink.

### Routine Maintenance Tasks
1. **Updating Crop Nutrient Targets:** Edit `_CROP_PROFILES` in `agri_tools.py` and add the crop name to `TOOL_SCHEMAS` and `_AGRI_DOMAIN_KEYWORDS` in `guardrails.py`.
2. **Calibrating Classifier Confidence Threshold:** Inspect `confidence` values for `"verdict": "allowed"` vs `"blocked"` entries in `guardrail_events.log`, then adjust `DEFAULT_CONFIDENCE_THRESHOLD` (currently `0.85`) in `guardrails.py`.
3. **Pre-Deployment Regression Verification:** Always run `pytest -v` after any prompt, schema, or guardrail modification before pushing to `main`.