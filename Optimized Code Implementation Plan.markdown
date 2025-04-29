# Optimized Code Implementation Plan

This plan details the development of the Lambda functions, Whisper integration, and frontend UIs for the AI-powered meeting analysis system, using OpenAI Whisper for speech-to-text and incorporating optimizations for latency, cost, accuracy, and scalability.

## Backend (Lambda Functions, Whisper Hosting, Step Functions - Python)

### Project Structure
- **Folders:**
  - `src/audio_upload`: Handles audio uploads to S3.
  - `src/start_transcription`: Triggers Whisper transcription and diarization.
  - `src/transcript_consolidation`: Combines transcription and diarization results.
  - `src/agent_1`: Analyzes meeting transcript and generates PDF.
  - `src/agent_2`: Analyzes client data and generates clarifying questions.
  - `src/agent_3`: Analyzes salesperson data and generates clarifying questions.
  - `src/get_analysis`: Retrieves analysis results for UIs.
  - `src/common`: Shared utilities and dependencies.
- **Shared Layer:** Lambda Layer for common dependencies (`boto3`, `openai`, `requests`, `reportlab`, `pyannote.audio`).
- **Requirements:** Each function includes a `requirements.txt` or uses the shared layer.

### Dependencies
- `boto3`: AWS SDK for S3, DynamoDB, SageMaker, Step Functions, CloudFront.
- `openai`: For OpenAI Assistant API.
- `requests`: For API calls.
- `reportlab`: For PDF generation (Agent 1).
- `pyannote.audio`: For speaker diarization.
- `torch`, `numpy`: For Whisper inference.
- `python-multipart`: For multipart/form-data uploads.
- **Packaging:** Use Docker for Lambda (due to Whisper dependencies) and SageMaker/ECS deployment.

### Whisper Hosting
- **Deployment:** Host Whisper (`whisper-medium`) on AWS SageMaker with GPU support (e.g., `ml.g4dn.xlarge` spot instances).
- **Diarization:** Integrate `pyannote.audio` (fine-tuned threshold) in the SageMaker endpoint.
- **API Endpoint:** Expose Whisper and diarization via SageMaker endpoint, callable from Lambda.
- **Scaling:** Auto-scale SageMaker instances (1–10) based on job volume.
- **Cost Optimization:** Use spot instances; monitor costs via AWS Budgets.

### Core Logic Implementation

#### common_utils.py
Shared utility functions:
- **AWS Clients:** Initialize S3, DynamoDB, SageMaker, Step Functions, Secrets Manager, CloudFront.
- **Secrets Management:** Retrieve OpenAI API key, SageMaker endpoint, Cognito credentials.
- **S3 Interactions:** Upload/download files (multipart for large files), generate pre-signed URLs.
- **DynamoDB Interactions:** Read/write meeting data, analysis results, access codes, cached transcripts.
- **OpenAI Assistant API:** Create threads, add messages, run assistants, poll status, retrieve results with chunking and retry logic.
- **Whisper Transcription:** Call SageMaker with S3 URLs, process audio in 10-minute chunks.
- **Diarization:** Call `pyannote.audio` with tuned threshold; flag low-confidence segments.
- **Transcript Formatting:** Merge segments, compress output (gzip).
- **Caching:** Hash audio/transcript for caching in DynamoDB.

#### AudioUploadHandler
- **Trigger:** API Gateway POST `/meetings/{meetingId}/audio` (Cognito-authenticated).
- **Logic:**
  - Receive multipart/form-data audio blob (Opus codec, 16 kHz mono).
  - Use S3 multipart upload for large blobs.
  - Store in S3 (`uploads/{meetingId}/{timestamp}.opus`).
  - Update DynamoDB with S3 key and status.
- **Optimizations:**
  - Batch uploads every 30 seconds to reduce API calls.
  - Downsample to 16 kHz mono during upload.
  - Asynchronous uploads to avoid UI blocking.
- **Error Handling:** Retry S3 uploads; send failures to DLQ.

