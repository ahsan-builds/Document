# Hiro Backend API - Part 2: Applications & Jobs

## 3. APPLICATION MANAGEMENT

### 3.1 Create Application

**Endpoint**: `POST /api/applications/`

**Handler**: `server/applications/views.py::ApplicationListCreateAPIView.create()` (Line 64-98)

**Purpose**: Submits a job application with resume upload. Creates or retrieves an `Applicant` record and links it to a `Job` via the `Application` model.

**Request Content-Type**: `multipart/form-data`

**Request Body (FormData)**:
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Applicant's full name |
| `email` | string | Yes | Applicant's email (unique identifier) |
| `job` | integer | Yes | Job ID to apply for |
| `resume_file` | file | Yes | Resume file (PDF/DOCX) |
| `score` | string | No | Initial AI score (default: 0) |
| `status` | string | No | Application status (default: "pending") |

**Business Logic** (`applications/serializers.py::ApplicationCreateSerializer.create()`, Line 42-124):

1. **Get or Create Applicant** (Line 48-62):
   ```python
   applicant, created = Applicant.objects.get_or_create(
       email=validated_data['email'],
       defaults={'name': validated_data['name'], ...}
   )
   ```
   - Uses email as unique identifier
   - If applicant exists, updates name
   - Creates new applicant if email not found

2. **Read Resume File** (Line 64-65):
   ```python
   resume_file = validated_data.get('resume_file')
   resume_bytes = resume_file.read() if resume_file else None
   ```

3. **Create Application** (Line 70-76):
   ```python
   application = Application.objects.create(
       applicant=applicant,
       job=validated_data['job'],
       resume=resume_bytes,  # Stored as PostgreSQL bytea
       score=validated_data.get('score', 0),
       status=validated_data.get('status', 'pending')
   )
   ```

4. **Unique Constraint**: Database enforces `unique_together = ['applicant', 'job']` - prevents duplicate applications

**Response 201 Created**:
```json
{
  "id": 5,
  "applicant": {
    "id": 3,
    "name": "John Doe",
    "email": "john@example.com"
  },
  "job": {
    "id": 2,
    "title": "Senior Backend Developer"
  },
  "score": "0.00",
  "status": "pending",
  "date": "2025-12-29",
  "has_resume": true,
  "resume_url": "http://127.0.0.1:8000/api/applications/5/resume/"
}
```

**Database Tables Modified**:
- `applicants_applicant`: May create or update applicant
- `applications_application`: Creates new application record

---

### 3.2 List Applications

**Endpoint**: `GET /api/applications/` or `GET /api/applications/?job=<job_id>&status=<status>`

**Handler**: `server/applications/views.py::ApplicationListCreateAPIView.get_queryset()` (Line 31-52)

**Query Parameters**:
| Parameter | Type | Description |
|-----------|------|-------------|
| `job` | integer | Optional. Filter by job ID |
| `status` | string | Optional. Filter by status (pending/shortlisted/rejected/hired) |

**Business Logic**:
1. Base query: `Application.objects.all().order_by("-date")`
2. If `job` provided: Filter `queryset.filter(job_id=job_id)`
3. If `status` provided: Filter `queryset.filter(status=status.lower())`

**Response 200 OK**:
```json
[
  {
    "id": 5,
    "applicant": {...},
    "job": {...},
    "score": "85.50",
    "status": "shortlisted",
    "date": "2025-12-29",
    "has_resume": true,
    "resume_url": "http://127.0.0.1:8000/api/applications/5/resume/"
  }
]
```

---

### 3.3 Update Application (Change Status/Score)

**Endpoint**: `PATCH /api/applications/<id>/`

**Handler**: `server/applications/views.py::ApplicationRetrieveUpdateDestroyAPIView.update()` (Line 111-130)

**Purpose**: Updates application status or score (typically used by recruiters to shortlist/reject candidates or update AI scores).

**Request Content-Type**: `application/json` or `multipart/form-data`

**Request Body (JSON)**:
```json
{
  "status": "shortlisted",
  "score": "92.5"
}
```

**Business Logic**:
- Uses `PATCH` for partial updates (only sends fields that changed)
- Validates status choices: pending, shortlisted, rejected, hired
- Score is DecimalField with max 5 digits, 2 decimal places

**Response 200 OK**:
```json
{
  "id": 5,
  "applicant": {...},
  "job": {...},
  "score": "92.50",
  "status": "shortlisted",
  "date": "2025-12-29",
  "has_resume": true,
  "resume_url": "http://127.0.0.1:8000/api/applications/5/resume/"
}
```

---

### 3.4 View/Download Resume

**Endpoint**: `GET /api/applications/<id>/resume/`

**Handler**: `server/applications/views.py::application_resume()` (Line 341-357)

**Purpose**: Retrieves resume file from database BLOB storage and serves it as PDF.

**Business Logic** (Line 346-357):
1. Fetch `Application` by ID: `get_object_or_404(Application, pk=pk)`
2. Check resume exists: `if not application.resume: return 404`
3. Create `HttpResponse` with resume bytes
4. Set `Content-Type: application/pdf`
5. Set `Content-Disposition: inline; filename="resume_{pk}.pdf"`
   - Browser displays PDF inline (not download)

**Response 200 OK**:
- Binary PDF content
- Headers:
  ```
  Content-Type: application/pdf
  Content-Disposition: inline; filename="resume_5.pdf"
  ```

**Error 404**: Resume not found

---

## 4. JOB POSTINGS

### 4.1 List Jobs

**Endpoint**: `GET /api/jobs/`

