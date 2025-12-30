# Hiro - Data Flow & Lifecycle Analysis

## 1. USER REGISTRATION & AUTHENTICATION LIFECYCLE

### Input: Registration Form

**Entry Point**: Frontend registration page

**Data Captured**:
```typescript
{
  username: string,
  email: string,
  password: string,
  first_name?: string,
  last_name?: string,
  telephone?: string,
  department?: string,
  position?: string
}
```

### Processing Flow

**Step 1: Frontend Validation** (`Formik + Yup`)
- Validates email format
- Ensures password length >= 8 characters
- Confirms password_confirm matches password

**Step 2: API Call** (`AuthService.apiSignUp()`)
- POST request to `http://127.0.0.1:8000/users/register/`
- Content-Type: `application/json`
- Body contains user registration data

**Step 3: Backend Serializer** (`users/serializers.py::UserRegisterSerializer`)
```python
class UserRegisterSerializer:
    def validate(self, data):
        if data['password'] != data['password_confirm']:
            raise ValidationError("Passwords don't match")
        return data
    
    def create(self, validated_data):
        # 1. Create User
        user = User.objects.create_user(
            username=validated_data['username'],
            email=validated_data['email'],
            first_name=validated_data.get('first_name', ''),
            last_name=validated_data.get('last_name', '')
        )
        # 2. Hash password
        user.set_password(validated_data['password'])
        user.save()
        
        # 3. Create UserProfile
        profile = UserProfile.objects.create(
            user=user,
            telephone=validated_data.get('telephone', ''),
            department=validated_data.get('department', ''),
            position=validated_data.get('position', ''),
            password_last_changed=timezone.now()
        )
        return user
```

### Storage Mapping

**Database Tables**:
1. **`auth_user`** (Django built-in):
   - `id`: Auto-incremented primary key
   - `username`: Unique username
   - `email`: Email address
   - `password`: Hashed using PBKDF2 (e.g., `pbkdf2_sha256$600000$...`)
   - `first_name`, `last_name`: Name fields
   - `is_active`: Default `True`
   - `date_joined`: Auto-set timestamp

2. **`users_userprofile`**:
   - `id`: Auto-incremented primary key
   - `user_id`: Foreign key to `auth_user.id`
   - `telephone`: Phone number
   - `avatar`: File path (nullable)
   - `department`: Department name
   - `position`: Job position
   - `language`: Default `'en'`
   - `timezone`: Default `'UTC'`
   - `password_last_changed`: Timestamp

### Output: Response to Frontend

```json
{
  "id": 2,
  "username": "farhan1",
  "first_name": "Farhan",
  "last_name": "Dev",
  "email": "farhan@example.com",
  "profile": {
    "telephone": "03214567890",
    "avatar": null,
    "department": "Engineering",
    "position": "Backend Developer",
    "language": "en",
    "timezone": "Asia/Karachi",
    "password_last_changed": "2025-11-24T05:43:45.102880Z"
  }
}
```

**Frontend State Update**:
- Redux stores user object in `auth.user`
- Persisted to localStorage via redux-persist
- Redirects to dashboard

---

## 2. JOB APPLICATION SUBMISSION LIFECYCLE

### Input: Application Form with Resume Upload

**Entry Point**: Frontend job application page

**Data Captured**:
```typescript
{
  name: "John Doe",
  email: "john@example.com",
  job: 2,  // Job ID
  resume_file: File  // PDF/DOCX file object
}
```

### Processing Flow

**Step 1: Frontend File Handling**
```typescript
const formData = new FormData()
formData.append('name', 'John Doe')
formData.append('email', 'john@example.com')
formData.append('job', '2')
formData.append('resume_file', fileInput.files[0])  // Binary file
```

**Step 2: API Call** (`ApplicationService.createApplication()`)
- POST to `http://127.0.0.1:8000/api/applications/`
- Content-Type: `multipart/form-data` (automatic with FormData)
- Body: FormData object with file

**Step 3: Backend Serializer** (`applications/serializers.py::ApplicationCreateSerializer`)

**Get or Create Applicant**:
```python
def create(self, validated_data):
    # 1. Get or create applicant by email
    applicant, created = Applicant.objects.get_or_create(
        email=validated_data['email'],
        defaults={'name': validated_data['name']}
    )
    
    if not created:
        # Update name if applicant already exists
        applicant.name = validated_data['name']
        applicant.save()
```

