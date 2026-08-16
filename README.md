# CV Processing Platform - Global Architecture

## System Overview

This is a cloud-based, AI-driven CV processing and job matching platform built on **Google Cloud Platform (GCP)**. The system is designed for scalability, security, and modularity to support CV processing, user data management, job matching, and AI-driven interview preparation.

The architecture leverages GCP services with integrated enterprise identity management via **Auth0**, ensuring robustness, compliance, and seamless user experiences.

---

## Architecture Design Principles

### 1. **Zero Trust Model**
Every request—whether from user or service—is authenticated and authorized before access is granted.

### 2. **Defense in Depth**
Multiple layers of protection:
- IAM policies for granular access control
- VPC Service Controls for security perimeters
- Secret Manager for credential protection
- Encryption at rest and in transit
- Comprehensive audit logging

### 3. **Compliance-Ready**
The architecture is designed to meet regulatory requirements through:
- Data minimization
- Access controls
- Audit trails for forensic analysis

---

## Core Components

### 6.1.1 Google Cloud Storage (GCS)

**Purpose**: Centralized storage for raw and processed CV files

**Usage**:
- Stores original uploaded CVs (PDF/DOCX formats)
- Hosts rewritten or reformatted versions of CVs after processing
- Maintains organized directory structure by user: `gs://bucket/{user_id}/{cv_id}.pdf`

**Security**:
- Data is encrypted at rest (Google-managed keys) and in transit (HTTPS/TLS)
- Access is strictly controlled via Identity and Access Management (IAM) roles
- Public access is explicitly disabled
- Signed URLs for secure, time-limited access to files
- Audit logging tracks all access attempts

---

### 6.1.2 Cloud Run (Fully Managed)

**Purpose**: Serverless execution environment hosting microservices as containerized applications

**Deployed Services**:

1. **CV Extract API** (Cloud Run)
   - Parses structured data (skills, experience, education) from CVs
   - Outputs JSON-formatted extracted data
   - Integrates with LLMs (Google GenAI, OpenAI) for intelligent parsing
   - Stores results directly in Firestore

2. **CV Rewrite API** (Cloud Run)
   - Reformats and enhances CV content using NLP models
   - Tailors CVs for specific job opportunities
   - Generates professional DOCX files
   - Uploads finalized CVs to GCS for download

3. **Job Match & CV Scoring API** (Cloud Run)
   - Matches user profiles with relevant job opportunities
   - Computes fit scores using ML models and semantic embeddings
   - Uses vector databases (Chroma) for efficient similarity search
   - Supports at-scale job matching

4. **Interview Prep API** (Cloud Run) *[Future]*
   - Generates personalized mock interviews
   - Based on user profile and target job context
   - Stores interaction history in Firestore

**Features**:
- **Auto-Scaling**: Automatically scales based on incoming request volume (0 to N instances)
- **Stateless Containers**: Each instance is independent, enabling horizontal scaling
- **Security**:
  - Identity-Aware Proxy (IAP) for external access control
  - Mutual TLS for service-to-service authentication
  - Service accounts with minimal necessary permissions
  - All traffic encrypted end-to-end

---

### 6.1.3 Firestore (Native Mode)

**Purpose**: NoSQL database for flexible, real-time storage of structured user and job-related data

**Data Stored**:
- Parsed CV data in JSON format
- User profiles and preferences
- Job match results and scoring outputs
- Interview history and feedback logs
- Real-time processing status (for frontend listeners)

**Database Schema**:
```
/users/{user_id}/
  ├── profile/
  │   ├── email
  │   ├── name
  │   └── preferences
  └── cvs/{cv_id}/
      ├── extracted_data (JSON)
      ├── matches (Job matches)
      └── rewrites/{job_id}/ (Rewrite history)

/jobs/{job_id}/
  ├── title
  ├── company
  ├── requirements
  ├── skills
  └── location

/cv_status/{user_id}_{cv_id}/
  ├── status (processing|done|failed)
  ├── error (if failed)
  └── timestamp
```

