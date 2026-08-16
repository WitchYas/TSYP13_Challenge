# Job Matcher Module

## Overview
The **Job Matcher** is a FastAPI-based microservice that matches extracted CV data against job opportunities using semantic similarity and vector embeddings. It leverages HuggingFace embeddings and Chroma vector database for intelligent job matching and scoring.

## Purpose
- **Input**: Extracted CV data from Firestore
- **Process**: Generate embeddings and match against job database
- **Output**: Ranked list of matching job opportunities with similarity scores

## Key Features
- **Semantic Matching**: Uses E5-Large-V2 embeddings for deep semantic understanding
- **Vector Database**: Chroma for efficient similarity search
- **Text Preprocessing**: Removes noise (punctuation, numbers, extra spaces)
- **Scoring System**: Computes relevance scores between CV and job requirements
- **Real-time Matching**: Direct Firestore integration for live job data
- **Scalable Search**: Handles large job datasets efficiently

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `GCP_PROJECT_ID` | Google Cloud Project ID | `tsyp-477814` |
| `FIRESTORE_DATABASE_ID` | Firestore Database ID | `tsypstore` |

## Installation & Setup

### Prerequisites
- Python 3.9+
- GCP Service Account with Firestore read access
- Pre-populated Firestore with job listings
- ~2GB disk space for Chroma vector database

### Install Dependencies
```bash
pip install -r requirements.txt
```

### Docker Build
```bash
docker build -t job-matcher .
docker run job-matcher
```

### Initialize Vector Database
The vector database is built on first run from Firestore job data:
```bash
python -c "from app import vector_store; print('Vector store initialized')"
```

## API Endpoints

### POST `/match-cv`
Finds jobs matching a candidate's CV profile.

**Request Body:**
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "top_k": 10
}
```

**Response:**
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "matches": [
    {
      "job_id": "job-789",
      "title": "Senior Python Developer",
      "company": "Tech Corp",
      "similarity_score": 0.92,
      "skills_match": ["Python", "FastAPI", "Cloud"],
      "skills_gap": ["Kubernetes"]
    },
    {
      "job_id": "job-790",
      "title": "Data Engineer",
      "company": "Data Inc",
      "similarity_score": 0.85,
      "skills_match": ["Python", "SQL"],
      "skills_gap": ["Apache Spark"]
    }
  ]
}
```

### GET `/health`
Health check endpoint.

**Response:**
```json
{
  "status": "healthy",
  "vector_store": "ready"
}
```

## Configuration Details

### Embedding Model
- **Model**: `intfloat/e5-large-v2`
- **Vector Dimension**: 1024
- **Provider**: HuggingFace

### Vector Store
- **Type**: Chroma (persistent)
- **Directory**: `chroma_jobs_db/`
- **Collection**: `intfloat_e5_large_v2`

### Job Data Schema
Each job document in Firestore contains:
```json
{
  "title": "Job Title",
  "company": "Company Name",
  "company_description": "Company overview",
  "requirements": "Required qualifications",
  "skills": ["skill1", "skill2", ...],
  "location": {
    "city": "City",
    "country": "Country"
  },
  "contract_type": "Full-time|Contract|Part-time",
  "experience_level": "Junior|Mid|Senior",
  "education_level": "Bachelor|Master|PhD"
}
```

### Text Preprocessing
- Converts to lowercase
- Removes punctuation and special characters
- Removes numbers
- Removes extra whitespace

## Firestore Collections
- **jobs**: Contains all job postings
- **cv_matches**: Stores matching results for later analysis

## Dependencies
See [requirements.txt](requirements.txt) for full dependency list.

### Key Libraries
- `fastapi` - Web framework
- `chroma-db` - Vector database
- `sentence-transformers` - Embeddings (HuggingFace)
- `google-cloud-firestore` - Firestore integration
- `langchain-community` - Vector store interface

## Deployment
Deploy to Google Cloud Run:
```bash
gcloud run deploy job-matcher \
  --source . \
  --platform managed \
  --region us-central1 \
  --memory 2Gi \
  --timeout 300
```

## Performance Considerations
- **Latency**: ~200-500ms per match request
- **Throughput**: Handles 10+ concurrent matches
- **Memory**: ~2GB for Chroma vector store
- **Scaling**: Auto-scales with Cloud Run

## Error Handling
- Returns 400 for invalid CV data format
- Returns 404 if CV not found in Firestore
- Returns 500 for vector database errors
- Includes error messages for debugging

## Monitoring
- Request latency tracked via Cloud Trace
- Similarity score distribution monitored
- Vector store health checked regularly
- Error rates tracked via Cloud Monitoring

## Related Modules
- **Extractor**: Provides CV data for matching
- **Rewriter**: Uses job matches to tailor CVs
- **Trigger Function**: Initiates entire workflow

