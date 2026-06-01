# Axioms — AI Purchase Decision Engine

Axioms is an opinionated procurement decision platform that helps organisations compare vendor quotations or explore catalogue options and produce explainable, defensible recommendations. This repository contains both the public-facing site (Astro-based frontend) and the FastAPI backend that handles uploads, parsing, normalisation and evaluation.

Contents
- `DataNova/` — Astro frontend (landing, decision UI, contact pages). The landing has been rebranded to *Axiom* visually.
- `backend_stuff/` — FastAPI backend providing upload, normalisation, requirements discovery and evaluation endpoints.
- `lovable/` — visual prototype (design reference used for landing visuals).

Quick links
- Frontend entry: [DataNova/src/pages/index.astro](DataNova/src/pages/index.astro)
- Landing component: [DataNova/src/components/procurement/LandingPage.astro](DataNova/src/components/procurement/LandingPage.astro)
- Backend upload routes: [backend_stuff/app/routes/upload.py](backend_stuff/app/routes/upload.py)

Prerequisites
- Node >= 18 and `pnpm` for the frontend.
- Python 3.10+ and `pip` for the backend.
- (Optional) `git` for source control operations.

Local setup

1. Frontend (DataNova)

```bash
cd DataNova
pnpm install
pnpm dev
```

- The frontend dev server runs via Astro and typically opens on `http://127.0.0.1:4323/` (Vite/Dev server may pick a different port if one is busy).
- The frontend reads the backend base URL from the `PUBLIC_API_BASE_URL` environment variable — see `DataNova/src/lib/api.ts`.

2. Backend (FastAPI)

```bash
cd backend_stuff
python -m venv .venv
.venv\Scripts\Activate.ps1   # PowerShell
# or: source .venv/bin/activate  # macOS / Linux
pip install -r requirements.txt
uvicorn app.main:app --reload --host 127.0.0.1 --port 8000
```

- Backend default API base: `http://127.0.0.1:8000`.
- Uploaded files are stored under `backend_stuff/app/storage/uploads` and normalised JSON is written to `backend_stuff/app/storage/normalised`.

API overview

- `POST /upload` — Accepts `kind` (form) and `files` (multipart). Supports uploading multiple PDF files in a single request. Returns metadata for each uploaded file. See `backend_stuff/app/routes/upload.py`.
- `POST /upload/normalise` — Same inputs as `/upload`, but additionally parses and normalises each file and writes normalised JSON to storage. Returns per-file normalisation results.
- `GET /health` — Health check endpoint.
- Other endpoints: requirements discovery, weighting handoff, catalogue evaluation and vendor evaluation — see `DataNova/src/lib/api.ts` for client routes and `backend_stuff/app/routes` for server implementations.

Important notes and constraints
- File types: only PDFs are accepted (server-side check in `upload.py`).
- Multiple files: both `/upload` and `/upload/normalise` accept `files: list[UploadFile]` — there is no application-level cap on the number of files per request, but server/proxy timeouts and request size limits still apply.
- Error handling: invalid files result in `400`/`415`; parsing or normalisation errors return `422`/`502` respectively for affected files.

Design & assets

- The landing page uses assets and styles ported from the `lovable` prototype (OKLCH tokens and layered gradients). The hero video is `DataNova/public/axiom-hero.mp4`.
- Visual components like translucent cards and gradient blending live in `DataNova/src/components` and inline styles in `LandingPage.astro`.

Testing

- Backend tests: (none included by default). To validate endpoints manually, use `curl` or Postman.

Example multi-file upload (curl)

```bash
curl -X POST "http://127.0.0.1:8000/upload" \
	-F "kind=quotations" \
	-F "files=@/path/to/quote1.pdf" \
	-F "files=@/path/to/quote2.pdf"
```

Frontend example (fetch)

```js
const form = new FormData();
form.append('kind', 'quotations');
for (const file of input.files) form.append('files', file);
await fetch(`${import.meta.env.PUBLIC_API_BASE_URL || 'http://127.0.0.1:8000'}/upload`, { method: 'POST', body: form });
```

Contribution & development notes

- Keep backend routes and the frontend API list in sync (`DataNova/src/lib/api.ts`).
- Use the layout flags in `DataNova/src/layout/BaseLayout.astro` (`landingFonts`, `footerMinimal`, `chrome`) to control page chrome and font preloads.
- When modifying visual tokens, prefer migrating OKLCH tokens into `DataNova/src/assets/styles/global.css` for consistent color parity across pages.

Where to look next
- Landing visuals: `DataNova/src/components/procurement/LandingPage.astro` and `lovable/src/styles.css` (reference tokens).
- Upload flow and normalisation: `backend_stuff/app/routes/upload.py`, `backend_stuff/app/services/file_parser.py`, `backend_stuff/app/services/normaliser.py`.

Solution

- Architecture overview:
	- Frontend: `DataNova` (Astro + client-side JS) provides the landing, decision UI, and upload UX. It keeps the UX-focused code separate from the backend and uses `PUBLIC_API_BASE_URL` to target the API.
	- Backend: `backend_stuff` (FastAPI) exposes REST endpoints for uploads, parsing, normalisation, requirements discovery and evaluation. Core services live under `app/services` and are orchestrated by route handlers in `app/routes`.

- Data flow (happy path):
	1. User uploads one or more PDF quotations from the frontend (multipart form `files[]`, `kind`).
 2. Backend `/upload` stores the PDFs in `storage/uploads` and returns metadata per file.
 3. Optionally `/upload/normalise` parses each file (`file_parser`), converts PDF content into structured vendor facts, and runs domain normalisation (`normaliser`).
 4. Normalised outputs are saved to `storage/normalised` and returned to the client for downstream steps (requirements discovery, weighting, evaluation).

- Key design choices:
	- Explainability first: normalised documents and evaluation outputs are structured so tradeoffs and scoring can be surfaced to users.
	- Modular services: parsing and normalisation are implemented as discrete services to allow replacing or scaling them independently.
	- Visual parity: the landing uses OKLCH tokens and layered gradients copied from the `lovable` prototype to match the target design closely.

- Operational considerations & scaling:
	- The current implementation writes uploads and normalised JSON to local disk — for production, switch to object storage (S3/Blob) and store metadata in a database.
	- Parsing/normalisation are CPU/IO bound. For higher throughput, process uploads asynchronously via a queue (Redis/RabbitMQ) with worker processes and return a task ID to clients.
	- Add rate-limiting, request-size limits and reverse-proxy timeouts to protect the API from oversized multipart requests.

- Security & validation:
	- Server-side checks enforce allowed file extensions (PDF) and non-empty content. Add virus scanning and stricter MIME/content validation before storing files.
	- Add authentication and authorization (JWT/OAuth) for production APIs, and ensure uploads are scoped to authenticated tenants/projects.

- Observability & reliability:
	- Add structured logging, request tracing (W3C Trace Context), and metrics (Prometheus) around parsing and normalisation to monitor failures and performance.
	- Surface per-file errors clearly in `/upload/normalise` responses so clients can handle partial failures.

- Limitations & next steps:
	- No automated tests are included; add unit and integration tests (file parsing edge cases, normaliser rules, API error handling).
	- Replace local disk storage with durable cloud storage and add metadata persistence.
	- Harden file upload UX with progress indicators, client-side validation, and resumable uploads for large batches.

License
- See `LICENSE` at the repo root (if present). If no license exists, treat this repo as private/internal work until a license is added.

Questions or changes
- If you'd like a README section expanded (deployment, CI, Docker, or environment samples), tell me which area and I will extend it.