**Security**:
- Access enforced through Firestore security rules
- Only authenticated users and authorized services can read/write data
- Field-level security using custom rules
- Audit logging tracks all data modifications

---

### 6.1.4 Firebase Authentication

**Purpose**: Handles user sign-in and identity management for end-users

**Supported Methods**:
- Email and password
- Google Sign-In
- Social identity providers (GitHub, Facebook, etc.)

**Integration**:
- Integrated with Auth0 for advanced identity federation
- Enables multi-factor authentication (MFA)
- Supports custom identity workflows
- Facilitates enterprise-grade single sign-on (SSO)
- Complies with organizational security policies

**Session Management**:
- Secure JWT tokens with expiration
- Refresh token rotation
- Real-time token revocation support

---

### 6.1.5 Identity and Access Management (IAM)

**Role-Based Access Control (RBAC)**:
- Granular permissions assigned to service accounts and users
- Principle of least privilege enforced

**Example: CV Extractor Service Account**
```
- Storage Bucket Read Access: {extract_bucket}
- Firestore Write Access: /users/{user_id}/cvs/*
- No access to Job data or User profiles
```

**Access Separation**:
- Developers: Deploy and update services
- Operators: Monitor and troubleshoot
- Services: Access only required resources
- Users: Access only their own data

**Outcome**:
- Separation of duties across development, operations, and data access roles
- Reduced attack surface and improved auditability
- Clear audit trails for compliance investigations

---

### 6.1.6 Google Cloud Security Services

#### Secret Manager
- Stores sensitive credentials (API keys, database passwords)
- Automatic rotation policies
- Never hardcoded in source code or environment variables
- Granular access control per secret

**Managed Secrets**:
- `GOOGLE_API_KEY` - GenAI API credential
- `OPENAI_API_KEY` - OpenRouter API credential
- `HF_TOKEN` - HuggingFace model tokens
- `EXTRACTION_SERVICE_URL` - Internal service URLs

#### Security Command Center
- Continuously monitors for threats and misconfigurations
- Automated compliance checking
- Vulnerability scanning for container images
- Real-time alerts for security events

#### VPC Service Controls
- Defines security perimeters around critical services
- Restricts data exfiltration:
  - Cloud Run can access only permitted Firestore databases
  - Storage access limited to specific buckets
  - Prevents lateral movement across projects

#### Audit Logging
- All administrative activities logged:
  - Service deployments
  - Permission changes
  - Resource modifications
- All data access activities logged:
  - Firestore reads/writes
  - GCS uploads/downloads
  - Secret access attempts
- Logs retained for compliance (90+ days, configurable)
- Logs ingested into Cloud Logging for real-time alerting

---

### 6.1.7 Auth0 (External Identity Layer)

**Purpose**: Acts as an enterprise-grade identity broker between Firebase Authentication and external identity providers

**Capabilities**:
- Enforces complex authorization policies
- User lifecycle management (provisioning/deprovisioning)
- Seamless integration with corporate SAML/OIDC systems
- Support for legacy directory services (Active Directory)

**Centralized Control**:
- Multi-factor authentication (MFA) enforcement
- Session timeout policies
- Risk-based access control
- Real-time user provisioning/deprovisioning
- Compliance with organization security standards

**Enterprise Features**:
- Single sign-on (SSO) across applications
- Custom claims and attributes
- Role-based access policies
- Audit trails for user access

---

## Data Flow Overview

The system operates through the following sequential workflow:

### Step 1: CV Upload & Trigger
```
User uploads CV to GCS bucket
        ↓
GCS triggers Cloud Function (Trigger)
```

### Step 2: Extraction Initiation
```
Trigger Function extracts metadata (user_id, cv_id)
        ↓
Updates Firestore: cv_status/{user_id}_{cv_id} → "processing"
        ↓
Calls CV Extractor API (Cloud Run)
```

