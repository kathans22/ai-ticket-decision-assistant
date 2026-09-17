# Project: AI Support-Ticket Decision Assistant (take-home assignment)

## What this is
Users register/login (JWT), submit a support ticket, and get an evidence-backed decision
(action, confidence, reason, sources) produced by Gemini using RAG over knowledge_base/*.md.
Tickets and decisions are stored in SQLite. Streamlit talks to FastAPI ONLY over HTTP.
The brief is in docs/ASSIGNMENT.md. Data facts are in docs/DATA_ANALYSIS.md. Read them when relevant.

## Stack (do not change without asking)
Python 3.12 · FastAPI · Uvicorn · SQLAlchemy 2.0 (typed Mapped style, sync) · Pydantic v2 · pydantic-settings
PyJWT · pwdlib[argon2] · google-genai SDK · NumPy · Streamlit · requests · pytest · httpx
NOT allowed: LangChain, LlamaIndex, FAISS/Chroma/Pinecone or any vector DB, Docker, async DB drivers,
Alembic, React, passlib.

## Layout
src/config.py settings · src/database.py engine/session · src/models.py ORM · src/schemas.py API models
src/auth.py hashing + JWT · src/api.py FastAPI app · src/embeddings.py · src/ingest.py · src/retrieval.py
src/decision.py LLM decision + guardrails · src/services.py ticket workflow · streamlit_app.py
scripts/ (run as modules: python -m scripts.<name>) · tests/ · docs/

## Domain facts (filled in after data discovery. Exact strings only.)
- Allowed actions: <PASTE EXACT LIST FROM docs/DATA_ANALYSIS.md>
- Insufficient-info action: <EXACT STRING>
- Knowledge base files: cancellations.md, damaged_goods.md, defective_products.md, returns.md, shipping.md, wrong_item.md

## Coding rules
- Type hints everywhere. Small, single-purpose functions. Module docstring on each src file.
- Use pathlib for paths. Use logging (not print) inside src/. CLIs may print.
- Read config only via src/config.py get_settings().
- Gemini and embedding calls go through injectable interfaces so tests can use fakes.
- API endpoints that call Gemini are plain `def` (sync) so FastAPI runs them in a threadpool.
- Errors: consistent JSON {"detail": ...}. Correct status codes (201, 401, 404, 409, 422, 503).

## Tests
- All pytest tests run OFFLINE with fake LLM/embedder and a temp SQLite DB. No network, no real key.
- Every code unit ships with its tests in the same commit.

## Security
- Never read, print, or commit .env or any key/secret. Never hardcode secrets. .env.example has placeholders only.
- Never store plaintext passwords. Never log passwords or tokens.

## Accuracy / integrity
- NEVER hardcode test-case text, IDs, or keyword→action shortcuts copied from data/sample_test_cases.json.
- Never invent numbers in docs (accuracy, counts). Use only real command output.

## Workflow rules
- Stay strictly inside the current task's scope. Before editing, list the files you will create/modify.
- Do not refactor or reformat unrelated code. Do not add dependencies without stating why.
- If a requirement is ambiguous, stop and ask. Don't guess.
- Never write or edit source files through shell heredocs, echo, or redirection. Use the editor's file tools.
- Environment is Windows (cmd/PowerShell). Give Windows commands. Virtualenv is .venv.

## GIT COMMIT PROTOCOL (MANDATORY)
Each task ends with a numbered list of COMMIT UNITS (message + files). Work through them strictly in order.
For EACH unit:
1. Implement ONLY that unit's scope. Do not start the next unit's code.
2. Verify: run `pytest -q` (for UI-only units also `python -m py_compile streamlit_app.py`; docs-only units skip tests).
   It must pass. Never commit failing tests, broken imports, or half-finished code.
3. Show me: the output of `git status --short`, then every created/modified file with a one-line summary of the change.
4. Stage EXPLICIT paths only: `git add <path1> <path2> ...`
   Never `git add .`, `git add -A`, `git add -u`, or `git commit -a`. Never stage .env, *.db, .venv, caches.
5. Commit with EXACTLY the unit's message: `git commit -m "<message>"`. A single -m. No body. No trailers.
6. Push: `git push`.
7. Show: `git log -1 --format="%h | %an <%ae> | %s"`.
8. Only then start the next unit.
If tests fail, the commit hook rejects the commit, or push fails: STOP and report the exact output. Do not work around it.
If a unit's real scope differs from its planned message, propose a corrected message and wait for my OK.

FORBIDDEN in git:
- Any `git config` change (any scope), `--author`, `--no-verify`, `commit --amend`, `rebase`, `reset`,
  `push --force`/`--force-with-lease`, deleting or creating branches, opening PRs.
- `Co-authored-by`, `Generated with`, `Generated-by`, or ANY mention of an AI tool/agent/model in commit messages.
- Committing files from outside this repository folder.
Allowed for discarding an uncommitted experiment: `git restore <explicit file paths>`.
