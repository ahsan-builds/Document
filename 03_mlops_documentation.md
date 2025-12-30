# Hiro MLOps - AI Resume Extraction & Scoring

## 1. RESUME EXTRACTION PIPELINE

### Entry Point
**Function**: `extract_resume_insights(resume_bytes, filename)` in `mlops/rag/resume_extractor.py` (Line 95-186)

### Step-by-Step Process

#### Step 1: Document Loading (`mlops/rag/utils.py::load_single_file`)

**PDF Processing** extracts text AND hyperlinks:
- Uses `pypdf.PdfReader` to read PDF pages
- Extracts annotations (`/Annots`) to find embedded URLs
- Appends LinkedIn/GitHub links to text as "Extracted Links"

#### Step 2: LLM-Based Extraction

**Model**: OpenAI GPT-4o-mini (temperature=0 for deterministic output)

**Structured Output Schema**:
```python
{
  "summary": "2-3 sentence professional summary",
  "github_url": "https://github.com/username",
  "linkedin_url": "https://www.linkedin.com/in/username",
  "skills": [
    {"name": "Python", "category": "Programming", "proficiency": "Advanced"}
  ],
  "experience": [
    {"company": "Tech Corp", "title": "Developer", "duration": "2020-2023"}
  ],
  "education": [
    {"institution": "University", "degree": "Bachelor's", "field": "CS", "year": "2020"}
  ],
  "certifications": [],
  "total_experience_years": 3.0
}
```

**Prompt Instructions**:
- Categorize skills (Programming, Design, Management, Communication)
- Calculate total years of experience
- Extract social media links from "Extracted Links" section
- Generate professional summary highlighting key strengths

---

## 2. RESUME SCORING ALGORITHM

### Entry Point
**Function**: `rank_resume_against_job(file_path)` in `mlops/rag/utils.py` (Line 118-175)

### Step-by-Step Process

#### Step 1: Text Chunking (Line 123-124)
```python
documents = load_single_file(file_path)
text_chunks = create_chunks(documents)  # 500 char chunks, 50 char overlap
```

#### Step 2: Embed Resume Chunks (Line 129-137)
```python
embedding_model = OpenAIEmbeddings(model="text-embedding-3-large")
resume_vectors = [
    {"text": chunk.page_content, "vector": embedding_model.embed_query(chunk.page_content)}
    for chunk in text_chunks
]
```

#### Step 3: Retrieve Job Description from Qdrant (Line 140-148)
```python
job_qdrant = QdrantVectorStore(client, collection_name="job_descriptions")
job_docs = job_qdrant.similarity_search("job description", k=1)
job_vector = embedding_model.embed_query(job_docs[0].page_content)
```

#### Step 4: Category-Based Scoring (Line 153-165)

**Category Detection** (`detect_category` function, Line 76-87):
- **experience**: Keywords like "experience", "worked", "developer"
- **skills**: "skill", "technologies", "tools", "languages"
- **projects**: "project", "built", "created"
- **education**: "bachelor", "degree", "education"
- **institute**: Default category

**Similarity Calculation** (Line 161-162):
```python
sim = cosine_similarity([job_vector], [chunk_vector])[0][0]
sim = (sim + 1) / 2  # Normalize from [-1,1] to [0,1]
```

#### Step 5: Weighted Final Score (Line 167-172)

**Category Weights** (`mlops/rag/config.py`, Line 41-47):
```python
CATEGORY_WEIGHTS = {
    "experience": 0.05,  # 5%
    "skills": 0.25,      # 25%
    "projects": 0.50,    # 50% - highest weight
    "education": 0.10,   # 10%
    "institute": 0.10,   # 10%
}
```

**Formula**:
```python
final_score = 0
for category, similarities in category_scores.items():
    avg_similarity = sum(similarities) / len(similarities)
    final_score += avg_similarity * CATEGORY_WEIGHTS[category]

final_score = round(final_score * 100, 2)  # Convert to 0-100 scale
```

---

