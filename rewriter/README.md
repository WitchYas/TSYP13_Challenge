# CV Rewriter Module

## Overview
The **CV Rewriter** is a FastAPI-based microservice that takes extracted CV data and job matching results, then uses Large Language Models to rewrite and tailor the CV specifically for matched job opportunities. The output is a professionally formatted DOCX file optimized for applicant tracking systems (ATS) and human recruiters.

## Purpose
- **Input**: Extracted CV data from Firestore + matched job details
- **Process**: Rewrite CV content to emphasize relevant skills and experience
- **Output**: Professionally formatted DOCX file stored in GCS

## Key Features
- **AI-Powered Tailoring**: Uses LLMs to customize CV content for specific jobs
- **Professional Formatting**: Generates ATS-friendly DOCX documents
- **Skill Emphasis**: Highlights relevant skills and downplays irrelevant experience
- **Content Enhancement**: Improves wording and presentation
- **Cloud Storage**: Uploads finalized CVs to GCS for easy download
- **Streaming Response**: Supports direct file download without temporary storage


## Installation & Setup

### Prerequisites
- Python 3.9+
- GCP Service Account with Firestore read and GCS write access
- OpenRouter API key (for LLM access)
- python-docx library for document generation

### Install Dependencies
```bash
pip install -r requirements.txt
```

### Docker Build
```bash
docker build -t cv-rewriter .
docker run -e OPENAI_API_KEY="..." cv-rewriter
```

## API Endpoints

### POST `/rewrite`
Rewrites a CV tailored to a specific job match.

**Request Body:**
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "job_id": "job-789"
}
```

**Response:**
Returns a DOCX file with status 200, or JSON error response.

**Response Headers:**
```
Content-Type: application/vnd.openxmlformats-officedocument.wordprocessingml.document
Content-Disposition: attachment; filename="CV_John_Doe_SeniorDeveloper.docx"
```

### POST `/rewrite-and-upload`
Rewrites CV and uploads to GCS automatically.

**Request Body:**
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "job_id": "job-789"
}
```

**Response:**
```json
{
  "status": "success",
  "gcs_path": "gs://tsypbucket/user-123/rewritten_cv_job-789.docx",
  "download_url": "https://storage.googleapis.com/tsypbucket/..."
}
```

### GET `/status/{user_id}/{cv_id}`
Retrieves rewrite history for a CV.

**Response:**
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "rewrites": [
    {
      "job_id": "job-789",
      "job_title": "Senior Developer",
      "rewritten_at": "2024-01-15T10:30:00Z",
      "gcs_path": "gs://tsypbucket/..."
    }
  ]
}
```

## Configuration Details

### LLM Configuration
- **Provider**: OpenRouter
- **Model**: Alibaba DeepResearch 30B
- **Temperature**: 0.3 (deterministic output)
- **Base URL**: `https://openrouter.ai/api/v1`

### Firestore Schema
- **Input Source**: `users/{user_id}/cvs/{cv_id}/` (CV data)
- **Job Reference**: `jobs/{job_id}/` (Job details)
- **Output Log**: `users/{user_id}/cvs/{cv_id}/rewrites/{job_id}` (Rewrite history)

### Document Generation
- **Format**: DOCX (Microsoft Word)
- **Sections**: Contact, Summary, Skills, Experience, Education
- **Styling**: Professional formatting with proper spacing
- **ATS Compatibility**: Clean structure for parsing systems

### Rewrite Prompt Strategy
The LLM is instructed to:
1. Analyze job requirements and skills
2. Identify matching experience in CV
3. Rewrite descriptions to emphasize relevant skills
4. Improve wording and clarity
5. Maintain factual accuracy
6. Add quantifiable achievements where possible

## GCS Storage Structure
```
gs://tsypbucket/
  {user_id}/
    cv/{cv_id}.pdf                    (Original upload)
    rewrites/
      {job_id}_v1.docx               (Tailored CV for job)
      {job_id}_v2.docx               (Iteration)
```

## Dependencies
See [requirements.txt](requirements.txt) for full dependency list.

### Key Libraries
- `fastapi` - Web framework
- `python-docx` - DOCX document creation
- `google-cloud-storage` - GCS integration
- `google-cloud-firestore` - Firestore integration
- `langchain-openai` - LLM orchestration
- `langchain-core` - Prompt/chain management

## Deployment
Deploy to Google Cloud Run:
```bash
gcloud run deploy cv-rewriter \
  --source . \
  --platform managed \
  --region us-central1 \
  --memory 1Gi \
  --timeout 600 \
  --set-env-vars OPENAI_API_KEY="...",GCS_BUCKET_NAME="tsypbucket"
```

## Performance Considerations
- **Latency**: ~10-30 seconds per rewrite (LLM processing)
- **Throughput**: Handles 5-10 concurrent rewrites
- **Memory**: ~1GB for LLM interactions
- **Cost**: Depends on OpenRouter API pricing

## Error Handling
- Returns 400 for missing CV or job data
- Returns 404 if CV/job not found in Firestore
- Returns 500 for LLM or GCS errors
- Includes detailed error messages for debugging
- Automatic retry for transient failures

## Monitoring
- Request latency tracked via Cloud Trace
- Token usage monitored for cost control
- GCS upload success rate tracked
- Error patterns analyzed via Cloud Logging

## Quality Assurance
- Validates extracted CV data before rewriting
- Checks job posting completeness
- Verifies DOCX file integrity before upload
- Maintains rewrite audit trail in Firestore

## Related Modules
- **Extractor**: Provides CV data
- **Job Matcher**: Provides job matches for tailoring
- **Trigger Function**: Initiates workflow