**Handler**: `server/posts/views.py::JobListCreateAPIView.get()` (Line 45-47)

**Response 200 OK**:
```json
[
  {
    "id": 2,
    "title": "Senior Backend Developer",
    "description": "We are looking for an experienced backend developer...",
    "status": "active",
    "date": "2025-12-20",
    "jobtype": "remote",
    "jobtime": "full-time",
    "shift": "Day",
    "required_skills": "Python, Django, PostgreSQL, REST APIs",
    "domain": "SaaS",
    "total_applicants": 15
  }
]
```

---

### 4.2 Create Job

**Endpoint**: `POST /api/jobs/`

**Handler**: `server/posts/views.py::JobListCreateAPIView.post()`

**Request Body (JSON)**:
```json
{
  "title": "Frontend Developer",
  "description": "Build modern React applications...",
  "status": "active",
  "jobtype": "onsite",
  "jobtime": "full-time",
  "shift": "Day",
  "required_skills": "React, TypeScript, Tailwind CSS",
  "domain": "E-commerce"
}
```

**Response 201 Created**: Returns created job object

---

## 5. DASHBOARD & STATISTICS

### 5.1 Customer Statistics

**Endpoint**: `GET /api/crm/customers-statistic`

**Handler**: `server/applications/views.py::CustomerStatisticAPIView.get()` (Line 137-211)

**Purpose**: Provides applicant statistics with growth metrics for dashboard widgets.

**Business Logic**:

1. **Total Applicants** (Line 139-141):
   ```python
   Applicant.objects.filter(applications__isnull=False).distinct().count()
   ```
   - Counts unique applicants who have submitted applications

2. **Shortlisted Count** (Line 144-146):
   ```python
   Applicant.objects.filter(applications__status='shortlisted').distinct().count()
   ```

3. **Growth Calculation** (Line 153-168):
   - Compares last 3 months vs previous 3 months
   - Formula: `((recent - previous) / previous) * 100`

**Response 200 OK**:
```json
{
  "totalCustomers": {
    "value": 125,
    "growShrink": 15.2
  },
  "activeCustomers": {
    "value": 45,
    "growShrink": -5.8
  },
  "newCustomers": {
    "value": 12,
    "growShrink": 20.0
  }
}
```

---

### 5.2 Sales Dashboard

**Endpoint**: `POST /api/sales/dashboard`

**Handler**: `server/applications/views.py::SalesDashboardAPIView.post()` (Line 218-338)

**Purpose**: Returns comprehensive dashboard data including statistics, recent applicants, jobs, and charts.

**Response 200 OK**:
```json
{
  "statisticData": {
    "revenue": {"value": 125, "growShrink": 15.2},
    "orders": {"value": 8, "growShrink": 33.3},
    "purchases": {"value": 12, "growShrink": 20.0}
  },
  "latestApplicantData": [
    {
      "name": "John Doe",
      "email": "john@example.com",
      "status": "shortlisted",
      "telephone": "123-456-7890",
      "job": "Backend Developer",
      "score": 92.5
    }
  ],
  "jobData": [
    {
      "title": "Backend Developer",
      "pay": 0,
      "description": "...",
      "status": "active",
      "date": "2025-12-20",
      "jobtype": "remote",
      "jobtime": "full-time",
      "shift": "Day",
      "reqskills": ["Python", "Django"],
      "domain": "SaaS",
      "totalapplicants": 15
    }
  ],
  "salesReportData": {
    "series": [{
      "name": "Applications",
      "data": [10, 15, 20, 25, 30, 35]
    }],
    "categories": ["Jul", "Aug", "Sep", "Oct", "Nov", "Dec"]
  },
  "recruitmentByCategoriesData": {
    "labels": ["Hired", "Shortlisted", "Pending", "Rejected"],
    "data": [12, 45, 50, 18]
  }
}
```


---

## Application and Job APIs (Extended Guidance)

### Background Theory
Applications form a many-to-one relationship between applicants and job postings. Each application encapsulates resume artifacts, status metadata, and derived scores. This model enables lifecycle tracking and analytics.

### Practical Example: Submitting an Application
```http
POST /api/applications/
Content-Type: multipart/form-data

fields:
  job_id: 42
  name: "Jane Doe"
  email: "jane@example.com"
  resume: resume.pdf
```
**Expected Response**: `201 Created` with application ID and resume URL.

---

## Use Cases

### Use Case: Automated Shortlisting
- **Actors**: Recruiter, Scoring service
- **Goal**: Prioritize candidates based on AI score
- **Flow**: Application submitted → Score computed → Status updated → Recruiter review

### Use Case: Job Posting Lifecycle
- **Goal**: Publish, close, and archive job postings
- **Flow**: Create job → Set status to `open` → Close → Archive for analytics

---

## Best Practices

- **Validate resumes** for size and MIME type before storage.
- **Make status transitions explicit** (e.g., `applied → reviewed → shortlisted`).
- **Log scoring metadata** to support transparency and tuning.

---

## Limitations

- Resume files are stored in the database as BLOBs, which can increase DB size over time.
- Score calculation depends on external services (OpenAI, Qdrant), impacting latency.

---

## FAQ

**Q: How are application scores updated?**  
A: A `PATCH /api/applications/<id>/` call can update the score based on the MLOps calculation.

**Q: Can applications be deleted?**  
A: The current API favors status updates. Hard deletes can be added with appropriate audit controls.

---

## Future Considerations

- **Asynchronous scoring** via background workers.
- **Versioned scoring models** to compare historical scores.
- **File storage migration** to object storage (S3/GCS) for scale.
