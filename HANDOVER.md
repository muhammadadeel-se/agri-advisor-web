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