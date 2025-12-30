# Hiro - Deep-Dive Architecture & Integration Logic

## Project Overview

**Hiro** is an AI-powered recruitment web application built with a modern three-tier architecture consisting of:

1. **Backend**: Django 5.x + Django REST Framework + PostgreSQL
2. **Web Frontend**: React 18 + TypeScript + Redux Toolkit + Vite
3. **MLOps**: RAG-based Resume Extraction using OpenAI GPT-4o-mini + Qdrant Vector Database

---

## System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                       WEB FRONTEND (React)                       │
│  ┌────────────┐  ┌────────────┐  ┌─────────────┐               │
│  │  Services  │  │   Store    │  │    Views    │               │
│  │   Layer    │  │  (Redux)   │  │ Components  │               │
│  └─────┬──────┘  └────────────┘  └─────────────┘               │
└────────┼─────────────────────────────────────────────────────────┘
         │ HTTP/REST (axios)
         │ Base URL: http://127.0.0.1:8000
         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   BACKEND (Django + DRF)                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  URL Router (backend/urls.py)                            │  │
│  │  ├─ /users/*          → UserViews                        │  │
│  │  ├─ /applicants/*     → ApplicantViewSet                 │  │
│  │  ├─ /api/applications/* → ApplicationViews               │  │
│  │  ├─ /api/jobs/*       → JobViews                         │  │
│  │  └─ /api/applicants/<id>/profile/* → Profile Extraction  │  │
│  └────────────┬─────────────────────────────────────────────┘  │
│               │                                                  │
│  ┌────────────▼──────────────────────────────────────────────┐ │
│  │  Django Apps                                              │ │
│  │  ├─ users: Authentication, Sessions, LoginHistory        │ │
│  │  ├─ applicants: Applicant, ApplicantProfile              │ │
│  │  ├─ applications: Application (links Applicant + Job)    │ │
│  │  └─ posts: Job postings                                  │ │
│  └───────────┬───────────────────────────────────────────────┘ │
└──────────────┼──────────────────────────────────────────────────┘
               │ PostgreSQL (ORM)
               ▼
      ┌────────────────────┐
      │   PostgreSQL DB    │
      │   Database: posts  │
      └────────────────────┘
               │
               │ MLOps Integration Point
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    MLOPS (Resume Extraction)                     │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  rag/resume_extractor.py                                 │  │
│  │  └─ extract_resume_insights(resume_bytes, filename)      │  │
│  │     ├─ Loads PDF/DOCX using PyPDF/UnstructuredWord       │  │
│  │     ├─ Calls OpenAI GPT-4o-mini with structured output   │  │
│  │     └─ Returns JSON: {skills, experience, education...}  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  rag/utils.py - rank_resume_against_job()                │  │
│  │  ├─ Chunks resume text                                   │  │
│  │  ├─ Embeds chunks using OpenAI Embeddings                │  │
│  │  ├─ Retrieves job description from Qdrant               │  │
│  │  └─ Calculates weighted similarity score (0-100)         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
               │
               ▼
      ┌────────────────────┐
      │  Qdrant Vector DB  │
      │  (Job Descriptions)│
      └────────────────────┘
```

---

## Connection Points: Frontend ↔ Backend

### Authentication Flow

**File**: `client/src/services/AuthService.ts`

```typescript
const AUTH_BASE_URL = 'http://127.0.0.1:8000'
```

**Integration Points**:
- `POST /users/login/` → `server/users/views.py::LoginView`
- `POST /users/register/` → `server/users/views.py::RegisterView`
- `GET /users/me/` → `server/users/views.py::MeView`
- `POST /users/logout/` → `server/users/views.py::LogoutView`

**Authentication Mechanism**: Session-based authentication using Django sessions. The backend sets a `sessionid` cookie on successful login, which is automatically sent with subsequent requests via `withCredentials: true` in axios config.

### Applicant Management Flow

**File**: `client/src/services/ApplicantService.ts`

```typescript
const BASE_URL = 'http://127.0.0.1:8000/applicants/'
```

**Integration Points**:
- `GET /applicants/` → `server/applicants/views.py::ApplicantViewSet.list()`
- `GET /applicants/?job=<id>` → Filters applicants by job using Applications
- `POST /applicants/` → `ApplicantViewSet.create()`
- `GET /applicants/<id>/` → `ApplicantViewSet.retrieve()`
- `PATCH /applicants/<id>/` → `ApplicantViewSet.partial_update()`

### Application Submission Flow

**File**: `client/src/services/ApplicationService.ts`

```typescript
const BASE_URL = 'http://127.0.0.1:8000/api/applications/'
```

**Integration Points**:
- `POST /api/applications/` → `server/applications/views.py::ApplicationListCreateAPIView.create()`
- `GET /api/applications/?job=<id>` → Filters applications by job
- `PATCH /api/applications/<id>/` → Updates application status/score

---

## Backend ↔ MLOps Integration

### Resume Extraction Trigger Point

**File**: `server/applicants/views.py`

**Function**: `trigger_profile_extraction(applicant)` (Line 201-269)

**Workflow**:

1. **Invocation**: Called when `GET /api/applicants/<id>/profile/` is requested and profile doesn't exist
2. **Resume Retrieval**: Fetches most recent `Application.resume` (binary blob)
3. **MLOps Import**: 
   ```python
   from rag.resume_extractor import extract_resume_insights
   ```
4. **Extraction Call**:
   ```python
   insights = extract_resume_insights(
       resume_bytes=application.resume,
       filename=f"resume_{applicant.id}.pdf"
   )
   ```
5. **Database Write**: Creates `ApplicantProfile` with extracted insights

### MLOps Resume Extractor

**File**: `mlops/rag/resume_extractor.py`

**Function**: `extract_resume_insights(resume_bytes, filename)` (Line 95-186)

**Step-by-Step Logic**:

1. **Temp File Creation** (Line 122-125):
   - Writes binary resume to `tempfile.NamedTemporaryFile`
   - Preserves file extension for format detection

2. **Document Loading** (Line 129-131):
   - Calls `load_single_file(tmp_path)` from `rag/utils.py`
   - Supports PDF, DOCX, TXT formats
   - Extracts hyperlinks from PDF annotations for LinkedIn/GitHub

3. **LLM Configuration** (Line 139-142):
   - Model: `gpt-4o-mini` (fast, cost-effective)
   - Temperature: 0 (deterministic extraction)
   - Uses `JsonOutputParser` with Pydantic schema `ResumeInsights`

4. **Structured Output Schema** (Line 56-88):
   ```python
   class ResumeInsights(BaseModel):
       summary: str
       github_url: Optional[str]
       linkedin_url: Optional[str]
       skills: List[Skill]
       experience: List[Experience]
       education: List[Education]
       certifications: List[Certification]
       total_experience_years: Optional[float]
   ```

5. **Prompt Execution** (Line 165-169):
   - Sends resume text (first 10,000 chars) to OpenAI
   - Returns structured JSON matching schema

6. **Cleanup** (Line 179-185):
   - Removes temporary file

---

## Configuration & Environment

### Backend Configuration

**File**: `server/backend/settings.py`

**Database Connection** (Line 105-113):
```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": "posts",
        "USER": "postgres",
        "PASSWORD": "affan",
        "HOST": "localhost"
    }
}
```

**CORS Configuration** (Line 37-51):
```python
CORS_ALLOWED_ORIGINS = [
    "http://localhost:5173",  # Vite dev server
    "http://127.0.0.1:5173",
]
CORS_ALLOW_CREDENTIALS = True  # Required for session cookies
```

**MLOps Path Integration** (Line 20-23):
```python
MLOPS_DIR = BASE_DIR.parent / 'mlops'
if str(MLOPS_DIR) not in sys.path:
    sys.path.insert(0, str(MLOPS_DIR))
```

### MLOps Configuration

**File**: `mlops/rag/config.py`

**Environment Variables**:
- `QDRANT_URL`: Qdrant cloud URL
- `QDRANT_API_KEY`: API key for authentication
- `OPENAI_API_KEY`: OpenAI API key (loaded by langchain)

**Embedding Model** (Line 27-28):
```python
def get_embedding_model():
    return OpenAIEmbeddings(model="text-embedding-3-large")
```

**Category Weights for Scoring** (Line 41-47):
```python
CATEGORY_WEIGHTS = {
    "experience": 0.05,
    "skills": 0.25,
    "projects": 0.50,
    "education": 0.10,
    "institute": 0.10,
}
```

---

## Middleware & Request Flow

### Django Middleware Stack

**File**: `server/backend/settings.py` (Line 71-80)

```python
MIDDLEWARE = [
    "django.middleware.security.SecurityMiddleware",
    "corsheaders.middleware.CorsMiddleware",  # 1. CORS headers
    "django.contrib.sessions.middleware.SessionMiddleware",  # 2. Session management
    "django.middleware.common.CommonMiddleware",
    "django.middleware.csrf.CsrfViewMiddleware",  # 3. CSRF protection
    "django.contrib.auth.middleware.AuthenticationMiddleware",  # 4. User auth
    "django.contrib.messages.middleware.MessageMiddleware",
    "django.middleware.clickjacking.XFrameOptionsMiddleware",
]
```

**Request Flow**:
1. **CORS Middleware**: Adds CORS headers for cross-origin requests
2. **Session Middleware**: Loads session from `sessionid` cookie
3. **Auth Middleware**: Sets `request.user` from session
4. **View**: Processes request, checks `IsAuthenticated` permission
5. **Response**: Middleware adds headers on the way back

### Frontend Axios Interceptors

**File**: `client/src/services/BaseService.ts`

```typescript
BaseService.interceptors.response.use(
    (response) => response,
    (error) => {
        const { response } = error
        if (response && (response.status === 401 || response.status === 403)) {
            // Clear auth state and redirect to login
            store.dispatch(signOutSuccess())
        }
        return Promise.reject(error)
    }
)
```

**Purpose**: Automatically logs out users when backend returns 401/403 (session expired or unauthorized).

