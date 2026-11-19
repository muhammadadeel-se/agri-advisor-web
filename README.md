# GreenField Farmers Co-op — Guarded Agricultural Advisory Web App

A Streamlit web application wrapping a schema-validated, tool-calling agricultural advisor with multi-layer input/output safety guardrails, audit logging, and offline demo fallback support.

## Live Demo
* **Public App URL:** https://agri-advisor-web-dsyjbsntvzqo679pwtwgza.streamlit.app/
* **Client Handover Documentation:** See [`HANDOVER.md`](HANDOVER.md) for architecture, token cost estimates, safe agronomic limits, and maintenance procedures.

## Key Features
* **Farmer Chat Web UI (`app.py`):** Interactive Streamlit chat interface with one-click test prompts, expandable JSON tool-call traces, live guardrail audit log viewer, and automatic **Offline Demo Mode** when no API key is supplied.
* **Two-Layer Input Guardrails (`guardrails.py`):** Sanitization + deterministic regex/keyword filtering (zero token cost) paired with an XML-delimited `LLMSemanticClassifier` (`<untrusted_user_input>`) and fail-closed defaults.
* **Post-Tool Dosage Ceilings (`guardrails.py` & `agent_loop.py`):** Blocks fertilizer outputs exceeding safe agronomic limits (`N ≤ 350`, `P2O5 ≤ 250`, `K2O ≤ 400 lbs/acre`, `≤ 10,000 acres`).
* **Deterministic Agronomy Tools (`agri_tools.py`):** JSON-Schema validated simulated weather lookup (`get_simulated_weather`) and N-P-K fertilizer calculator (`calculate_fertilizer_recommendation`).

## Local Setup & Testing

```bash
pip install -r requirements.txt
pytest -v
streamlit run app.py
```