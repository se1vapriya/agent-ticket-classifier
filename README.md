# agent-ticket-classifier
# Run & Debug — Support Ticket Classifier

## 1. Set up

```powershell
cd "O:\ticket -classification"
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

## 2. Add your API key

Edit `.env` — it currently holds the placeholder `your_api_key_here`:

```
OPENAI_API_KEY=sk-...your real key...
PROMPT_VERSION=v2
DEFAULT_MODEL=gpt-4o-mini
LOG_COSTS=true
```

Without a real key the injection guard fails-safe and **every** ticket comes back
`injection_blocked=true` — that is by design, not a crash.

## 3. Run

```powershell
python main.py          # or: uvicorn main:app --reload --port 8000
```

- Health:  http://localhost:8000/health
- Docs:    http://localhost:8000/docs
- Demo UI: open `demo_ui/index.html` in a browser (it calls `http://localhost:8000/classify`)

## 4. Tests

```powershell
pytest tests/test_classifier.py -v
```

---

## Bugs fixed

| File | Problem | Fix |
|---|---|---|
| `production_modules/fallback_retry.py` | `ValidationError.from_exception_data(title=..., input_type=..., input=...)` is an invalid signature (needs `line_errors`). It raised `TypeError`, which wasn't in the retry list, so **retries never ran** — every bad response went straight to `SAFE_CLASSIFICATION`. | Added `ClassificationValidationError`, raised it instead, and added it to `retry_if_exception_type`. |
| `production_modules/fallback_retry.py` | `classify_with_retry.statistics` crashes with `AttributeError` if the attribute isn't set yet. | Guarded with `getattr(..., {})`. |
| `production_modules/structured_output.py` | `ChatOpenAI(model_kwargs={"response_format": ...})` — current `langchain-openai` rejects this: *"Parameters {'response_format'} should be specified explicitly."* JSON mode was fully broken. | Switched to `.bind(response_format={"type": "json_object"})`. |
| `production_modules/structured_output.py` | Stray `print(json.dumps(raw, indent=4))` on every call. | Removed. |
| `graph.py` | Stray `print(result)` in `pii_redact_node`. | Changed to `logger.debug`. |
| `tests/test_classifier.py` (test 1) | Patched `graph.classify_node`, but LangGraph compiles at import time and holds the original function object — the patch had no effect, so the test made a **real** LLM call. It also left `check_injection` unmocked, so the guard failed-safe and the ticket came back `blocked`. | Patched `graph.classify_with_json_mode` and `graph.check_injection` (looked up at call time). |
| `requirements.txt` | `openai` and `langchain-core` imported directly but only pulled in transitively. | Added explicitly. |

## Notes (not bugs)

- `schema.py::TicketState` is unused — the graph runs on a plain `dict`. Harmless, but dead code.
- `demo_ui/index.html` hardcodes `http://localhost:8000`; change it if you run on another port.
