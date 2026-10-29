# cvstudio

Self-hosted AI CV & cover letter generator. Paste or upload a job role and get
a tailored resume and cover letter PDF, on the fly. Built from facts in your
own profile (a Google Sheet) — nothing is invented.

The project started as the CLI pipeline `sergioruiz_cv_ATS`. This repo wraps
that pipeline in a small web dashboard and keeps the files organized the way
Dossier (the shared repo) does — minus all the job-application tracking.

## What it does

- **One click** — paste text, upload a `.pdf`/`.txt`, or give a posting URL.
- **Two LLM passes** always tailor the resume to the role (rewrite + section filter).
- **Cover letter** written to match, rendered with the same fonts.
- **Downloads** — CV + cover letter PDFs, previewed inline.
- **History** — a simple list of past documents. Not a tracker.
- **Profile from Google Sheets** — your sheet stays the single source of truth.

## Quickstart (local)

```bash
cd cvstudio
python -m venv .venv
.venv\Scripts\activate          # Windows (bash: source .venv/bin/activate)
pip install -r requirements.txt -e .
cp .env.example .env            # set CVSTUDIO_PASSWORD + your LLM key
uvicorn app:app --reload        # → http://localhost:8000
```

Or with [uv](https://docs.astral.sh/uv/):

```bash
uv venv
uv pip install -r requirements.txt -e .
uv run uvicorn app:app --reload
```

Sign in with `CVSTUDIO_PASSWORD`. If you did not set one, a generated password
is printed at startup (set `CVSTUDIO_PASSWORD` in `.env`).

## Using it

1. **Profile** → *Refresh from Google Sheets* pulls `career_full.json` from
   your sheet. The sheet must be shared with "anyone with the link".
2. **New CV** → paste/upload/URL. The app reads the posting and asks you to
   confirm the job title and company. *Generate* then builds both PDFs.
3. **Documents** → every previous pair, with CV/letter downloads.

Published files are named `[NAME]_[COMPANY]_[CODE]_YYMM_CV.pdf` and
`..._Cover.pdf`, for example
`Sergio_Ruiz_Carmona_Veeva_Systems_SRWEDA_2610_CV.pdf`. `CODE` is the
initials of the role title. A repeat run in the same month gets a `_2`
suffix, so nothing is overwritten.

The resume is capped at 2 pages. When it overflows, content is cut in this
order: one bullet on the two oldest roles, the oldest role, presentations,
publications, teaching, then a compact layout. Dates print as `Apr 2024` when
the month is known and as `2012` when only the year is known.

## Directory layout

```
cvstudio/
├─ app.py                  # FastAPI entry (uvicorn app:app)
├─ src/cvstudio/
│  ├─ pipeline/            # build.py, letter.py, llm.py, analyze.py, converter.py, orchestrator.py
│  ├─ web/                 # app.py (factory), routes.py, jobs.py, auth.py
│  ├─ store.py             # SQLite: settings + history
│  ├─ templates/           # Jinja2 pages
│  └─ static/              # css/js (Dossier-inspired design)
├─ data/                   # career_data.xlsx, career_full.json
├─ job_roles/              # job postings you've generated from
├─ outputs/                # generated PDFs
├─ runs/                   # per-run intermediates (yaml, typ, raw pdfs)
└─ tools/cli.py            # terminal pipeline (run.sh replacement)
```

## Settings

| Setting | Meaning |
|---|---|
| `llm_provider` / `llm_model` | writing model: `ollama` + `gemma4:31b` by default |
| `llm_api_key` | key for the writing provider |
| `llm_fallback_*` | tried only when the writing model fails |
| `sheet_id` | Google Sheets ID for your profile |

The writing model is picked from a short, curated list in Settings. No free
text, so an expensive model cannot slip in. The role reader is fixed:
`google/gemma-4-31b-it` on OpenRouter, which needs `OPENROUTER_API_KEY`.

Provider presets come from env keys (`OPENROUTER_API_KEY`,
`OLLAMA_API_KEY`). API keys live in `.env` or Settings — only the server sees them.
The `custom` option still accepts any OpenAI-compatible endpoint.

## CLI (optional)

```bash
python tools/cli.py --job job_roles/rwe_data_analyst_sanofi.pdf
python tools/cli.py --job ... --skip-llm      # no LLM call
python tools/cli.py --posting "…" --refresh
```

## Deploy on a VPS (Oracle free tier works)

```bash
# on the server
git clone <your repo> cvstudio && cd cvstudio
cp .env.example .env        # set password, keys, sheet id
docker compose up -d        # → http://SERVER_IP:8000
```

Put HTTPS in front with Caddy (same machine, your domain):

```
cv.example.com {
    reverse_proxy 127.0.0.1:8000
    encode gzip
}
```

Set `CVSTUDIO_HTTPS=1` in `.env` once TLS is on. Persistent data lives in
`data/ job_roles/ outputs/ runs/`, which the compose file binds as volumes.

## Secrets

`.env` holds the password and the API keys. It is gitignored: never commit it.

Scan before the first push and before each release:

```bash
gitleaks git --redact --verbose
gitleaks dir . --redact --verbose
```

`.gitleaks.toml` extends the default rules and skips local-only folders. Setup
notes, install commands and an optional pre-commit hook are in
[`docs/NOTES.md`](docs/NOTES.md).
