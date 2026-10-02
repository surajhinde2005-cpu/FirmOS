# FirmOS — Full Software Development Workbook
### AI-Powered Practice Intelligence Assistant for Chartered Accountancy Firms
*(Based on your Project Synopsis, AY 2026-27 — GGSCOER, Dept. of Computer Engineering)*

This is a complete, phase-by-phase build guide: architecture → database → backend → frontend → OCR → deployment. Follow it top to bottom. Every phase lists **exact file names** for frontend and backend so you know precisely what to create and when.

---

## 0. Recommended Tech Stack (locked in, matches your synopsis Section 7)

| Layer | Technology | Why |
|---|---|---|
| Frontend | **React + Next.js (App Router)**, TypeScript, Tailwind CSS | SSR for dashboards, file-based routing, fast to build |
| Backend | **Python + FastAPI** | Async, auto Swagger docs, great for OCR/AI pipelines |
| Database | **PostgreSQL** | Relational data (clients, documents, requirements) |
| ORM | **SQLAlchemy 2.0 + Alembic** | Migrations + type-safe models |
| OCR | **Tesseract OCR** (`pytesseract`) + `pdf2image`, `Pillow` | Matches your synopsis; free, offline |
| File Storage | Local `/uploads` folder (dev) → **AWS S3 / Cloudinary** (prod) | Cheap start, easy swap later |
| Auth | **JWT (access + refresh tokens)**, `passlib[bcrypt]` | Stateless, works for CA portal + client portal |
| Background Jobs | **Celery + Redis** (or simple `BackgroundTasks` for MVP) | OCR is slow — don't block requests |
| NLQ (Section 4 scope) | **LangChain / LlamaIndex + Gemini API or OpenAI API**, pgvector | "Natural-language Q&A over authorized firm data" |
| Testing | `pytest` (backend), `Jest + React Testing Library` (frontend) | |
| Dev Tools | VS Code, Git, GitHub, Postman/Thunder Client, Docker | Matches synopsis |
| Deployment | Backend → Render/Railway; Frontend → Vercel; DB → Neon/Supabase Postgres | Free tiers, student-friendly |

---

## 1. SDLC Model to Use

Use **Agile/Incremental SDLC**, structured as **8 sprints of ~1–2 weeks each**, matching how CA-firm projects are normally evaluated in college (Review 1 = design, Review 2 = mid, Final = complete + report).

```
Phase 1: Requirement Analysis & Planning
Phase 2: System Design (DB + API + Architecture)
Phase 3: Project Setup & Environment
Phase 4: Authentication & User Management
Phase 5: Client & Workspace Management
Phase 6: Document Upload & Storage
Phase 7: OCR & Verification Engine
Phase 8: CA Dashboard & Tracking
Phase 9: Natural-Language Q&A Module
Phase 10: Testing, Security & Deployment
```

---

## 2. Phase 1 — Requirement Analysis & Planning

**Deliverables (no code yet):**
- Finalize actors: `CA (Admin)`, `Staff/Employee`, `Client`
- Finalize entities: User, Client, Workspace, DocumentRequirement, Document, Deadline
- Write **User Stories** for each module (e.g., "As a CA, I want to see pending documents so I can follow up with clients")
- Freeze **scope** exactly as your synopsis Section 4 states — don't add auto tax-filing etc. (explicitly a limitation)

**Tool to use:** Notion / Google Docs for user stories, GitHub Projects (Kanban) for task tracking.

---

## 3. Phase 2 — System Design

### 3.1 High-Level Architecture

```
┌─────────────┐      HTTPS/JSON       ┌──────────────────┐
│   Next.js   │ ───────────────────▶ │   FastAPI Backend │
│  Frontend   │ ◀─────────────────── │   (REST API)      │
└─────────────┘                       └─────────┬─────────┘
                                                 │
                     ┌───────────────────────────┼───────────────────────────┐
                     ▼                           ▼                           ▼
             ┌───────────────┐          ┌─────────────────┐         ┌────────────────┐
             │  PostgreSQL   │          │  File Storage    │         │  OCR Worker      │
             │  (clients,    │          │  (local/S3)      │         │  (Tesseract +    │
             │  docs, users) │          │                  │         │  Celery/Redis)   │
             └───────────────┘          └─────────────────┘         └────────────────┘
                                                                              │
                                                                              ▼
                                                                     ┌────────────────┐
                                                                     │  NLQ Engine     │
                                                                     │  (LangChain +   │
                                                                     │  Gemini API)    │
                                                                     └────────────────┘
```

