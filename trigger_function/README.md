# Trigger Function Module

## Overview
The **Trigger Function** is a Google Cloud Function that serves as the entry point for the entire CV processing pipeline. It's triggered by file uploads to Google Cloud Storage, extracts metadata, updates processing status, and orchestrates the extraction service.

## Purpose
- **Trigger**: Automatically activates on PDF upload to GCS
- **Orchestration**: Manages workflow initiation and status tracking
- **Communication**: Calls the Extractor service to begin processing
- **Status Tracking**: Updates Firestore with real-time processing status

## Key Features
- **Event-Driven Architecture**: Automatically triggers on file uploads
- **Serverless Execution**: No infrastructure management required
- **Metadata Extraction**: Parses user_id and cv_id from file path
- **Status Updates**: Real-time status tracking in Firestore
- **Error Handling**: Graceful error logging and status updates
- **Asynchronous Processing**: Non-blocking call to Extractor service
- **Audit Trail**: Complete logging of all operations

## Installation & Setup

### Prerequisites
- Google Cloud Project with Cloud Functions enabled
- Service account with permissions:
  - Firestore write access
  - Cloud Run Invoker (to call Extractor)
- GCS bucket configured with object finalize trigger

### Deploy to Google Cloud Functions
```bash
gcloud functions deploy trigger_cv_extraction \
  --runtime python311 \
  --trigger-resource YOUR_GCS_BUCKET \
  --trigger-event google.storage.object.finalize \
  --entry-point trigger_cv_extraction \
  --set-env-vars EXTRACTION_SERVICE_URL="https://cv-extractor-url" \
  --service-account YOUR_SERVICE_ACCOUNT
```

### Local Testing
```bash
pip install functions-framework
functions-framework --target trigger_cv_extraction --debug --port 8080
```

Then simulate a GCS event:
```bash
curl -X POST http://localhost:8080/ \
  -H "Content-Type: application/json" \
  -d '{
    "context": {"eventId": "test-id"},
    "data": {
      "bucket": "your-bucket",
      "name": "user-123/cv-456.pdf"
    }
  }'
```

## GCS Event Structure

### Trigger Configuration
- **Event Type**: `google.storage.object.finalize`
- **Bucket**: Your CV storage bucket
- **File Pattern**: `{user_id}/{cv_id}.pdf`

### Example File Path
```
gs://tsypbucket/user-123/cv-456.pdf
```

Where:
- `user-123` = user_id (first directory level)
- `cv-456.pdf` = cv_id (filename)

## Workflow & Data Flow

### 1. File Upload Triggers Function
```
User uploads CV to GCS → Cloud Functions triggered
```

### 2. Extract Metadata
```
File path: gs://bucket/user-123/cv-456.pdf
↓
user_id = "user-123"
cv_id = "cv-456"
```

### 3. Update Status to "Processing"
```
Firestore: cv_status/{user_id}_{cv_id}
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "status": "processing"
}
```

### 4. Call Extractor Service
```
POST https://cv-extractor-url/extract
{
  "gcs_path": "gs://bucket/user-123/cv-456.pdf",
  "user_id": "user-123"
}
```

### 5. Update Final Status
```
On Success:
  status = "done"
  
On Failure:
  status = "failed"
  error = "Error message"
```

## Firestore Schema

### Status Collection
```
Collection: cv_status
Document ID: {user_id}_{cv_id}

Fields:
  user_id: string
  cv_id: string
  status: "processing" | "done" | "failed"
  error: string (optional, on failure)
  timestamp_created: timestamp
  timestamp_completed: timestamp (optional)
```

## Error Handling

### Ignored Events
- Non-PDF files: Logged as info, silently ignored
- Missing metadata: Logged and skipped

### Processing Errors
- Extraction service unreachable: Logs error, updates status to "failed"
- Network timeout: Retries automatically (Cloud Functions default)
- Invalid response: Logs detailed error message

### Status Updates on Errors
```json
{
  "user_id": "user-123",
  "cv_id": "cv-456",
  "status": "failed",
  "error": "HTTP 500: Service unavailable"
}
```

## Configuration Details

### Service Account Permissions
Required roles:
- `roles/datastore.user` - Firestore read/write
- `roles/run.invoker` - Call Cloud Run services
- `roles/logging.logWriter` - Write to Cloud Logging

### Timeout & Retry
- **Function Timeout**: 60 seconds (default)
- **HTTP Call Timeout**: 30 seconds (to Extractor)
- **Auto-Retry**: Up to 2 times on transient failures

## API Integration

### Extractor Service Contract
**Endpoint**: `POST {EXTRACTION_SERVICE_URL}/extract`

**Request:**
```json
{
  "gcs_path": "gs://bucket/user-id/file.pdf",
  "user_id": "user-id"
}
```

**Expected Response (Success - 200):**
```json
{
  "status": "success",
  "user_id": "user-123",
  "cv_id": "cv-456"
}
```

**Expected Response (Error - 4xx/5xx):**
```json
{
  "error": "Error message",
  "details": "Additional error context"
}
```

## Monitoring & Logging

### Cloud Logging
All operations logged with severity levels:
- **INFO**: File processed, status updated
- **WARNING**: Non-PDF file ignored
- **ERROR**: Processing failed, extraction failed

### Example Log Entry
```
Extraction done for user-123/cv-456.pdf
Status updated: processing → done
```

### Cloud Monitoring
- Function execution count tracked
- Error rate monitored
- Average latency tracked
- Duration per invocation logged

## Related Modules
- **Extractor**: Service called to process CV
- **Job Matcher**: Uses extracted data
- **Rewriter**: Uses extracted data for tailoring
- **Frontend**: Calls status endpoint via Firestore listeners