#### StartTranscriptionHandler
- **Trigger:** Step Function state or S3 event (`uploads/` prefix).
- **Logic:**
  - Fetch audio files from S3.
  - Split into 10-minute chunks for parallel Whisper processing.
  - Call SageMaker endpoint (`whisper-medium`) with S3 URLs.
  - Call `pyannote.audio` for diarization (tuned threshold).
  - Store transcription/diarization JSON in S3 (`transcripts/{meetingId}/`).
  - Cache results in DynamoDB (keyed by audio hash).
  - Update DynamoDB with status.
- **Optimizations:**
  - Parallel chunk processing to reduce latency.
  - Use `whisper-medium` for speed.
  - Cache transcriptions to avoid reprocessing.
- **Error Handling:** Retry SageMaker calls; send failures to DLQ.

#### TranscriptConsolidationHandler
- **Trigger:** Step Function state or S3 event (`transcripts/` prefix).
- **Logic:**
  - Fetch transcription/diarization JSON from S3.
  - Merge segments in parallel (Lambda concurrency).
  - Align text with speaker labels; flag low-confidence segments.
  - Compress formatted transcript (gzip) and store in S3 (`transcripts/{meetingId}/formatted.txt.gz`) and DynamoDB.
- **Optimizations:**
  - Parallel merging for speed.
  - Compression to reduce storage costs.
  - Error flagging for diarization issues.
- **Error Handling:** Validate JSON; retry S3/DynamoDB writes.

#### Agent[1/2/3]Handler
- **Trigger:** Step Function state or API Gateway POST `/meetings/{meetingId}/analyze`.
- **Logic:**
  - Fetch formatted transcript from S3/DynamoDB.
  - Chunk transcript (>100k tokens) into 10k-token segments.
  - Call OpenAI Assistant API with optimized JSON prompts:
    - Agent 1: Summarize meeting, identify misunderstandings, decisions, steps; generate lightweight PDF.
    - Agent 2: Analyze client data, mock research, generate clarifying questions.
    - Agent 3: Analyze salesperson data, mock research, generate clarifying questions.
  - Cache results in DynamoDB (keyed by transcript hash).
  - Save analysis to DynamoDB (`analysis/{meetingId}/{agentId}`).
  - (Agent 1): Upload PDF to S3 (`analysis_pdfs/{meetingId}.pdf`).
- **Optimizations:**
  - Chunking to handle token limits.
  - Optimized prompts to reduce tokens.
  - Caching to skip redundant analysis.
  - Lightweight PDF generation.
- **Error Handling:** Retry OpenAI API with backoff; handle token errors; send failures to DLQ.

#### GetAnalysisHandler
- **Trigger:** API Gateway GET `/meetings/{meetingId}/analysis?agent=agent[1/2/3]`.
- **Logic:**
  - Query DynamoDB for analysis by `meetingId` and `agentId`.
  - Return analysis text or PDF URL (cached via CloudFront).
  - Paginate large results.
- **Optimizations:**
  - CloudFront caching for PDFs and analysis.
  - Pagination for large outputs.
- **Error Handling:** Handle missing data; return HTTP 403 for unauthorized requests.

### Workflow Orchestration (Step Functions)
- **State Machine:**
  - **States:**
    1. `UploadAudio`: Call `AudioUploadHandler`.
    2. `TranscribeAudio`: Call `StartTranscriptionHandler` (parallel chunks).
    3. `ConsolidateTranscript`: Call `TranscriptConsolidationHandler`.
    4. `AnalyzeMeeting`: Parallel tasks for `Agent1Handler`, `Agent2Handler`, `Agent3Handler`.
    5. `StoreResults`: Save outputs to S3/DynamoDB.
    6. `NotifyUsers`: Send WebSocket notification via API Gateway.
  - **Optimizations:**
    - Parallel states for transcription and analysis.
    - WebSocket for real-time notifications.
  - **Error Handling:** Retry failed states; route errors to DLQ.

### Configuration
- **Environment Variables:** S3 buckets, DynamoDB table, SageMaker endpoint, OpenAI Assistant IDs, Cognito IDs, CloudFront distribution.
- **IAM Roles:** Least-privilege access to S3, DynamoDB, SageMaker, Step Functions, Secrets Manager, CloudFront.

