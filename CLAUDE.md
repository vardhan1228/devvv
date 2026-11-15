# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Layout

This directory is not a git repo itself. It holds:

- `app.py` — a single-file proof of concept: an AI email-classification "harness" built on Google Gemini.
- `FP/` — a separate git repo of Python practice Jupyter notebooks (`*.ipynb`), unrelated to `app.py`.

## Running app.py

No requirements file exists. Dependencies are `google-genai` and `python-dotenv`:

```
pip install google-genai python-dotenv
python app.py
```

- `GEMINI_API_KEY` must be set in `.env` (loaded via `load_dotenv()`); the script raises at import time if it is missing. Never print or log its value.
- `app.py` starts an interactive menu (`input()`), so it can't run unattended. To run the evaluation suite non-interactively:
  `python -c "import app; app.evaluate_agent()"`
- Classify a single email: `python -c "import app, json; print(json.dumps(app.run_harness('...'), indent=2))"`
- There is no test framework; `evaluate_agent()` is the test suite (5 hardcoded cases that make real API calls to Gemini).

## Architecture of app.py

`run_harness(email)` is the orchestrator, and the pipeline is deliberately split into deterministic steps around one non-deterministic model call:

1. `validate_input` — type/empty/length check (`MAX_EMAIL_LENGTH`) before any API call.
2. `classify_with_ai` — calls `client.models.generate_content` with `temperature: 0` and `response_mime_type: application/json`; the email is wrapped in `<email>` tags and the prompt treats it as untrusted data.
3. `validate_output` — runs **inside** the retry loop, so malformed model output triggers a retry (up to `MAX_ATTEMPTS`, sleeping `attempt` seconds between tries).
4. `apply_policy` — a deterministic phrase-match guardrail that runs after the model and can override its answer with `REVIEW_REQUIRED` (keeping `original_category`).
5. `log_event` — appends JSON Lines to `harness.log`, relative to the current working directory.

Non-obvious points:

- `REVIEW_REQUIRED` is intentionally **not** in `ALLOWED_LABELS`. The model may only return IMPORTANT/NORMAL/SPAM; `REVIEW_REQUIRED` comes only from `apply_policy`. Failures return category `ERROR`.
- `run_harness` never raises. It always returns a dict with `status` of `success` or `failed`.
- Logs must not contain the full email text or the API key; only `input_length`, category, attempts and timing are logged.
- Use the `google-genai` SDK style (`client.models.generate_content`). Older entries in `harness.log` show `'Client' object has no attribute 'chat'` errors from OpenAI-style calls.