### 3.2 Database Schema (PostgreSQL)

```
users
 ├─ id (PK, UUID)
 ├─ name
 ├─ email (unique)
 ├─ phone
 ├─ password_hash
 ├─ role  (enum: 'ca_admin' | 'staff' | 'client')
 ├─ created_at

clients
 ├─ id (PK, UUID)
 ├─ ca_id (FK → users.id)          -- which CA firm owns this client
 ├─ user_id (FK → users.id)        -- the client's own login (nullable until they register)
 ├─ name
 ├─ email
 ├─ phone
 ├─ pan_number
 ├─ gstin
 ├─ status (active/inactive)
 ├─ created_at

workspaces
 ├─ id (PK, UUID)
 ├─ client_id (FK → clients.id)
 ├─ financial_year   e.g. "2026-27"
 ├─ created_at

document_requirements
 ├─ id (PK, UUID)
 ├─ workspace_id (FK → workspaces.id)
 ├─ doc_type        e.g. "PAN Card", "Bank Statement", "GST Return"
 ├─ is_mandatory (bool)
 ├─ deadline_date
 ├─ status (pending / submitted / verified / rejected)

documents
 ├─ id (PK, UUID)
 ├─ requirement_id (FK → document_requirements.id)
 ├─ file_path / s3_key
 ├─ original_filename
 ├─ uploaded_by (FK → users.id)
 ├─ ocr_text (text, nullable)
 ├─ extracted_fields (JSONB)     -- e.g. {"pan":"ABCDE1234F","name":"..."}
 ├─ verification_status (pending/passed/failed)
 ├─ verification_notes
 ├─ uploaded_at

notifications
 ├─ id (PK, UUID)
 ├─ user_id (FK)
 ├─ message
 ├─ is_read (bool)
 ├─ created_at

chat_queries   (for NLQ module)
 ├─ id (PK, UUID)
 ├─ ca_id (FK → users.id)
 ├─ question
 ├─ answer
 ├─ created_at
```

### 3.3 REST API Design (contract-first)

| Method | Endpoint | Purpose |
|---|---|---|
| POST | `/api/auth/register` | Register CA or client |
| POST | `/api/auth/login` | Login → returns JWT |
| POST | `/api/auth/refresh` | Refresh token |
| GET | `/api/clients` | List CA's clients |
| POST | `/api/clients` | Add new client |
| GET | `/api/clients/{id}` | Client detail |
| PUT | `/api/clients/{id}` | Update client |
| DELETE | `/api/clients/{id}` | Remove client |
| POST | `/api/workspaces` | Create workspace for client |
| POST | `/api/workspaces/{id}/requirements` | Add required document type |
| GET | `/api/workspaces/{id}/requirements` | List requirements + status |
| POST | `/api/documents/upload` | Client uploads a document (multipart) |
| GET | `/api/documents/{id}` | Get document + OCR result |
| POST | `/api/documents/{id}/verify` | Trigger/re-run verification |
| GET | `/api/dashboard/pending` | CA dashboard: pending docs + deadlines |
| GET | `/api/dashboard/attention` | Clients requiring attention |
| POST | `/api/chat/ask` | Natural-language Q&A over firm data |

**Tool to design this visually:** dbdiagram.io (for ER diagram) + Postman collection (for API contracts) — build the Postman collection *before* writing backend code.

---

## 4. Phase 3 — Project Setup & Folder Structure

### 4.1 Backend folder structure (FastAPI)