### Step 3: Data Extraction
```
CV Extractor API retrieves CV from GCS
        ↓
Parses PDF/DOCX and chunks text
        ↓
Uses LLMs (GenAI + OpenAI) for intelligent extraction
        ↓
Stores structured JSON data in Firestore: users/{user_id}/cvs/{cv_id}/
```

### Step 4: Job Matching
```
Job Match API reads user profile from Firestore
        ↓
Generates semantic embeddings from CV data
        ↓
Matches against job database using vector similarity
        ↓
Saves scoring results to Firestore: users/{user_id}/cvs/{cv_id}/matches/
```

### Step 5: CV Rewriting
```
Rewrite API retrieves CV data and matched jobs from Firestore
        ↓
Uses LLM to tailor CV for top job matches
        ↓
Generates professional DOCX file
        ↓
Uploads finalized CV to GCS: {user_id}/rewrites/{job_id}.docx
```

### Step 6: User Access
```
User retrieves rewritten CV from GCS (signed URL)
        ↓
Or downloads directly from web interface
        ↓
Follows application workflow
```

### Authentication & Authorization Throughout
- **User Authentication**: Firebase Authentication validates user identity
- **Auth0 Integration**: For enterprise users, Auth0 provides federation and MFA
- **Service-to-Service Auth**: Services authenticate using:
  - IAM service account credentials
  - Mutual TLS certificates
  - Workload Identity (Kubernetes-like integration)

---

## Security Principles

### 1. **Zero Trust Model**
- Every request is authenticated and authorized
- No implicit trust based on network location
- Service-to-service communication encrypted and verified
- All API calls require valid credentials

### 2. **Defense in Depth**
Multiple overlapping security layers:
- **Network Layer**: VPC Service Controls, VPC peering
- **Authentication Layer**: Firebase + Auth0
- **Authorization Layer**: IAM + Firestore rules
- **Data Layer**: Encryption at rest, encryption in transit
- **Application Layer**: Input validation, rate limiting
- **Monitoring Layer**: Audit logs, security alerts

### 3. **Least Privilege**
- Service accounts have minimal necessary permissions
- Users can only access their own data
- API quotas prevent abuse

### 4. **Encryption**
- At Rest: Google-managed and customer-managed keys (CMEK)
- In Transit: TLS 1.2+ for all communication
- In Application: Sensitive fields encrypted before storage

### 5. **Audit & Compliance**
- All activities logged with timestamp and user identity
- Tamper-proof logs (append-only)
- Regular security audits and penetration testing
- Compliance with GDPR, SOC 2, and other standards

---

## Deployment Architecture

### Development Environment
```
Cloud Run (dev) → Firestore (dev) → GCS (dev)
                ↓
            Auth0 (shared)
```

### Production Environment
```
Cloud Run (prod) ┐
                 ├→ Firestore (prod, multi-region)
                 ├→ GCS (prod, multi-region)
                 ├→ Secret Manager (prod)
                 ├→ IAP (prod)
                 └→ VPC Service Controls (prod)
                     ↓
                  Auth0 (prod)
```

### High Availability
- Cloud Run: Automatic multi-zone deployment
- Firestore: Multi-region replication (automatic failover)
- GCS: Geo-redundant storage
- CDN: Cloud CDN for static asset delivery

---

## Scalability Considerations

### Horizontal Scaling
- **Cloud Run**: Auto-scales to 0-1000 instances based on load
- **Firestore**: Automatically scales read/write capacity
- **GCS**: Unlimited storage with consistent performance
- **Load Balancing**: Cloud Load Balancer distributes traffic

### Performance Optimization
- Caching layer (Cloud Memorystore) for frequent queries
- Connection pooling to Firestore
- Batch operations for bulk data processing
- CDN for file delivery

