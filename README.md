# RevisAI

AI weak-topic diagnostic quiz generator. Paste or photograph your notes for 3–5 topics,
answer a generated quiz, and see a confidence-vs-accuracy breakdown per topic.

## How it works (architecture)

One Flask file (`app.py`), no separate backend/frontend split, no build step. Routes render
server-side Jinja templates (`templates/`) directly.

Everything AI-related runs through a single external service — Google's Gemini API:
- **Photo of notes → text:** the image is sent to Gemini's multimodal endpoint, which reads
  and transcribes it directly (no local OCR engine, no separate OCR API).
- **Notes → quiz questions:** the same Gemini API is prompted to write grounded multiple-choice
  questions from the extracted text.
- **No Gemini key set:** the app falls back to a built-in offline question generator
  (`semantic_question_synthesizer`) that recognizes a handful of hardcoded subjects
  (physics, Python, biology, OS/deadlocks, economics) and otherwise builds generic
  questions from your pasted text. Photo upload specifically requires a Gemini key,
  since there's no local OCR fallback — paste text instead if you don't have one.

## Run locally

```bash
python -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements.txt

export SECRET_KEY="something-random"       # Windows: set SECRET_KEY=...
export GEMINI_API_KEY="your-key-here"      # optional — required only for photo upload
python app.py
```

Visit http://localhost:7860.

## Deploy

The app is a standard Flask app served by gunicorn. Any host that runs a Docker
container or a Python web service works — no system packages are needed anymore
(no Tesseract, no OCR binaries), which keeps the Dockerfile minimal.

### Option A — Docker

```bash
docker build -t revisai .
docker run -p 7860:7860 -e SECRET_KEY=... -e GEMINI_API_KEY=... revisai
```

Push this image to any container host: Render (New → Web Service → Docker),
Railway, Fly.io, Google Cloud Run, or a VPS.

### Option B — Native Python host (Procfile)

1. Push this folder to a GitHub repo.
2. On Render/Railway: New → Web Service → connect the repo.
3. Build command: `pip install -r requirements.txt`
4. Start command: (from `Procfile`, usually auto-detected) `gunicorn app:app --bind 0.0.0.0:$PORT`
5. Set environment variables `SECRET_KEY` and `GEMINI_API_KEY`.

### Environment variables

| Variable         | Required | Purpose                                                        |
|------------------|----------|------------------------------------------------------------------|
| `SECRET_KEY`     | Yes (prod) | Flask session signing key — set a random string                |
| `GEMINI_API_KEY` | Recommended | Powers photo-upload OCR and AI-written questions; without it, photo upload is disabled and quizzes use the offline engine |
| `PORT`           | No       | Defaults to 7860; most hosts set this automatically             |

## Known limitations

- **Sessions live in memory** (`SESSIONS` dict in `app.py`), not a database. That's fine for a
  single-process demo, but it means: (a) restarting the app clears everyone's quizzes/history,
  and (b) it will not work correctly if the host scales you to more than one process/worker —
  a user's requests could land on a worker that never saw their data. Keep worker count at 1
  (already set in the Dockerfile/Procfile) unless you swap in a real session store
  (e.g. Redis via `Flask-Session`).
- **Photo upload requires a Gemini API key.** There's no local OCR fallback — if no key is set,
  the upload form asks you to paste notes as text instead.
- The offline "semantic question synthesizer" (used when no Gemini key is set) recognizes a
  handful of hardcoded subjects (physics/mechanics, Python, cell biology, OS/deadlocks,
  economics) — topics outside those get generic template-based questions built from your
  pasted text, which won't be as sharp as Gemini-generated ones.