```
firmos-backend/
├── app/
│   ├── main.py                        # FastAPI app entrypoint
│   ├── config.py                      # env vars, settings (pydantic-settings)
│   ├── database.py                    # SQLAlchemy engine + session
│   ├── dependencies.py                # get_db, get_current_user
│   │
│   ├── models/
│   │   ├── user.py
│   │   ├── client.py
│   │   ├── workspace.py
│   │   ├── document.py
│   │   └── requirement.py
│   │
│   ├── schemas/                       # Pydantic request/response models
│   │   ├── user_schema.py
│   │   ├── client_schema.py
│   │   ├── document_schema.py
│   │   └── auth_schema.py
│   │
│   ├── routers/                       # API route files (1 per module)
│   │   ├── auth_router.py
│   │   ├── client_router.py
│   │   ├── workspace_router.py
│   │   ├── document_router.py
│   │   ├── dashboard_router.py
│   │   └── chat_router.py
│   │
│   ├── services/                      # business logic (kept OUT of routers)
│   │   ├── auth_service.py
│   │   ├── client_service.py
│   │   ├── ocr_service.py             # Tesseract wrapper
│   │   ├── verification_service.py    # rule-based checks
│   │   ├── storage_service.py         # save/read files (local or S3)
│   │   └── nlq_service.py             # LangChain + Gemini/OpenAI
│   │
│   ├── core/
│   │   ├── security.py                # JWT, password hashing
│   │   └── celery_app.py              # background job config
│   │
│   └── utils/
│       └── validators.py              # PAN/GST regex validators
│
├── alembic/                            # DB migrations
│   └── versions/
├── tests/
│   ├── test_auth.py
│   ├── test_clients.py
│   └── test_documents.py
├── uploads/                             # local dev file storage
├── requirements.txt
├── alembic.ini
├── Dockerfile
└── .env
```

### 4.2 Frontend folder structure (Next.js App Router + TypeScript)

```
firmos-frontend/
├── app/
│   ├── layout.tsx
│   ├── page.tsx                        # landing page
│   │
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   │
│   ├── (ca-portal)/
│   │   ├── dashboard/page.tsx          # pending docs + deadlines
│   │   ├── clients/page.tsx            # client list
│   │   ├── clients/[id]/page.tsx       # client detail + requirements
│   │   ├── clients/new/page.tsx        # add client form
│   │   └── chat/page.tsx               # NLQ assistant UI
│   │
│   ├── (client-portal)/
│   │   └── workspace/[id]/page.tsx     # client's upload workspace
│   │
│   └── api/                            # (only if using Next.js API proxy)
│
├── components/
│   ├── ui/                             # buttons, inputs, modals (shadcn/ui)
│   ├── layout/
│   │   ├── Navbar.tsx
│   │   └── Sidebar.tsx
│   ├── clients/
│   │   ├── ClientCard.tsx
│   │   └── ClientForm.tsx
│   ├── documents/
│   │   ├── UploadDropzone.tsx
│   │   ├── DocumentStatusBadge.tsx
│   │   └── DocumentList.tsx
│   ├── dashboard/
│   │   ├── PendingTable.tsx
│   │   └── DeadlineTimeline.tsx
│   └── chat/
│       └── ChatWindow.tsx
│
├── lib/
│   ├── api.ts                          # axios/fetch wrapper, base URL
│   ├── auth.ts                         # token storage/refresh logic
│   └── types.ts                        # shared TypeScript interfaces
│
├── hooks/
│   ├── useAuth.ts
│   ├── useClients.ts
│   └── useDocuments.ts
│
├── store/                              # Zustand or Redux (auth + UI state)
│   └── authStore.ts
│
├── styles/
│   └── globals.css
│
├── .env.local
├── next.config.js
├── tailwind.config.ts
└── package.json
```

---

## 5. Phase 4 — Authentication Module

**Backend files to build (in order):**
1. `app/models/user.py` — SQLAlchemy `User` model
2. `app/core/security.py` — `hash_password()`, `verify_password()`, `create_access_token()`
3. `app/schemas/auth_schema.py` — `RegisterRequest`, `LoginRequest`, `TokenResponse`
4. `app/services/auth_service.py` — register + login logic
5. `app/routers/auth_router.py` — `/api/auth/register`, `/api/auth/login`, `/api/auth/refresh`
6. `app/dependencies.py` — `get_current_user()` dependency (decodes JWT, used to protect all other routes)

**Frontend files:**
1. `lib/auth.ts` — store JWT (httpOnly cookie or memory + refresh)
2. `app/(auth)/login/page.tsx`
3. `app/(auth)/register/page.tsx`
4. `hooks/useAuth.ts`
5. `store/authStore.ts`
6. Middleware: `middleware.ts` (Next.js route protection based on role)

