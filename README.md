# Resume Analyzer & Builder (v3)

An easy-to-run, local web app that uses a Groq/OpenAI-compatible model to analyze and improve resumes.

This repository contains a small Flask backend that accepts PDF/DOCX resumes, extracts text, calls an OpenAI-compatible AI (Groq) to produce a structured analysis, and a minimal frontend (static HTML/JS/CSS) that uploads resumes and displays the suggestions.

Key points:

- Backend: `backend/app.py` (Flask)
- Frontend: `frontend/` (static assets served by the Flask app)
- AI: Uses an OpenAI-compatible client to call Groq (requires `GROQ_API_KEY`)
- Upload limits: 5 MB, allowed types: `.pdf`, `.docx`

## Quickstart (Windows PowerShell)

These instructions assume you are on Windows and using PowerShell (the repository already includes helpful notes in `backend/installRequirements.txt`).

1. Create and activate a virtual environment, then install dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install --upgrade pip
pip install -r .\backend\requirements.txt
# Install the spaCy English model (required at runtime)
python -m spacy download en_core_web_sm
```

2. Configure your AI key (one of these options):

- Create a `.env` file in the repo root or in `backend/` with:

```text
GROQ_API_KEY=your_groq_api_key_here
# Optional: override model
GROQ_MODEL=llama-3.3-70b-versatile
```

- Or create `backend/local_settings.py` (development only) with:

```python
GROQ_API_KEY = "your_groq_api_key_here"
```

Important: never commit real secrets to source control. Use `.gitignore` to keep keys out of the repo.

3. Run the app:

```powershell
python .\backend\app.py
```

4. Open your browser to http://localhost:5000 — the Flask app serves the static frontend automatically.

## What the backend provides (API contract)

Base URL: http://localhost:5000

- GET /health

  - Returns service health and some configuration flags.
  - Example response keys: `status`, `has_groq_key`, `model`, `spaCy_model_loaded`, `max_upload_mb`

- GET /debug/config

  - Returns helpful (non-secret) diagnostic info about environment detection and paths.

- POST /upload

  - Purpose: Upload a resume file (PDF or DOCX). The server extracts text, calls the AI, and returns structured analysis.
  - Request: multipart/form-data with field `resume` (file). Max file size ~5 MB.
  - Success response JSON (successful analysis):
    - `ai_rating` (string: 1-10)
    - `ai_suggestions` (string: detailed bullet suggestions)
    - `ai_example` (string: legacy field with improved bullets/summary)
    - `keyword_gaps` (string: comma-separated keywords)
    - `improved_summary` (string)
    - `improved_bullet_examples` (string)
    - `priority_fix_order` (string)
    - `raw_ai_output` (string: full unparsed AI reply)
  - Errors: JSON with `error` key and HTTP status codes (400, 500, etc.).

- POST /analyze
  - Purpose: Send plain resume text to receive a concise suggestion + summary.
  - Request JSON: `{ "resume_text": "..." }`
  - Response JSON: `{ "ai_suggestions": "...", "model": "..." }` or `{ "error": "..." }`

Notes: The `/upload` endpoint performs text extraction (pdfminer for PDFs, python-docx for DOCX) and parsing of the AI reply into sections; however, the `raw_ai_output` is provided if parsing is incomplete.

## Frontend

The minimal frontend is in `frontend/`:

- `index.html`, `script.js`, `styles.css` provide a drag/drop file-picker and show the AI analysis.
- The Flask backend serves the frontend at `/` so once `backend/app.py` is running, visit http://localhost:5000.

If you prefer to serve the frontend separately (not necessary), you can run a static file server from the `frontend` folder.

## Configuration and environment

Primary environment variables:

- `GROQ_API_KEY` — required. The app will exit if not configured. (Also the code checks alias names like `GROQ_KEY`, `GROQ_APIKEY`, etc.)
- `GROQ_MODEL` — optional. Defaults to `llama-3.3-70b-versatile` in code.

Where to put the key (choose one):

- Repository root (create `.env`)
- `backend/.env`
- `backend/local_settings.py` (development only)
- `backend/groq_key.txt` (convenience file; the repo ignores this file in .gitignore patterns)

## Troubleshooting

- The app prints helpful diagnostics if `GROQ_API_KEY` is missing, and will not start until a key is provided. Follow the printed suggestions (set env var, create `.env` or `local_settings.py`).
- If spaCy fails to load `en_core_web_sm`, run: `python -m spacy download en_core_web_sm`.
- If file uploads fail, confirm file is under 5 MB and one of `.pdf` or `.docx`.
- For debugging, use `/health` and `/debug/config` endpoints.

Common errors and quick fixes:

- "GROQ_API_KEY is not configured": set the environment variable or create `backend/local_settings.py` as shown above.
- "Could not extract text from resume": try a different resume file or ensure the file is not encrypted/corrupted.

## Developer notes

- AI call implementation is in `backend/_call_groq` (uses OpenAI-compatible `OpenAI` client). The code retries on transient failures.
- Response parsing is implemented with regex heuristics in `backend/app.py` (functions: `parse_ai_analysis`, `_clean_section`). Adjust these if you change the prompt format.
- The prompt used for `/upload` is intentionally prescriptive: the AI is asked to return clearly labeled sections so parsing is more reliable. If you modify the prompt, update the parsing regexes accordingly.

## Tests & Quality Gates

This project does not include an automated test suite in the repo. Quick manual checks:

- Start the server and verify `/health` returns `status: ok`.
- Upload a valid PDF via the frontend or with `Invoke-RestMethod` and confirm a JSON response with `ai_suggestions` is returned.

If you want, I can scaffold a small pytest-based unit test for the parsing logic (`parse_ai_analysis`) and a tiny integration script that posts a sample resume text to `/analyze`.

## Security

- Do not commit API keys. Use environment variables or local-only files for development.
- The app prints masked key info for debugging only and refuses to run without a key to prevent confusing runtime errors in AI routes.

## Contributing

Contributions welcome. For small fixes, open a pull request. If you'd like me to add tests, CI, or Docker support, tell me which you'd prefer and I can implement it.