**Read Resume Binary Data**:
```python
    # 2. Read file bytes
    resume_file = validated_data.get('resume_file')
    resume_bytes = resume_file.read() if resume_file else None
```

**Create Application**:
```python
    # 3. Create application record
    application = Application.objects.create(
        applicant=applicant,
        job=validated_data['job'],
        resume=resume_bytes,  # Stored as PostgreSQL bytea
        score=validated_data.get('score', 0),
        status=validated_data.get('status', 'pending')
    )
    return application
```

### Storage Mapping

**Database Tables**:

1. **`applicants_applicant`**:
   - `id`: 3
   - `name`: "John Doe"
   - `email`: "john@example.com" (unique)
   - `telephone`: ""
   - `prior_experience`: ""
   - `source`: ""
   - `skill_set`: ""

2. **`applications_application`**:
   - `id`: 5
   - `applicant_id`: 3 (FK to applicants_applicant)
   - `job_id`: 2 (FK to posts_job)
   - `resume`: Binary data (bytea) - raw PDF/DOCX bytes
   - `score`: 0.00
   - `status`: "pending"
   - `date`: "2025-12-29" (auto-set)

**Unique Constraint**: `unique_together = ['applicant', 'job']` prevents duplicate applications

### Output: Response to Frontend

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

---

## 3. RESUME EXTRACTION & PROFILE CREATION LIFECYCLE

### Input: GET Request for Applicant Profile

**Trigger**: Frontend requests `GET /api/applicants/3/profile/`

### Processing Flow

**Step 1: Check Profile Exists** (`applicants/views.py::get_applicant_profile`)
```python
try:
    profile = applicant.profile  # OneToOne relationship
    return Response(ApplicantProfileSerializer(profile).data)
except ApplicantProfile.DoesNotExist:
    # Trigger extraction
    return trigger_profile_extraction(applicant)
```

**Step 2: Retrieve Resume Binary** (`trigger_profile_extraction`)
```python
application = applicant.applications.filter(
    resume__isnull=False
).exclude(resume=b'').order_by('-date').first()

resume_bytes = application.resume  # Binary PDF data
```

**Step 3: MLOps Extraction** (`rag/resume_extractor.py`)

**Temp File Creation**:
```python
with tempfile.NamedTemporaryFile(suffix='.pdf', delete=False) as tmp:
    tmp.write(resume_bytes)
    tmp_path = tmp.name
```

**Document Loading** (`rag/utils.py::load_single_file`):
```python
reader = pypdf.PdfReader(tmp_path)
documents = []
for page in reader.pages:
    text = page.extract_text()
    # Extract hyperlinks
    links = extract_annotations(page)
    if links:
        text += "\n\nExtracted Links:\n" + "\n".join(links)
    documents.append(Document(page_content=text))
```

**LLM Extraction**:
```python
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)
parser = JsonOutputParser(pydantic_object=ResumeInsights)
chain = prompt | llm | parser

result = chain.invoke({
    "resume_text": resume_text,
    "format_instructions": parser.get_format_instructions()
})
# Returns structured JSON matching ResumeInsights schema
```

**Step 4: Database Write** (`trigger_profile_extraction`)
```python
profile, created = ApplicantProfile.objects.update_or_create(
    applicant=applicant,
    defaults={
        'extraction_source': f"Application #{application.id}",
        'skills': insights.get('skills', []),
        'experience': insights.get('experience', []),
        'education': insights.get('education', []),
        'certifications': insights.get('certifications', []),
        'summary': insights.get('summary', ''),
        'total_experience_years': insights.get('total_experience_years'),
        'github_url': insights.get('github_url'),
        'linkedin_url': insights.get('linkedin_url'),
        'raw_extraction': insights
    }
)
```

### Storage Mapping

**Database Table**: `applicants_applicantprofile`
- `id`: Auto-incremented
- `applicant_id`: 3 (FK to applicants_applicant, OneToOne)
- `extracted_at`: Auto-set timestamp
- `extraction_source`: "Application #5"
- `skills`: JSON array `[{"name": "Python", "category": "Programming", "proficiency": "Advanced"}]`
- `experience`: JSON array of work history
- `education`: JSON array of education records
- `certifications`: JSON array
- `summary`: Text field with professional summary
- `total_experience_years`: Decimal (e.g., 3.5)
- `github_url`: URL or NULL
- `linkedin_url`: URL or NULL
- `social_insights`: JSON object (empty initially, populated by scraper)
- `raw_extraction`: Complete JSON response from LLM