**Test:** Postman → register a CA, login, confirm JWT returned and protected route rejects requests without token.

---

## 6. Phase 5 — Client & Workspace Management

**Backend:**
- `app/models/client.py`, `app/models/workspace.py`
- `app/schemas/client_schema.py`
- `app/services/client_service.py`
- `app/routers/client_router.py`, `app/routers/workspace_router.py`

**Frontend:**
- `app/(ca-portal)/clients/page.tsx` — table of clients + "Add Client" button
- `app/(ca-portal)/clients/new/page.tsx` — form (name, email, phone, PAN)
- `app/(ca-portal)/clients/[id]/page.tsx` — client's workspace, requirement list
- `components/clients/ClientForm.tsx`, `components/clients/ClientCard.tsx`
- `hooks/useClients.ts` — fetch/create/update client via `lib/api.ts`

**Business rule:** when a client is created, auto-create a `workspace` for the current financial year, and let the CA attach `document_requirements` (checklist) to it.

---

## 7. Phase 6 — Document Upload & Storage

**Backend:**
- `app/services/storage_service.py` — saves file to `/uploads/{client_id}/{doc_type}/filename`, returns path (swap to S3 later by changing only this file)
- `app/schemas/document_schema.py`
- `app/routers/document_router.py` — `POST /api/documents/upload` (multipart form), validates file type (pdf/jpg/png), size limit
- `app/models/document.py`

**Frontend:**
- `components/documents/UploadDropzone.tsx` — drag-drop uploader (use `react-dropzone`)
- `components/documents/DocumentList.tsx`, `DocumentStatusBadge.tsx`
- `app/(client-portal)/workspace/[id]/page.tsx` — client sees checklist, uploads against each requirement
- `hooks/useDocuments.ts`

---

## 8. Phase 7 — OCR & Verification Engine (core of synopsis Section 5–6)

**Backend:**
1. `app/services/ocr_service.py`
   - Use `pdf2image` to convert PDF pages to images (if PDF)
   - Use `pytesseract.image_to_string()` to extract raw text
   - Use regex/field-parsers to pull structured fields (PAN number pattern `[A-Z]{5}[0-9]{4}[A-Z]{1}`, GSTIN pattern, dates, names)
2. `app/services/verification_service.py`
   - Rule-based checks: does extracted PAN match client's stored PAN? Is the document not expired? Is the required field present?
   - Sets `verification_status = passed/failed` + `verification_notes`
3. `app/core/celery_app.py` + Celery worker — run OCR as a background task so upload API responds instantly, OCR runs async, and updates DB when done
4. Update `document_router.py` to trigger `ocr_service` + `verification_service` after upload (via Celery task `.delay()`)

**Frontend:**
- `components/documents/DocumentStatusBadge.tsx` — show "Processing... / Verified ✅ / Failed ⚠️" with polling or WebSocket
- Add a "re-verify" button calling `POST /api/documents/{id}/verify`

**Tool note:** Install Tesseract binary separately (`sudo apt install tesseract-ocr` on Linux, or the Windows installer) — `pytesseract` is just a Python wrapper around it.

---

## 9. Phase 8 — CA Dashboard & Tracking

**Backend:**
- `app/routers/dashboard_router.py`
  - `GET /api/dashboard/pending` — aggregate query: all `document_requirements` where `status='pending'` joined with deadline
  - `GET /api/dashboard/attention` — clients with overdue or >X pending docs

**Frontend:**
- `app/(ca-portal)/dashboard/page.tsx`
- `components/dashboard/PendingTable.tsx`
- `components/dashboard/DeadlineTimeline.tsx`
- Use a charting lib (Recharts) for a simple "documents pending by client" bar chart

---

## 10. Phase 9 — Natural-Language Q&A Module (Section 4 scope item)

This is the "AI-Powered" differentiator in your title — treat it as its own phase.

**Backend:**
- `app/services/nlq_service.py`
  - Convert firm data (client status, pending docs) into text chunks
  - Store embeddings in **pgvector** (Postgres extension) or a simple in-memory FAISS index for the prototype
  - On a question, retrieve relevant chunks (RAG) → send to **Gemini API** with context → return answer
  - **Critical guardrail:** restrict retrieval to only the logged-in CA's own client data (authorization filter in the SQL query, not just the prompt)
