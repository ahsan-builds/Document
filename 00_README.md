# Hiro - Complete Technical Documentation Package

## Overview

This documentation package provides exhaustive technical documentation for the **Hiro AI-Powered Recruitment Platform**, focusing exclusively on the **Backend (Django), Web Frontend (React), and MLOps (AI Resume Extraction)** components.

---

## Documentation Structure

### 1. Architecture & Integration Logic
**File**: [01_architecture_overview.md](01_architecture_overview.md)

**Covers**:
- System architecture diagram showing all three tiers
- Connection points between Frontend ↔ Backend ↔ MLOps
- Integration flows and API contracts
- Configuration and environment setup
- Middleware stack and request processing
- Session-based authentication mechanism

### 2. Backend API Documentation (Part 1)
**File**: [02_backend_api_part1.md](02_backend_api_part1.md)

**Covers**:
- User Authentication & Management APIs
  - Registration (`POST /users/register/`)
  - Login with session creation (`POST /users/login/`)
  - Current user retrieval (`GET /users/me/`)
  - Password change (`POST /users/change-password/`)
  - Logout (`POST /users/logout/`)
  - Login history (`GET /users/login-history/`)
- Detailed request/response schemas
- Business logic explanations
- Database table mappings

### 3. Backend API Documentation (Part 2)
**File**: [02_backend_api_part2.md](02_backend_api_part2.md)

**Covers**:
- Application Management APIs
  - Create application with resume upload (`POST /api/applications/`)
  - List applications with filtering (`GET /api/applications/`)
  - Update application status/score (`PATCH /api/applications/<id>/`)
  - Resume retrieval (`GET /api/applications/<id>/resume/`)
- Job Posting APIs
- Dashboard Statistics APIs
  - Customer statistics endpoint
  - Sales dashboard with charts data

### 4. MLOps & Algorithm Logic
**File**: [03_mlops_documentation.md](03_mlops_documentation.md)

**Covers**:
- **Resume Extraction Pipeline**:
  - Document loading (PDF/DOCX with hyperlink extraction)
  - OpenAI GPT-4o-mini structured extraction
  - Pydantic schema definitions
  - JSON output parsing
- **Resume Scoring Algorithm**:
  - Text chunking strategy
  - Vector embedding using OpenAI text-embedding-3-large
  - Qdrant vector database integration
  - Category-based weighted scoring (projects 50%, skills 25%, etc.)
  - Cosine similarity calculations
- **LinkedIn Scraping**:
  - Rate limiting mechanism (50 scrapes/day)
  - Activity tracking
- Configuration and environment variables

### 5. Frontend Functional Logic
**File**: [04_frontend_documentation.md](04_frontend_documentation.md)

**Covers**:
- Technology stack (React 18, TypeScript, Vite, Redux Toolkit)
- Application structure and directory layout
- State management with Redux + Redux Persist
- API integration layer:
  - `AuthService.ts` - Authentication APIs
  - `ApplicantService.ts` - Applicant CRUD operations
  - `ApplicationService.ts` - Application submissions
  - `BaseService.ts` - Axios configuration with interceptors
- Routing and page structure
- Key feature data flows:
  - User login flow
  - Application submission flow
  - Applicant profile viewing

### 6. Data Flow & Lifecycle Analysis
**File**: [05_dataflow_lifecycle.md](05_dataflow_lifecycle.md)

**Covers complete end-to-end data flows for**:
1. **User Registration & Authentication Lifecycle**:
   - Input: Registration form data
   - Processing: Frontend validation → API call → Serializer → Database write
   - Storage: `auth_user` and `users_userprofile` tables
   - Output: User object with profile

2. **Job Application Submission Lifecycle**:
   - Input: FormData with resume file
   - Processing: File upload → Get-or-create applicant → Store resume as BLOB
   - Storage: `applicants_applicant` and `applications_application` tables
   - Output: Application object with resume URL