### Output: Response to Frontend

```json
{
  "id": 1,
  "applicant": 3,
  "applicant_name": "John Doe",
  "applicant_email": "john@example.com",
  "extracted_at": "2025-12-29T15:30:00Z",
  "extraction_source": "Application #5",
  "skills": [
    {"name": "Python", "category": "Programming", "proficiency": "Advanced"},
    {"name": "Django", "category": "Programming", "proficiency": "Intermediate"}
  ],
  "experience": [...],
  "education": [...],
  "certifications": [],
  "summary": "Experienced backend developer...",
  "total_experience_years": 3.5,
  "github_url": "https://github.com/johndoe",
  "linkedin_url": "https://www.linkedin.com/in/johndoe",
  "social_insights": {}
}
```

**Frontend Rendering**:
- Displays skills in categorized list
- Shows experience timeline
- Renders education cards
- Links to GitHub/LinkedIn if available

---

## 4. APPLICATION STATUS UPDATE LIFECYCLE

### Input: Status Change (Recruiter Action)

**Entry Point**: Frontend applicant detail page, recruiter clicks "Shortlist" button

**Data**:
```typescript
{
  application_id: 5,
  status: "shortlisted"
}
```

### Processing Flow

**Step 1: API Call** (`ApplicationService.updateApplication()`)
```typescript
const formData = new FormData()
formData.append('status', 'shortlisted')

await fetch(`http://127.0.0.1:8000/api/applications/5/`, {
    method: 'PATCH',  // Partial update for status-only change
    body: formData
})
```

**Step 2: Backend Update** (`applications/views.py::ApplicationRetrieveUpdateDestroyAPIView.update()`)
```python
instance = self.get_object()  # Fetches Application id=5
serializer = self.get_serializer(instance, data=request.data, partial=True)
serializer.is_valid(raise_exception=True)
serializer.save()  # Updates status field only
```

### Storage Mapping

**Database Update**: `applications_application`
```sql
UPDATE applications_application
SET status = 'shortlisted'
WHERE id = 5
```

### Output: Response to Frontend

```json
{
  "id": 5,
  "status": "shortlisted",
  ...
}
```

**Frontend State Update**:
- Updates local state/Redux store
- Refreshes applicant list with new status
- Status badge changes color (e.g., yellow → green)


---

## Data Lifecycle Principles

### Data Governance
- **Single source of truth** per domain (users, jobs, applications).
- **Auditability** with immutable log records (login history, status transitions).
- **Retention policies** for resumes and derived AI profiles.

### Data Quality
- **Validation at ingestion** (serializers, file type checks).
- **Normalization** for structured fields (skills, titles).
- **Error states** for failed extraction to avoid silent corruption.

---

## Practical Examples

### Example: Resume Reprocessing
1. Recruiter updates a resume file.
2. Application record is updated with new resume bytes.
3. Applicant profile is flagged for re-extraction.
4. New AI profile overwrites or version-tags the existing entry.

### Example: Status Transition Timeline
- `applied` → `reviewed` → `shortlisted` → `offer` → `hired`
- Each transition is timestamped and auditable.

---

## Use Cases

### Use Case: Compliance Reporting
- **Goal**: Demonstrate data access and changes for audits.
- **Flow**: Pull login history, application status changes, and profile extraction logs.

### Use Case: Analytics and KPIs
- **Goal**: Track time-to-hire and application conversion rates.
- **Flow**: Aggregate application statuses and timestamps.

---

## Best Practices

- **Event-driven updates** for extraction and scoring tasks.
- **Data anonymization** when exporting for analytics or testing.
- **Regular schema reviews** to ensure AI outputs align with business needs.

---

## Limitations

- Data flows are synchronous in several paths, which can increase latency.
- BLOB storage of resumes may complicate analytics pipelines.

---

## FAQ

**Q: How do we handle resume updates?**  
A: Replace the resume in the application record and trigger re-extraction.

**Q: What if AI output conflicts with user-provided data?**  
A: Favor user-entered data as the source of truth and flag discrepancies.

---

## Future Considerations

- **Event sourcing** for end-to-end state reconstruction.
- **CDC pipelines** for real-time analytics.
- **Data catalogs** and lineage tracking for compliance.