- `app/routers/chat_router.py` — `POST /api/chat/ask`
- `app/models` — add `chat_queries` table to log Q&A for audit

**Frontend:**
- `app/(ca-portal)/chat/page.tsx`
- `components/chat/ChatWindow.tsx` — simple chat UI, calls `/api/chat/ask`

---

## 11. Phase 10 — Testing, Security & Deployment

**Testing:**
- Backend: `tests/test_auth.py`, `test_clients.py`, `test_documents.py` using `pytest` + `httpx.AsyncClient`
- Frontend: `Jest` + `React Testing Library` for components; Cypress/Playwright for E2E (login → add client → upload doc → see verified status)

**Security checklist (mention in your report — examiners look for this):**
- Passwords hashed with bcrypt, never stored plain
- JWT with short expiry + refresh token rotation
- File upload validation (type, size, virus-scan optional)
- Role-based access control (CA cannot see another CA's clients)
- HTTPS in production, CORS locked to your frontend domain
- Input validation via Pydantic schemas (prevents SQL injection by default with ORM)

**Deployment:**
| Component | Where | How |
|---|---|---|
| Backend + Celery worker | Render / Railway | Dockerfile-based deploy |
| Database | Neon / Supabase (managed Postgres) | Free tier |
| Redis | Upstash | Free tier |
| Frontend | Vercel | Auto-deploy from GitHub `main` branch |
| File storage | AWS S3 (or keep local disk if within academic scope) | |

---

## 12. Suggested Sprint Timeline (for your 2-semester academic project)

| Sprint | Weeks | Deliverable |
|---|---|---|
| 1 | 1–2 | Phase 1–2: requirements frozen, DB schema, API contract, wireframes |
| 2 | 3–4 | Phase 3–4: project scaffolding + auth working end-to-end |
| 3 | 5–6 | Phase 5: client + workspace CRUD |
| 4 | 7–8 | Phase 6: document upload working |
| 5 | 9–11 | Phase 7: OCR + verification (hardest phase, buffer extra time) |
| 6 | 12–13 | Phase 8: dashboard |
| 7 | 14–15 | Phase 9: NLQ chatbot |
| 8 | 16–18 | Phase 10: testing, bug fixes, deployment, report writing |

---

## 13. Best Tools for Each Job (your "which tool is best" question)

| Job | Best tool | Why |
|---|---|---|
| **AI coding assistant (writing actual code)** | **Claude Code** (or Cursor) | Best for multi-file, whole-repo edits and staying consistent with an existing codebase — better than plain chat-based Gemini for this size of project |
| **Quick AI Q&A / architecture brainstorming** | Gemini (web/app) or Claude chat | Good for planning conversations, not ideal for large repo edits |
| API design/testing | Postman or Thunder Client (VS Code ext) | |
| DB schema design | dbdiagram.io | Visual ER diagrams, exports SQL |
| UI/UX mockup | Figma | Design screens before coding |
| Version control | Git + GitHub (with GitHub Projects for Kanban) | |
| Local dev orchestration | Docker Compose (Postgres + Redis + backend together) | One command to run everything |
| Project/task tracking | GitHub Projects or Trello | Simple for a 4-person team |

**Practical recommendation:** Do architecture + planning conversations in Gemini/Claude chat, then hand the *finalized* schema + file structure (this document) to **Claude Code** or **Gemini CLI** running inside your actual project folder to generate the files — a coding agent needs the real repo open to be most effective, which a plain chat window doesn't have.

---

## 14. Ready-to-Use Prompt for Gemini (or any AI coding assistant / Gemini CLI)

Copy-paste this as your first message when you open Gemini CLI (or Gemini in your IDE) inside your empty project folder. It encodes everything above so the AI builds consistently with this plan instead of improvising.

```
You are acting as a senior full-stack engineer. Build "FirmOS" — a CA
(Chartered Accountant) document management and verification platform —
using this exact stack and structure. Do not deviate from the file names
or architecture below unless you flag a specific technical reason.

STACK:
- Backend: Python 3.11, FastAPI, SQLAlchemy 2.0, Alembic, PostgreSQL,
  Celery + Redis for background jobs, pytesseract + pdf2image for OCR,
  JWT auth via python-jose + passlib[bcrypt].
- Frontend: Next.js 14 (App Router), TypeScript, Tailwind CSS, shadcn/ui,
  Zustand for state, react-dropzone for uploads, Recharts for charts.
- NLQ module: LangChain + Gemini API, pgvector for embeddings.

BACKEND FOLDER STRUCTURE (create exactly this):
app/main.py, app/config.py, app/database.py, app/dependencies.py
app/models/{user,client,workspace,document,requirement}.py
app/schemas/{user_schema,client_schema,document_schema,auth_schema}.py
app/routers/{auth_router,client_router,workspace_router,document_router,dashboard_router,chat_router}.py
app/services/{auth_service,client_service,ocr_service,verification_service,storage_service,nlq_service}.py
app/core/{security,celery_app}.py
app/utils/validators.py
alembic/ (migrations), tests/, uploads/, requirements.txt, Dockerfile, .env

FRONTEND FOLDER STRUCTURE (create exactly this):
app/layout.tsx, app/page.tsx
app/(auth)/login/page.tsx, app/(auth)/register/page.tsx
app/(ca-portal)/dashboard/page.tsx
app/(ca-portal)/clients/page.tsx, clients/new/page.tsx, clients/[id]/page.tsx
app/(ca-portal)/chat/page.tsx
app/(client-portal)/workspace/[id]/page.tsx
components/{ui,layout,clients,documents,dashboard,chat}/...
lib/{api.ts,auth.ts,types.ts}
hooks/{useAuth,useClients,useDocuments}.ts
store/authStore.ts

DATABASE SCHEMA:
users(id, name, email, phone, password_hash, role[ca_admin|staff|client], created_at)
clients(id, ca_id FK users, user_id FK users nullable, name, email, phone, pan_number, gstin, status, created_at)
workspaces(id, client_id FK clients, financial_year, created_at)
document_requirements(id, workspace_id FK workspaces, doc_type, is_mandatory, deadline_date, status[pending|submitted|verified|rejected])
documents(id, requirement_id FK document_requirements, file_path, original_filename, uploaded_by FK users, ocr_text, extracted_fields JSONB, verification_status, verification_notes, uploaded_at)
notifications(id, user_id FK, message, is_read, created_at)
chat_queries(id, ca_id FK users, question, answer, created_at)

BUILD ORDER (do not skip ahead):
1. Scaffold both repos with the folder structures above (empty files with
   TODO comments first, so I can review structure before logic).
2. Implement auth (register/login/JWT) end to end, backend + frontend,
   and confirm a protected route rejects unauthenticated requests.
3. Implement client + workspace CRUD, backend + frontend.
4. Implement document upload (multipart, validate type/size, store to
   local /uploads path via storage_service.py so it's swappable for S3
   later).
5. Implement OCR (pytesseract) + rule-based verification as a Celery
   background task, triggered after upload, updating document status
   asynchronously.
6. Implement the CA dashboard (pending documents, deadlines, clients
   needing attention) with real aggregate queries, not mock data.
7. Implement the NLQ chat module: retrieval must be scoped to only the
   logged-in CA's own client data — never cross-tenant.
8. Write pytest tests for auth, clients, and document upload/verification.
9. Add a docker-compose.yml that runs postgres + redis + backend together
   for one-command local dev.

RULES:
- Never put business logic directly in router files — routers call
  services/ only.
- Every API response must use a Pydantic response_model.
- Every new backend endpoint needs a matching frontend hook in hooks/.
- After each numbered step above, stop and summarize what you built and
  which files changed before moving to the next step.

Start with step 1 now.
```

---

## 15. What to Hand In at Each College Review (mapping back to your synopsis)

- **Review 1 (now):** This synopsis PDF + this workbook's Phase 1–2 (requirements, ER diagram, API contract, wireframes)
- **Review 2 (mid):** Working auth + client management + document upload (Phases 3–6), demo on localhost
- **Final Review:** Full OCR + verification + dashboard + NLQ + deployed link (Phases 7–10), final report referencing your synopsis's References [1]–[6]

---

*Keep this file in your repo root as `DEVELOPMENT_GUIDE.md` — update the phase checkboxes as your team completes each one.*