### Cost Optimization
- Pay-as-you-go pricing model
- Reserved capacity for predictable workloads
- Automated instance shutdown when idle
- Lifecycle policies for data retention

---

## Monitoring & Observability

### Key Metrics
- Request latency (p50, p95, p99)
- Error rates by service
- Firestore read/write throughput
- GCS transfer rates
- Service availability (uptime)

### Alerting
- High error rates (>5% of requests)
- Slow responses (p99 > 10s)
- Failed API calls
- Security policy violations
- Unusual data access patterns

### Logging
- **Cloud Logging**: Centralized log aggregation
- **Cloud Trace**: Distributed tracing for request flows
- **Cloud Profiler**: Performance profiling
- **Error Reporting**: Automatic error aggregation and alerts

---

## Integration Points

### External APIs
- **Google GenAI**: For CV extraction
- **OpenAI/OpenRouter**: For CV rewriting
- **HuggingFace**: For embeddings

### Identity Federation
- **Auth0**: Enterprise SSO and user management
- **Firebase**: End-user authentication
- **Google Workspace** (Optional): User provisioning

### Webhooks & Notifications
- Firestore listeners for real-time status updates
- Pub/Sub for event-driven workflows
- Email notifications via Cloud Tasks

---

## Disaster Recovery & Business Continuity

### Backup Strategy
- Firestore: Automatic daily backups, 7-day retention
- GCS: Versioning enabled, cross-region replication
- Configuration: Infrastructure-as-code (Terraform)

### Recovery Procedures
- RTO (Recovery Time Objective): < 1 hour
- RPO (Recovery Point Objective): < 15 minutes
- Automated failover for critical services
- Manual runbooks for complex scenarios

### Testing
- Monthly disaster recovery drills
- Automated backup restoration tests
- Load testing before major releases

---

## Modules Overview

This system consists of four interconnected microservices:

### 1. **Trigger Function** (`trigger_function/`)
- **Role**: Orchestrator and entry point
- **Trigger**: GCS file upload events
- **Function**: Initiates extraction pipeline, tracks status

### 2. **CV Extractor** (`extractor/`)
- **Role**: Data parser and structurer
- **Input**: PDF/DOCX files from GCS
- **Output**: Structured JSON in Firestore

### 3. **Job Matcher** (`job_matcher/`)
- **Role**: Semantic search and ranking engine
- **Input**: Extracted CV data
- **Output**: Ranked job matches with scores

### 4. **CV Rewriter** (`rewriter/`)
- **Role**: Content generation and formatting
- **Input**: CV data + matched jobs
- **Output**: Tailored DOCX files in GCS

Each module has its own detailed README documenting setup, APIs, and deployment.

---

## Getting Started

### Prerequisites
- Google Cloud Project with billing enabled
- Service account with appropriate roles
- Git and Docker installed

### Quick Deploy
```bash
# 1. Clone repository
git clone <repo-url>
cd cloud_prep

# 2. Set environment variables
export GCP_PROJECT_ID="your-project-id"
export GCS_BUCKET="your-bucket"
export GOOGLE_API_KEY="your-api-key"
export OPENAI_API_KEY="your-openai-key"
export HF_TOKEN="your-huggingface-token"
export EXTRACTION_SERVICE_URL="https://extractor-url"

# 3. Deploy each service
./deploy.sh

# 4. Configure Auth0 (if using enterprise SSO)
./configure-auth0.sh
```

See individual module READMEs for detailed setup instructions.

---

## Troubleshooting

### Service Connectivity Issues
- Check Cloud Run service URLs in environment variables
- Verify IAM permissions for service accounts
- Review Cloud Logging for detailed error messages

### Data Access Issues
- Verify Firestore security rules
- Check IAM roles for authenticated user
- Review audit logs for access patterns

### Performance Issues
- Check Cloud Monitoring for throughput metrics
- Monitor Firestore read/write quota usage
- Review Cloud Trace for bottleneck identification

---