## 3. LINKEDIN SCRAPING

### Entry Point
**Class**: `LinkedInScraper` in `server/applicants/scrapers.py` (Line 18-504)

### Rate Limiting Mechanism

**Model**: `LinkedInScrapingActivity` (`server/applicants/models.py`, Line 118-154)

```python
@classmethod
def can_scrape_today(cls, daily_limit=50):
    today_count = cls.get_today_count()  # Counts successful scrapes since midnight
    return today_count < daily_limit
```

**Enforcement**: `server/applicants/views.py::scrape_social_profile()` (Line 165-171) checks before scraping

### Scraping Logic (Pseudocode)
1. Uses Selenium WebDriver to navigate to LinkedIn profile
2. Waits for page load (handles dynamic content)
3. Extracts sections: headline, about, experience, education
4. Returns structured JSON
5. Saves to `ApplicantProfile.social_insights` field

---

## 4. CONFIGURATION

### Environment Variables (`.env`)
```
OPENAI_API_KEY=sk-...
QDRANT_URL=https://your-cluster.qdrant.io
QDRANT_API_KEY=your-api-key
JOB_COLLECTION=job_descriptions
```

### Model Selection

**Embeddings**: `text-embedding-3-large` (OpenAI)
- Dimension: 3072
- Cost-effective for production

**LLM**: `gpt-4o-mini`
- Fast inference (~1-2 seconds)
- Structured JSON output via Pydantic
- Lower cost than GPT-4


---

## Theoretical Background

### Retrieval-Augmented Generation (RAG)
RAG combines information retrieval with generative models. Instead of relying solely on model parameters, relevant text is retrieved, embedded, and fed into the model, yielding more grounded outputs.

### Vector Embeddings and Similarity
Embeddings represent text as high-dimensional vectors. Similarity (often cosine similarity) quantifies closeness between resume content and job descriptions, enabling ranking.

### Structured Extraction with LLMs
Structured extraction uses a schema-driven prompt and a parser. This ensures predictable fields and types for downstream processing.

---

## Practical Examples

### Example: Resume Extraction Output (Simplified)
```json
{
  "summary": "Data analyst with 5 years...",
  "skills": [{"name": "Python", "level": "advanced"}],
  "experience": [
    {"company": "Acme", "title": "Analyst", "years": 3}
  ]
}
```

### Example: Scoring Calculation (Conceptual)
- Projects similarity: 0.82 × 50%
- Skills similarity: 0.78 × 25%
- Experience similarity: 0.70 × 15%
- Education similarity: 0.65 × 10%
**Final score**: 0.77 (77/100)

---

## Use Cases

### Use Case: Automated Resume Parsing
- **Goal**: Convert unstructured resumes into structured profiles for search and analytics.
- **Outcome**: Consistent profiles across applicants, searchable by skill and experience.

### Use Case: Job-Candidate Matching
- **Goal**: Rank candidates by relevance to a job description.
- **Outcome**: Prioritized candidate lists for recruiter review.

---

## Best Practices

- **Prompt versioning**: Store prompt templates with version IDs.
- **Schema validation**: Fail fast on invalid AI output and capture raw responses.
- **Sampling control**: Use temperature=0 for determinism in extraction tasks.
- **Cost management**: Limit input size with truncation and summarize long sections.

---

## Limitations

- **Hallucinations**: LLMs can infer details not present in the resume.
- **File diversity**: Complex resume layouts may be parsed inconsistently.
- **Latency**: External model calls add response time variability.

---

## FAQ

**Q: Why use GPT-4o-mini instead of a larger model?**  
A: It balances cost, latency, and accuracy for structured extraction.

**Q: How are embeddings stored?**  
A: Resume and job embeddings are stored in Qdrant for fast similarity search.

---

## Future Considerations

- **Human-in-the-loop review** for low-confidence extractions.
- **Adaptive scoring** using historical recruiter feedback.
- **Model fine-tuning** to improve domain-specific extraction accuracy.
- **Multilingual resume support** via language detection and translation.