### Packaging
- **Lambda Functions:** Zip archives with shared Lambda Layer.
- **Whisper Deployment:** Docker container for SageMaker (Whisper + `pyannote.audio`).
- **Deployment Tool:** AWS CDK/SAM.

### Security
- **Authentication:** Cognito for login and access code validation; JWT authorizers for API Gateway.
- **Data Protection:** S3/DynamoDB encryption; pre-signed URLs.
- **Access Control:** Restrict S3 access to Lambda roles.

### Monitoring and Logging
- **CloudWatch Logs:** Enable for Lambda, API Gateway, Step Functions.
- **CloudWatch Alarms:** Monitor Lambda errors, SageMaker latency, DynamoDB throttling.
- **X-Ray:** Trace end-to-end requests.
- **Cost Monitoring:** AWS Budgets for Whisper hosting, OpenAI API, S3.

### Data Retention and Compliance
- **S3 Lifecycle Policies:** Archive to Glacier (30 days); delete (90 days).
- **DynamoDB TTL:** Expire records after 90 days.
- **Compliance:** Anonymize sensitive data; obtain user consent.

## Frontend (Recorder/Client/Salesperson UIs - HTML/CSS/JavaScript)

### Structure
- **Single-Page App:** `index.html` with routing for Recorder, Client, Salesperson.
- **Styling:** Tailwind CSS via CDN.
- **Dependencies:** None (vanilla JavaScript).

### Recorder UI
- **Features:**
  - Cognito login.
  - Buttons: Start, Pause, Resume, Finish.
  - Display: Meeting ID, access codes, recording duration, upload progress, status.
  - PDF download link (Agent 1).
- **JavaScript:**
  - Use `navigator.mediaDevices.getUserMedia` for microphone.
  - Record as Opus (16 kHz mono) via `MediaRecorder`.
  - Batch uploads every 30 seconds (multipart/form-data).
  - On Start: Generate `meetingId`, access codes; store in DynamoDB.
  - On Finish: Trigger Step Function; receive WebSocket updates.
  - Error Handling: Display messages for microphone errors, API failures.
- **Optimizations:**
  - Opus codec for smaller files.
  - Local buffering for batched uploads.
  - Progress feedback for UX.

### Client UI
- **Features:**
  - Input: Meeting ID, access code.
  - Display: Agent 2 analysis (paginated).
  - Button: Refresh.
- **JavaScript:**
  - Validate access code via API.
  - Fetch analysis via `GET /meetings/{meetingId}/analysis?agent=agent2`.
  - WebSocket for real-time updates.
  - Paginate large outputs.
- **Optimizations:** WebSocket and pagination.

### Salesperson UI
- **Features:** Same as Client UI (Agent 3 analysis).
- **JavaScript:** Same as Client UI (`agent=agent3`).
- **Optimizations:** Same as Client UI.

### Cross-Browser Compatibility
- Test `MediaRecorder` (Opus) on Chrome, Firefox, Safari, Edge, mobile browsers.
- Provide fallback instructions.

## Testing Strategy
- **Unit Tests:** `pytest` for Lambda handlers (mock with `moto`).
- **Integration Tests:** Test API endpoints with sample audio.
- **End-to-End Tests:** Simulate multi-speaker meetings; verify outputs.
- **Load Testing:** Test 100 concurrent meetings for scalability.

## User Onboarding and Documentation
- **In-App Guidance:** Tooltips for microphone setup, access codes.
- **User Guide:** Webpage/PDF with steps and troubleshooting.
- **Troubleshooting:** Solutions for microphone issues, invalid codes.

## Deployment
- **Infrastructure:** AWS CDK/SAM for Lambda, S3, DynamoDB, SageMaker, Step Functions, API Gateway, Cognito, CloudFront.
- **CI/CD:** CodePipeline for automated deployments.
- **Environment:** Staging before production.

## Cost Optimization
- **Whisper Hosting:** SageMaker spot instances; savings plans.
- **OpenAI API:** Cache results; optimized prompts.
- **S3/DynamoDB:** Compression, lifecycle policies, TTL.
- **Monitoring:** AWS Budget alerts.