3. **Resume Extraction & Profile Creation Lifecycle**:
   - Input: GET request for profile
   - Processing: Retrieve resume BLOB → Temp file → PDF parsing → LLM extraction → JSON parsing
   - Storage: `applicants_applicantprofile` with JSON fields
   - Output: Structured profile with skills, experience, education

4. **Application Status Update Lifecycle**:
   - Input: Status change (e.g., shortlist action)
   - Processing: PATCH request → Database update
   - Storage: Update `applications_application.status`
   - Output: Updated application object

---

## Key Technologies

### Backend
- **Framework**: Django 5.x
- **API**: Django REST Framework
- **Database**: PostgreSQL (database name: `posts`)
- **Authentication**: Session-based (sessionid cookie)
- **CORS**: Configured for `http://localhost:5173` (Vite dev server)

### Frontend
- **Framework**: React 18.2 with TypeScript
- **Build Tool**: Vite 4.3
- **State**: Redux Toolkit + Redux Persist
- **Styling**: Tailwind CSS 3.3
- **HTTP Client**: Axios with interceptors

### MLOps
- **LLM**: OpenAI GPT-4o-mini (structured output)
- **Embeddings**: OpenAI text-embedding-3-large
- **Vector DB**: Qdrant (cloud-hosted)
- **Document Loaders**: PyPDF, UnstructuredWordDocument
- **Framework**: LangChain

---

## Database Schema Summary

### Core Tables

**Users**:
- `auth_user`: Django built-in user model
- `users_userprofile`: Extended profile with telephone, department, position, preferences
- `users_loginhistory`: Login/logout audit trail

**Recruitment**:
- `applicants_applicant`: Applicant master data (name, email, contact info)
- `applicants_applicantprofile`: AI-extracted resume insights (JSON fields for skills, experience, education)
- `applicants_linkedinscrapingactivity`: Rate limiting for LinkedIn scraping
- `applications_application`: Links applicant to job with resume BLOB, status, score
- `posts_job`: Job postings with requirements, status, type

---

## Integration Points Summary

| Frontend Service | Backend Endpoint | Handler Function | Database Tables |
|-----------------|------------------|------------------|-----------------|
| `AuthService.apiSignIn()` | `POST /users/login/` | `users/views.py::LoginView.post()` | `auth_user`, `users_loginhistory` |
| `ApplicationService.createApplication()` | `POST /api/applications/` | `applications/views.py::ApplicationListCreateAPIView.create()` | `applicants_applicant`, `applications_application` |
| `ApplicantService.listApplicants()` | `GET /applicants/?job=<id>` | `applicants/views.py::ApplicantViewSet.list()` | `applicants_applicant`, `applications_application` |
| Frontend profile request | `GET /api/applicants/<id>/profile/` | `applicants/views.py::get_applicant_profile()` → MLOps | `applicants_applicantprofile` |

**Backend ↔ MLOps Integration**:
- `server/applicants/views.py::trigger_profile_extraction()` imports `rag.resume_extractor.extract_resume_insights()`
- Passes resume binary bytes to MLOps module
- Receives structured JSON response
- Saves to `ApplicantProfile` model

---

## Running the System

### Backend
```bash
cd server
python -m venv venv
.\venv\Scripts\activate  # Windows
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver  # http://127.0.0.1:8000
```

### Frontend
```bash
cd client
npm install
npm run start  # http://localhost:5173
```

### Environment Variables (MLOps)
Create `.env` file in `mlops/rag/`:
```
OPENAI_API_KEY=sk-...
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-api-key
```

---

## Exclusions

As per documentation requirements, the following components have been **completely excluded** from this documentation:
- Mobile app development (no iOS/Android/React Native/Flutter code exists in the codebase)
- Any mobile-specific logic or files

This documentation focuses 100% on the **Web Frontend, Backend, and MLOps** infrastructure.

