# CV Extractor Module

## Overview
The **CV Extractor** is a Flask-based microservice that parses PDF and DOCX CV files, extracts structured data, and stores the results in Firestore. It leverages LangChain for text processing and Large Language Models (Google GenAI and OpenAI via OpenRouter) for intelligent data extraction.

## Purpose
- **Input**: PDF/DOCX files from Google Cloud Storage
- **Process**: Extract and structure CV data (personal info, skills, experience, education, etc.)
- **Output**: Structured JSON data stored in Firestore

## Key Features
- **Multi-format Support**: Handles both PDF and DOCX documents
- **Intelligent Parsing**: Uses LLMs for accurate data extraction and structuring
- **Text Processing**: Recursive text chunking for large documents
- **Vector Embeddings**: Generates embeddings using HuggingFace models
- **Data Validation**: Cleans and validates extracted JSON data
- **Cloud Integration**: Directly reads from GCS and writes to Firestore


## Installation & Setup

### Prerequisites
- Python 3.9+
- GCP Service Account with credentials configured
- GCS bucket for CV storage
- Firestore database initialized

### Install Dependencies
```bash
pip install -r requirements.txt
```

### Docker Build
```bash
docker build -t cv-extractor .
docker run -e GOOGLE_API_KEY="..." \
           -e OPENAI_API_KEY="..." \
           -e HF_TOKEN="..." \
           cv-extractor
```

## API Endpoints

### POST `/extract`
Extracts structured data from a CV file.

**Request Body:**
```json
{
  "gcs_path": "gs://bucket-name/user-id/cv.pdf",
  "user_id": "user-123"
}
```

**Response:**
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "status": "success",
  "extracted_data": {
    "personal_information": {
      "name": "John Doe",
      "email": "john@example.com",
      "phone": "+1234567890"
    },
    "skills": ["Python", "JavaScript", "Cloud Computing"],
    "experience": [...],
    "education": [...]
  }
}
```

## Configuration Details

### Firestore Schema
Extracted CV data is stored at: `users/{user_id}/cvs/{cv_id}`

### Text Processing
- **Splitter**: RecursiveCharacterTextSplitter (chunk_size=1000, overlap=100)
- **Embeddings**: HuggingFace E5-Large-V2 model
- **Vector Store**: Chroma (local persistence)

### LLM Models Used
- **Google GenAI**: For initial data extraction
- **OpenRouter (Alibaba DeepResearch)**: For refinement and validation

## Security Considerations
- API keys stored in environment variables only
- GCS bucket access via IAM roles
- Firestore rules enforce authenticated access
- Data encrypted at rest and in transit

## Dependencies
See [requirements.txt](requirements.txt) for full dependency list.

### Key Libraries
- `flask` - Web framework
- `langchain` - Text processing and LLM orchestration
- `google-cloud-storage` - GCS integration
- `google-cloud-firestore` - Firestore integration
- `pypdf` - PDF parsing
- `python-docx` - DOCX parsing
- `google-genai` - Google Generative AI
- `langchain-openai` - OpenAI/OpenRouter integration

## Deployment
Deploy to Google Cloud Run:
```bash
gcloud run deploy cv-extractor \
  --source . \
  --platform managed \
  --region us-central1 \
  --set-env-vars GOOGLE_API_KEY="...",OPENAI_API_KEY="...",HF_TOKEN="..."
```

## Error Handling
- Returns 400 for invalid file formats
- Returns 500 for extraction failures with error details
- Logs all failures for debugging

## Monitoring
- Container logs available via Cloud Logging
- Error rates tracked via Cloud Monitoring
- Extraction performance metrics stored in Firestore

## Related Modules
- **Trigger Function**: Initiates extraction on CV upload
- **Job Matcher**: Uses extracted data for job matching
- **CV Rewriter**: Enhances extracted data for specific jobs
