# Phased Implementation Commands for Cursor

This document breaks the optimized code implementation plan for the AI-powered meeting analysis system into phases and commands for Cursor. Each phase includes objectives, commands, testing instructions, and estimated times.

## Phase 1: Project Setup and Dependencies
**Objective**: Set up the project structure, install dependencies, and configure AWS credentials.
**Estimated Time**: 1–2 hours

### Commands
1. **Create Project Directory and Initialize Git**
   ```bash
   mkdir meeting-analysis-system
   cd meeting-analysis-system
   git init
   ```
   - Cursor: Run in terminal (`Ctrl+T` or `Cmd+T`).

2. **Create Project Structure**
   ```bash
   mkdir -p src/audio_upload src/start_transcription src/transcript_consolidation src/agent_1 src/agent_2 src/agent_3 src/get_analysis src/common frontend
   touch src/common/common_utils.py
   touch src/audio_upload/lambda_function.py
   touch src/start_transcription/lambda_function.py
   touch src/transcript_consolidation/lambda_function.py
   touch src/agent_1/lambda_function.py
   touch src/agent_2/lambda_function.py
   touch src/agent_3/lambda_function.py
   touch src/get_analysis/lambda_function.py
   touch frontend/index.html frontend/styles.css frontend/app.js
   touch requirements.txt
   touch Dockerfile
   touch cdk.json
   ```
   - Cursor: Use `Ctrl+Shift+P` → “Create New File” or run commands in terminal.

3. **Install Python Dependencies**
   ```bash
   echo "boto3==1.28.0
   openai==1.10.0
   requests==2.31.0
   reportlab==4.0.4
   pyannote.audio==3.1.0
   torch==2.0.1
   numpy==1.24.3
   python-multipart==0.0.6
   pytest==7.4.0
   moto==4.2.0" > requirements.txt
   pip install -r requirements.txt
   ```
   - Cursor: Run in terminal or use `requirements.txt` auto-install.

4. **Configure AWS CLI**
   ```bash
   aws configure
   ```
   - Enter AWS Access Key ID, Secret Access Key, region (e.g., `us-east-1`), and output format (`json`).
   - Cursor: Run in terminal.

5. **Initialize AWS CDK**
   ```bash
   npm install -g aws-cdk
   cdk init app --language python
   ```
   - Cursor: Run in terminal; ensure Node.js is installed.

### Testing Instructions
- **Directory Structure**: Run `ls -R` to verify `src/`, `frontend/`, `requirements.txt`, `Dockerfile`, and `cdk.json` exist.
- **Dependencies**: Run `pip list` to confirm `boto3`, `openai`, `reportlab`, etc., are installed.
- **AWS CLI**: Run `aws sts get-caller-identity` to verify credentials are configured.
- **CDK**: Run `cdk ls` to confirm the CDK app is initialized (should list a stack).
- **Fix Issues**: If dependencies fail, check Python version (`python --version` should be 3.9+); if AWS CLI fails, re-run `aws configure`.

---

## Phase 2: Common Utilities (common_utils.py)
**Objective**: Implement shared utility functions for AWS interactions, OpenAI API, Whisper, and diarization.
**Estimated Time**: 2–3 hours

### Commands
1. **Create common_utils.py**
   ```python
   # src/common/common_utils.py
   import boto3
   import json
   import hashlib
   import gzip
   from openai import OpenAI
   from botocore.config import Config
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def init_aws_clients():
       retry_config = Config(retries={'max_attempts': 5, 'mode': 'adaptive'})
       return {
           's3': boto3.client('s3', config=retry_config),
           'dynamodb': boto3.resource('dynamodb', config=retry_config),
           'sagemaker': boto3.client('sagemaker-runtime', config=retry_config),
           'stepfunctions': boto3.client('stepfunctions', config=retry_config),
           'secretsmanager': boto3.client('secretsmanager', config=retry_config)
       }

   def get_secret(secret_arn):
       client = init_aws_clients()['secretsmanager']
       response = client.get_secret_value(SecretId=secret_arn)
       return json.loads(response['SecretString'])

   def upload_to_s3(s3_client, bucket, key, data, compress=False):
       if compress:
           data = gzip.compress(data.encode('utf-8'))
           key = f"{key}.gz"
       s3_client.put_object(Bucket=bucket, Key=key, Body=data)
       return key

   def download_from_s3(s3_client, bucket, key):
       response = s3_client.get_object(Bucket=bucket, Key=key)
       data = response['Body'].read()
       if key.endswith('.gz'):
           data = gzip.decompress(data).decode('utf-8')
       return data

   def get_openai_client(api_key):
       return OpenAI(api_key=api_key)

   def run_openai_assistant(client, assistant_id, thread_id, transcript_chunks):
       responses = []
       for chunk in transcript_chunks:
           thread = client.beta.threads.create() if not thread_id else client.beta.threads.retrieve(thread_id)
           client.beta.threads.messages.create(thread_id=thread.id, role="user", content=chunk)
           run = client.beta.threads.runs.create(thread_id=thread.id, assistant_id=assistant_id)
           while run.status != 'completed':
               run = client.beta.threads.runs.retrieve(thread_id=thread.id, run_id=run.id)
           messages = client.beta.threads.messages.list(thread_id=thread.id)
           responses.append(messages.data[0].content[0].text.value)
       return responses

   def chunk_transcript(transcript, max_tokens=10000):
       words = transcript.split()
       chunks = []
       current_chunk = []
       current_tokens = 0
       for word in words:
           current_chunk.append(word)
           current_tokens += 1  # Approximate token count
           if current_tokens >= max_tokens:
               chunks.append(" ".join(current_chunk))
               current_chunk = []
               current_tokens = 0
       if current_chunk:
           chunks.append(" ".join(current_chunk))
       return chunks

   def hash_content(content):
       return hashlib.sha256(content.encode()).hexdigest()

   def cache_result(dynamodb, table_name, key, content):
       table = dynamodb.Table(table_name)
       table.put_item(Item={'key': key, 'content': content})

   def get_cached_result(dynamodb, table_name, key):
       table = dynamodb.Table(table_name)
       response = table.get_item(Key={'key': key})
       return response.get('Item', {}).get('content')
   ```
   - Cursor: Open `src/common/common_utils.py`, paste code, and save (`Ctrl+S`).

2. **Create Unit Tests**
   ```python
   # src/common/test_common_utils.py
   import pytest
   from unittest.mock import MagicMock
   from common_utils import init_aws_clients, get_secret, chunk_transcript

   def test_init_aws_clients():
       clients = init_aws_clients()
       assert 's3' in clients
       assert 'dynamodb' in clients

   def test_chunk_transcript():
       transcript = "word " * 20000
       chunks = chunk_transcript(transcript, max_tokens=10000)
       assert len(chunks) == 2
       assert len(chunks[0].split()) <= 10000
   ```
   - Cursor: Create `src/common/test_common_utils.py`, paste code, save.

### Testing Instructions
- **Code Syntax**: Run `python -m pyflakes src/common/common_utils.py` to check for syntax errors.
- **Unit Tests**: Run `pytest src/common/test_common_utils.py` to verify `init_aws_clients` and `chunk_transcript` work.
- **Manual Test**: Create a test script to call `init_aws_clients()` and verify AWS clients are initialized (e.g., `print(clients['s3'])`).
- **Fix Issues**: If tests fail, check `boto3` installation or AWS credentials; if `pyannote.audio` imports fail, ensure GPU dependencies are installed.

---

## Phase 3: Audio Upload Handler
**Objective**: Implement the Lambda function to upload audio blobs to S3 with optimizations (Opus codec, multipart uploads).
**Estimated Time**: 2–3 hours

### Commands
1. **Create AudioUploadHandler**
   ```python
   # src/audio_upload/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients, upload_to_s3
   from python_multipart import parse_multipart
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           s3_client = clients['s3']
           dynamodb = clients['dynamodb']
           bucket = os.getenv('UPLOADS_BUCKET')
           meeting_id = event['pathParameters']['meetingId']

           # Parse multipart/form-data
           content_type = event['headers']['content-type']
           body = base64.b64decode(event['body'])
           parsed = parse_multipart(body, content_type)
           audio_data = parsed['audio'][0]['data']

           # Downsample to 16 kHz mono (assumed pre-processed by client)
           key = f"uploads/{meeting_id}/{int(time.time())}.opus"
           upload_to_s3(s3_client, bucket, key, audio_data)

           # Update DynamoDB
           table = dynamodb.Table(os.getenv('DYNAMODB_TABLE'))
           table.update_item(
               Key={'meetingId': meeting_id},
               UpdateExpression='SET uploadStatus = :status, s3Key = :key',
               ExpressionAttributeValues={':status': 'Uploaded', ':key': key}
           )

           return {
               'statusCode': 200,
               'body': json.dumps({'message': 'Audio uploaded', 's3Key': key})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/audio_upload/lambda_function.py`, paste code, save.

2. **Create Unit Tests**
   ```python
   # src/audio_upload/test_lambda_function.py
   import pytest
   from unittest.mock import MagicMock
   from lambda_function import lambda_handler

   def test_lambda_handler():
       event = {
           'pathParameters': {'meetingId': 'test123'},
           'headers': {'content-type': 'multipart/form-data; boundary=----test'},
           'body': base64.b64encode(b'------test\nContent-Disposition: form-data; name="audio"\n\ntestdata\n------test--')
       }
       context = {}
       response = lambda_handler(event, context)
       assert response['statusCode'] == 200
       assert 's3Key' in json.loads(response['body'])
   ```
   - Cursor: Create `src/audio_upload/test_lambda_function.py`, paste code, save.

### Testing Instructions
- **Syntax**: Run `python -m pyflakes src/audio_upload/lambda_function.py`.
- **Unit Tests**: Run `pytest src/audio_upload/test_lambda_function.py` to verify handler processes multipart data.
- **Manual Test**: Deploy the Lambda function locally using `sam local invoke` with a mock event; check S3 and DynamoDB for uploaded file and status.
- **Fix Issues**: If multipart parsing fails, ensure `python-multipart` is installed; if S3 fails, verify bucket permissions.

---

## Phase 4: Transcription and Diarization Handler
**Objective**: Implement the Lambda function to transcribe audio with Whisper and diarize with `pyannote.audio`.
**Estimated Time**: 3–4 hours

### Commands
1. **Create StartTranscriptionHandler**
   ```python
   # src/start_transcription/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients, upload_to_s3, hash_content, cache_result
   import time
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           s3_client = clients['s3']
           sagemaker_client = clients['sagemaker-runtime']
           dynamodb = clients['dynamodb']
           bucket = os.getenv('UPLOADS_BUCKET')
           transcripts_bucket = os.getenv('TRANSCRIPTS_BUCKET')
           table_name = os.getenv('DYNAMODB_TABLE')
           endpoint_name = os.getenv('SAGEMAKER_ENDPOINT')

           # Get audio files
           meeting_id = event['meetingId']
           response = s3_client.list_objects_v2(Bucket=bucket, Prefix=f"uploads/{meeting_id}/")
           audio_files = [obj['Key'] for obj in response.get('Contents', [])]

           # Process in 10-minute chunks
           transcriptions = []
           diarizations = []
           for audio_key in audio_files:
               audio_hash = hash_content(download_from_s3(s3_client, bucket, audio_key))
               cached = get_cached_result(dynamodb, table_name, audio_hash)
               if cached:
                   transcriptions.append(cached['transcription'])
                   diarizations.append(cached['diarization'])
                   continue

               # Call SageMaker for Whisper (assume 10-minute chunks)
               s3_url = s3_client.generate_presigned_url('get_object', Params={'Bucket': bucket, 'Key': audio_key})
               payload = json.dumps({'audio_url': s3_url, 'model': 'whisper-medium'})
               response = sagemaker_client.invoke_endpoint(
                   EndpointName=endpoint_name,
                   ContentType='application/json',
                   Body=payload
               )
               result = json.loads(response['Body'].read())
               transcription = result['transcription']
               diarization = result['diarization']  # pyannote.audio output

               # Cache and store
               cache_result(dynamodb, table_name, audio_hash, {'transcription': transcription, 'diarization': diarization})
               transcription_key = f"transcripts/{meeting_id}/{int(time.time())}_transcription.json"
               diarization_key = f"transcripts/{meeting_id}/{int(time.time())}_diarization.json"
               upload_to_s3(s3_client, transcripts_bucket, transcription_key, json.dumps(transcription))
               upload_to_s3(s3_client, transcripts_bucket, diarization_key, json.dumps(diarization))

               transcriptions.append(transcription)
               diarizations.append(diarization)

           # Update DynamoDB
           table = dynamodb.Table(table_name)
           table.update_item(
               Key={'meetingId': meeting_id},
               UpdateExpression='SET transcriptionStatus = :status',
               ExpressionAttributeValues={':status': 'Completed'}
           )

           return {
               'statusCode': 200,
               'body': json.dumps({'meetingId': meeting_id, 'transcriptions': transcriptions})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/start_transcription/lambda_function.py`, paste code, save.

2. **Create Unit Tests**
   ```python
   # src/start_transcription/test_lambda_function.py
   import pytest
   from unittest.mock import MagicMock
   from lambda_function import lambda_handler

   def test_lambda_handler():
       event = {'meetingId': 'test123'}
       context = {}
       response = lambda_handler(event, context)
       assert response['statusCode'] == 200
       assert 'transcriptions' in json.loads(response['body'])
   ```
   - Cursor: Create `src/start_transcription/test_lambda_function.py`, paste code, save.

### Testing Instructions
- **Syntax**: Run `python -m pyflakes src/start_transcription/lambda_function.py`.
- **Unit Tests**: Run `pytest src/start_transcription/test_lambda_function.py` to verify handler logic.
- **Manual Test**: Deploy locally with `sam local invoke`; simulate S3 audio files and mock SageMaker responses; check S3 for JSON outputs and DynamoDB for status.
- **Fix Issues**: If SageMaker calls fail, verify endpoint configuration; if caching fails, check DynamoDB permissions.

---

## Phase 5: Transcript Consolidation Handler
**Objective**: Implement the Lambda function to merge transcription and diarization results into a formatted transcript.
**Estimated Time**: 2–3 hours

### Commands
1. **Create TranscriptConsolidationHandler**
   ```python
   # src/transcript_consolidation/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients, download_from_s3, upload_to_s3
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           s3_client = clients['s3']
           dynamodb = clients['dynamodb']
           bucket = os.getenv('TRANSCRIPTS_BUCKET')
           table_name = os.getenv('DYNAMODB_TABLE')
           meeting_id = event['meetingId']

           # Fetch transcription and diarization files
           response = s3_client.list_objects_v2(Bucket=bucket, Prefix=f"transcripts/{meeting_id}/")
           files = [obj['Key'] for obj in response.get('Contents', [])]
           transcriptions = [json.loads(download_from_s3(s3_client, bucket, key)) for key in files if key.endswith('_transcription.json')]
           diarizations = [json.loads(download_from_s3(s3_client, bucket, key)) for key in files if key.endswith('_diarization.json')]

           # Merge chronologically
           formatted_transcript = []
           for t, d in zip(transcriptions, diarizations):
               for segment in t['segments']:
                   speaker = next((s['speaker'] for s in d['segments'] if s['start'] <= segment['start'] <= s['end']), 'Unknown')
                   confidence = next((s['confidence'] for s in d['segments'] if s['start'] <= segment['start'] <= s['end']), 1.0)
                   prefix = f"[Low Confidence] " if confidence < 0.7 else ""
                   formatted_transcript.append(f"{prefix}{speaker}: {segment['text']}")

           # Store formatted transcript
           transcript_text = "\n".join(formatted_transcript)
           key = f"transcripts/{meeting_id}/formatted.txt.gz"
           upload_to_s3(s3_client, bucket, key, transcript_text, compress=True)

           # Update DynamoDB
           table = dynamodb.Table(table_name)
           table.update_item(
               Key={'meetingId': meeting_id},
               UpdateExpression='SET transcript = :transcript, consolidationStatus = :status',
               ExpressionAttributeValues={':transcript': transcript_text, ':status': 'Completed'}
           )

           return {
               'statusCode': 200,
               'body': json.dumps({'meetingId': meeting_id, 'transcript': transcript_text})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/transcript_consolidation/lambda_function.py`, paste code, save.

2. **Create Unit Tests**
   ```python
   # src/transcript_consolidation/test_lambda_function.py
   import pytest
   from unittest.mock import MagicMock
   from lambda_function import lambda_handler

   def test_lambda_handler():
       event = {'meetingId': 'test123'}
       context = {}
       response = lambda_handler(event, context)
       assert response['statusCode'] == 200
       assert 'transcript' in json.loads(response['body'])
   ```
   - Cursor: Create `src/transcript_consolidation/test_lambda_function.py`, paste code, save.

### Testing Instructions
- **Syntax**: Run `python -m pyflakes src/transcript_consolidation/lambda_function.py`.
- **Unit Tests**: Run `pytest src/transcript_consolidation/test_lambda_function.py`.
- **Manual Test**: Deploy locally; provide mock transcription/diarization JSON in S3; check S3 for `formatted.txt.gz` and DynamoDB for transcript.
- **Fix Issues**: If merging fails, verify JSON structure; if compression fails, ensure `gzip` works.

---

## Phase 6: Agent Handlers (Agent 1, 2, 3)
**Objective**: Implement Lambda functions for agent analysis with optimized OpenAI API calls.
**Estimated Time**: 3–4 hours

### Commands
1. **Create Agent1Handler (Meeting Analysis and PDF)**
   ```python
   # src/agent_1/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients, download_from_s3, upload_to_s3, get_openai_client, run_openai_assistant, chunk_transcript
   from reportlab.lib.pagesizes import letter
   from reportlab.pdfgen import canvas
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           s3_client = clients['s3']
           dynamodb = clients['dynamodb']
           bucket = os.getenv('TRANSCRIPTS_BUCKET')
           pdf_bucket = os.getenv('PDF_BUCKET')
           table_name = os.getenv('DYNAMODB_TABLE')
           secret_arn = os.getenv('OPENAI_SECRET_ARN')
           assistant_id = os.getenv('AGENT1_ASSISTANT_ID')

           meeting_id = event['meetingId']
           transcript_key = f"transcripts/{meeting_id}/formatted.txt.gz"
           transcript = download_from_s3(s3_client, bucket, transcript_key)

           # Chunk and analyze
           api_key = get_secret(secret_arn)['api_key']
           client = get_openai_client(api_key)
           chunks = chunk_transcript(transcript)
           prompt = {"task": "Summarize meeting, identify misunderstandings, decisions, steps"}
           responses = run_openai_assistant(client, assistant_id, None, [f"{json.dumps(prompt)}\n{chunk}" for chunk in chunks])

           # Combine responses
           analysis = "\n".join(responses)
           table = dynamodb.Table(table_name)
           table.update_item(
               Key={'meetingId': meeting_id, 'agentId': 'agent1'},
               UpdateExpression='SET analysis = :analysis',
               ExpressionAttributeValues={':analysis': analysis}
           )

           # Generate PDF
           pdf_key = f"analysis_pdfs/{meeting_id}.pdf"
           c = canvas.Canvas(f"/tmp/{meeting_id}.pdf", pagesize=letter)
           c.drawString(100, 750, "Meeting Analysis")
           y = 700
           for line in analysis.split("\n"):
               if y < 50:
                   c.showPage()
                   y = 750
               c.drawString(100, y, line[:80])
               y -= 20
           c.save()
           with open(f"/tmp/{meeting_id}.pdf", 'rb') as f:
               upload_to_s3(s3_client, pdf_bucket, pdf_key, f.read())

           return {
               'statusCode': 200,
               'body': json.dumps({'meetingId': meeting_id, 'analysis': analysis, 'pdfKey': pdf_key})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/agent_1/lambda_function.py`, paste code, save.

2. **Create Agent2Handler (Client Analysis)**
   ```python
   # src/agent_2/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients, download_from_s3, get_openai_client, run_openai_assistant, chunk_transcript
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           s3_client = clients['s3']
           dynamodb = clients['dynamodb']
           bucket = os.getenv('TRANSCRIPTS_BUCKET')
           table_name = os.getenv('DYNAMODB_TABLE')
           secret_arn = os.getenv('OPENAI_SECRET_ARN')
           assistant_id = os.getenv('AGENT2_ASSISTANT_ID')

           meeting_id = event['meetingId']
           transcript_key = f"transcripts/{meeting_id}/formatted.txt.gz"
           transcript = download_from_s3(s3_client, bucket, transcript_key)

           # Chunk and analyze
           api_key = get_secret(secret_arn)['api_key']
           client = get_openai_client(api_key)
           chunks = chunk_transcript(transcript)
           prompt = {"task": "Analyze client data, generate clarifying questions"}
           responses = run_openai_assistant(client, assistant_id, None, [f"{json.dumps(prompt)}\n{chunk}" for chunk in chunks])

           # Combine responses
           analysis = "\n".join(responses)
           table = dynamodb.Table(table_name)
           table.update_item(
               Key={'meetingId': meeting_id, 'agentId': 'agent2'},
               UpdateExpression='SET analysis = :analysis',
               ExpressionAttributeValues={':analysis': analysis}
           )

           return {
               'statusCode': 200,
               'body': json.dumps({'meetingId': meeting_id, 'analysis': analysis})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/agent_2/lambda_function.py`, paste code, save.

3. **Create Agent3Handler (Salesperson Analysis)**
   ```python
   # src/agent_3/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients, download_from_s3, get_openai_client, run_openai_assistant, chunk_transcript
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           s3_client = clients['s3']
           dynamodb = clients['dynamodb']
           bucket = os.getenv('TRANSCRIPTS_BUCKET')
           table_name = os.getenv('DYNAMODB_TABLE')
           secret_arn = os.getenv('OPENAI_SECRET_ARN')
           assistant_id = os.getenv('AGENT3_ASSISTANT_ID')

           meeting_id = event['meetingId']
           transcript_key = f"transcripts/{meeting_id}/formatted.txt.gz"
           transcript = download_from_s3(s3_client, bucket, transcript_key)

           # Chunk and analyze
           api_key = get_secret(secret_arn)['api_key']
           client = get_openai_client(api_key)
           chunks = chunk_transcript(transcript)
           prompt = {"task": "Analyze salesperson data, generate clarifying questions"}
           responses = run_openai_assistant(client, assistant_id, None, [f"{json.dumps(prompt)}\n{chunk}" for chunk in chunks])

           # Combine responses
           analysis = "\n".join(responses)
           table = dynamodb.Table(table_name)
           table.update_item(
               Key={'meetingId': meeting_id, 'agentId': 'agent3'},
               UpdateExpression='SET analysis = :analysis',
               ExpressionAttributeValues={':analysis': analysis}
           )

           return {
               'statusCode': 200,
               'body': json.dumps({'meetingId': meeting_id, 'analysis': analysis})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/agent_3/lambda_function.py`, paste code, save.

4. **Create Unit Tests**
   ```python
   # src/agent_1/test_lambda_function.py
   import pytest
   from unittest.mock import MagicMock
   from lambda_function import lambda_handler

   def test_lambda_handler():
       event = {'meetingId': 'test123'}
       context = {}
       response = lambda_handler(event, context)
       assert response['statusCode'] == 200
       assert 'analysis' in json.loads(response['body'])
   ```
   - Cursor: Create `src/agent_1/test_lambda_function.py`, paste code, save. Repeat for `src/agent_2/` and `src/agent_3/`.

### Testing Instructions
- **Syntax**: Run `python -m pyflakes src/agent_1/lambda_function.py` (and for agent_2, agent_3).
- **Unit Tests**: Run `pytest src/agent_1/test_lambda_function.py` (and for agent_2, agent_3).
- **Manual Test**: Deploy locally; provide a mock transcript in S3; check DynamoDB for analysis and S3 for PDF (Agent 1).
- **Fix Issues**: If OpenAI API fails, verify API key; if PDF generation fails, check `reportlab` installation.

---

## Phase 7: Get Analysis Handler
**Objective**: Implement the Lambda function to retrieve analysis results for UIs.
**Estimated Time**: 1–2 hours

### Commands
1. **Create GetAnalysisHandler**
   ```python
   # src/get_analysis/lambda_function.py
   import json
   import boto3
   from common_utils import init_aws_clients
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           clients = init_aws_clients()
           dynamodb = clients['dynamodb']
           table_name = os.getenv('DYNAMODB_TABLE')
           meeting_id = event['pathParameters']['meetingId']
           agent_id = event['queryStringParameters']['agent']

           table = dynamodb.Table(table_name)
           response = table.get_item(Key={'meetingId': meeting_id, 'agentId': agent_id})
           analysis = response.get('Item', {}).get('analysis', '')

           if not analysis:
               return {
                   'statusCode': 404,
                   'body': json.dumps({'error': 'Analysis not found'})
               }

           return {
               'statusCode': 200,
               'body': json.dumps({'meetingId': meeting_id, 'agentId': agent_id, 'analysis': analysis})
           }
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {
               'statusCode': 500,
               'body': json.dumps({'error': str(e)})
           }
   ```
   - Cursor: Open `src/get_analysis/lambda_function.py`, paste code, save.

2. **Create Unit Tests**
   ```python
   # src/get_analysis/test_lambda_function.py
   import pytest
   from unittest.mock import MagicMock
   from lambda_function import lambda_handler

   def test_lambda_handler():
       event = {
           'pathParameters': {'meetingId': 'test123'},
           'queryStringParameters': {'agent': 'agent1'}
       }
       context = {}
       response = lambda_handler(event, context)
       assert response['statusCode'] in [200, 404]
   ```
   - Cursor: Create `src/get_analysis/test_lambda_function.py`, paste code, save.

### Testing Instructions
- **Syntax**: Run `python -m pyflakes src/get_analysis/lambda_function.py`.
- **Unit Tests**: Run `pytest src/get_analysis/test_lambda_function.py`.
- **Manual Test**: Deploy locally; insert mock analysis in DynamoDB; call API with `curl` or Postman; verify response.
- **Fix Issues**: If DynamoDB query fails, check table schema and permissions.

---

## Phase 8: Frontend Development
**Objective**: Implement the single-page app for Recorder, Client, and Salesperson UIs with optimizations.
**Estimated Time**: 4–5 hours

### Commands
1. **Create index.html**
   ```html
   <!-- frontend/index.html -->
   <!DOCTYPE html>
   <html lang="en">
   <head>
       <meta charset="UTF-8">
       <meta name="viewport" content="width=device-width, initial-scale=1.0">
       <title>Meeting Analysis System</title>
       <script src="https://cdn.tailwindcss.com"></script>
       <script src="app.js" defer></script>
       <link rel="stylesheet" href="styles.css">
   </head>
   <body class="bg-gray-100">
       <div id="app" class="container mx-auto p-4"></div>
   </body>
   </html>
   ```
   - Cursor: Open `frontend/index.html`, paste code, save.

2. **Create styles.css**
   ```css
   /* frontend/styles.css */
   .hidden { display: none; }
   .error { color: red; }
   .status { color: blue; }
   ```
   - Cursor: Open `frontend/styles.css`, paste code, save.

3. **Create app.js**
   ```javascript
   // frontend/app.js
   const API_BASE = 'https://your-api-id.execute-api.us-east-1.amazonaws.com/prod';
   const COGNITO_LOGIN = 'https://your-cognito-domain.auth.us-east-1.amazoncognito.com/login?client_id=your-client-id&response_type=token&redirect_uri=' + encodeURIComponent(window.location.origin);

   async function init() {
       const token = getToken();
       if (!token) {
           window.location.href = COGNITO_LOGIN;
           return;
       }

       const path = window.location.pathname;
       const app = document.getElementById('app');
       if (path === '/recorder') {
           renderRecorder(app, token);
       } else if (path === '/client') {
           renderClient(app, token);
       } else if (path === '/salesperson') {
           renderSalesperson(app, token);
       }
   }

   function getToken() {
       const hash = window.location.hash;
       const params = new URLSearchParams(hash.replace('#', ''));
       return params.get('id_token');
   }

   async function renderRecorder(app, token) {
       let mediaRecorder;
       let meetingId = generateUUID();
       let accessCodes = { client: generateUUID(), salesperson: generateUUID() };
       app.innerHTML = `
           <h1 class="text-2xl mb-4">Recorder</h1>
           <p>Meeting ID: <span id="meetingId">${meetingId}</span></p>
           <p>Client Code: <span id="clientCode">${accessCodes.client}</span></p>
           <p>Salesperson Code: <span id="salespersonCode">${accessCodes.salesperson}</span></p>
           <button id="start" class="bg-blue-500 text-white p-2 m-2">Start</button>
           <button id="pause" class="bg-yellow-500 text-white p-2 m-2 hidden">Pause</button>
           <button id="resume" class="bg-green-500 text-white p-2 m-2 hidden">Resume</button>
           <button id="finish" class="bg-red-500 text-white p-2 m-2 hidden">Finish</button>
           <p id="status" class="status"></p>
           <p id="error" class="error hidden"></p>
           <a id="pdfLink" class="hidden" href="#">Download PDF</a>
       `;

       document.getElementById('start').onclick = async () => {
           try {
               const stream = await navigator.mediaDevices.getUserMedia({ audio: true });
               mediaRecorder = new MediaRecorder(stream, { mimeType: 'audio/ogg; codecs=opus' });
               mediaRecorder.ondataavailable = async (e) => {
                   const formData = new FormData();
                   formData.append('audio', e.data, 'audio.opus');
                   const response = await fetch(`${API_BASE}/meetings/${meetingId}/audio`, {
                       method: 'POST',
                       headers: { 'Authorization': `Bearer ${token}` },
                       body: formData
                   });
                   if (!response.ok) throw new Error('Upload failed');
                   document.getElementById('status').textContent = 'Uploaded audio';
               };
               mediaRecorder.start(30000); // Batch every 30 seconds
               toggleButtons('recording');
           } catch (e) {
               document.getElementById('error').textContent = `Error: ${e.message}`;
               document.getElementById('error').classList.remove('hidden');
           }
       };

       document.getElementById('pause').onclick = () => {
           mediaRecorder.pause();
           toggleButtons('paused');
       };

       document.getElementById('resume').onclick = () => {
           mediaRecorder.resume();
           toggleButtons('recording');
       };

       document.getElementById('finish').onclick = async () => {
           mediaRecorder.stop();
           toggleButtons('finished');
           const response = await fetch(`${API_BASE}/meetings/${meetingId}/analyze`, {
               method: 'POST',
               headers: { 'Authorization': `Bearer ${token}` },
               body: JSON.stringify({ meetingId, accessCodes })
           });
           if (response.ok) {
               document.getElementById('status').textContent = 'Processing...';
               const ws = new WebSocket('wss://your-websocket-id.execute-api.us-east-1.amazonaws.com/prod');
               ws.onmessage = (event) => {
                   const data = JSON.parse(event.data);
                   if (data.meetingId === meetingId && data.status === 'Completed') {
                       document.getElementById('status').textContent = 'Analysis complete';
                       document.getElementById('pdfLink').href = `${API_BASE}/meetings/${meetingId}/analysis?agent=agent1`;
                       document.getElementById('pdfLink').classList.remove('hidden');
                   }
               };
           }
       };

       function toggleButtons(state) {
           document.getElementById('start').classList[state推: state === 'recording' ? 'add' : 'remove';
           document.getElementById('pause').classList[state === 'paused' ? 'remove' : 'add']('hidden');
           document.getElementById('resume').classList[state === 'paused' ? 'add' : 'remove']('hidden');
           document.getElementById('finish').classList[state === 'recording' || state === 'paused' ? 'add' : 'remove']('hidden');
       }
   }

   async function renderClient(app, token) {
       app.innerHTML = `
           <h1 class="text-2xl mb-4">Client UI</h1>
           <input id="meetingId" class="border p-2 m-2" placeholder="Meeting ID">
           <input id="accessCode" class="border p-2 m-2" placeholder="Access Code">
           <button id="submit" class="bg-blue-500 text-white p-2 m-2">Submit</button>
           <pre id="analysis" class="bg-white p-4"></pre>
           <p id="error" class="error hidden"></p>
       `;

       document.getElementById('submit').onclick = async () => {
           const meetingId = document.getElementById('meetingId').value;
           const accessCode = document.getElementById('accessCode').value;
           try {
               const response = await fetch(`${API_BASE}/meetings/${meetingId}/validate`, {
                   method: 'POST',
                   headers: { 'Authorization': `Bearer ${token}` },
                   body: JSON.stringify({ accessCode })
               });
               if (!response.ok) throw new Error('Invalid access code');
               const analysisResponse = await fetch(`${API_BASE}/meetings/${meetingId}/analysis?agent=agent2`, {
                   headers: { 'Authorization': `Bearer ${token}` }
               });
               const data = await analysisResponse.json();
               document.getElementById('analysis').textContent = data.analysis;
           } catch (e) {
               document.getElementById('error').textContent = `Error: ${e.message}`;
               document.getElementById('error').classList.remove('hidden');
           }
       };
   }

   async function renderSalesperson(app, token) {
       app.innerHTML = `
           <h1 class="text-2xl mb-4">Salesperson UI</h1>
           <input id="meetingId" class="border p-2 m-2" placeholder="Meeting ID">
           <input id="accessCode" class="border p-2 m-2" placeholder="Access Code">
           <button id="submit" class="bg-blue-500 text-white p-2 m-2">Submit</button>
           <pre id="analysis" class="bg-white p-4"></pre>
           <p id="error" class="error hidden"></p>
       `;

       document.getElementById('submit').onclick = async () => {
           const meetingId = document.getElementById('meetingId').value;
           const accessCode = document.getElementById('accessCode').value;
           try {
               const response = await fetch(`${API_BASE}/meetings/${meetingId}/validate`, {
                   method: 'POST',
                   headers: { 'Authorization': `Bearer ${token}` },
                   body: JSON.stringify({ accessCode })
               });
               if (!response.ok) throw new Error('Invalid access code');
               const analysisResponse = await fetch(`${API_BASE}/meetings/${meetingId}/analysis?agent=agent3`, {
                   headers: { 'Authorization': `Bearer ${token}` }
               });
               const data = await analysisResponse.json();
               document.getElementById('analysis').textContent = data.analysis;
           } catch (e) {
               document.getElementById('error').textContent = `Error: ${e.message}`;
               document.getElementById('error').classList.remove('hidden');
           }
       };
   }

   function generateUUID() {
       return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, c => {
           const r = Math.random() * 16 | 0;
           return (c === 'x' ? r : (r & 0x3 | 0x8)).toString(16);
       });
   }

   window.onload = init;
   ```
   - Cursor: Open `frontend/app.js`, paste code, save. Replace `API_BASE`, `COGNITO_LOGIN`, and WebSocket URL with actual values post-deployment.

### Testing Instructions
- **Syntax**: Open `index.html` in a browser; verify UI renders without errors.
- **Recorder UI**: Navigate to `/recorder`, click Start, grant microphone access, record 10 seconds, pause, resume, finish; check console for upload status.
- **Client/Salesperson UI**: Navigate to `/client` or `/salesperson`, enter mock meeting ID and access code, submit; verify analysis or error message.
- **Manual Test**: Use browser dev tools to inspect network requests; verify API calls and WebSocket connections.
- **Fix Issues**: If microphone fails, check browser permissions; if API calls fail, verify Cognito token and API Gateway URL.

---

## Phase 9: SageMaker Deployment for Whisper
**Objective**: Deploy Whisper and `pyannote.audio` on SageMaker with a Docker container.
**Estimated Time**: 4–6 hours

### Commands
1. **Create Dockerfile**
   ```dockerfile
   # Dockerfile
   FROM python:3.9-slim

   RUN apt-get update && apt-get install -y ffmpeg
   RUN pip install torch==2.0.1 numpy==1.24.3 pyannote.audio==3.1.0 openai-whisper==20231117

   COPY inference.py /opt/ml/code/inference.py

   ENV PYTHONUNBUFFERED=TRUE
   ENTRYPOINT ["python", "/opt/ml/code/inference.py"]
   ```
   - Cursor: Open `Dockerfile`, paste code, save.

2. **Create Inference Script**
   ```python
   # inference.py
   import json
   import whisper
   from pyannote.audio import Pipeline
   import boto3
   import urllib.request

   def handler(event, context):
       s3_client = boto3.client('s3')
       model = whisper.load_model('medium')
       pipeline = Pipeline.from_pretrained('pyannote/speaker-diarization')

       audio_url = json.loads(event['body'])['audio_url']
       urllib.request.urlretrieve(audio_url, '/tmp/audio.opus')
       transcription = model.transcribe('/tmp/audio.opus')
       diarization = pipeline('/tmp/audio.opus')

       diarization_segments = [{'start': turn.start, 'end': turn.end, 'speaker': f"Speaker {i}", 'confidence': 0.9} for i, turn in enumerate(diarization.itertracks(yield_label=True))]
       transcription_segments = [{'start': seg['start'], 'end': seg['end'], 'text': seg['text']} for seg in transcription['segments']]

       return {
           'statusCode': 200,
           'body': json.dumps({'transcription': {'segments': transcription_segments}, 'diarization': {'segments': diarization_segments}})
       }
   ```
   - Cursor: Create `inference.py`, paste code, save.

3. **Build and Push Docker Image**
   ```bash
   aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin your-account-id.dkr.ecr.us-east-1.amazonaws.com
   docker build -t meeting-analysis-whisper .
   docker tag meeting-analysis-whisper your-account-id.dkr.ecr.us-east-1.amazonaws.com/meeting-analysis-whisper:latest
   docker push your-account-id.dkr.ecr.us-east-1.amazonaws.com/meeting-analysis-whisper:latest
   ```
   - Cursor: Run in terminal; replace `your-account-id` with your AWS account ID.

4. **Create SageMaker Endpoint**
   ```python
   # cdk_stack.py (update CDK app)
   from aws_cdk import core
   from aws_cdk import aws_sagemaker as sagemaker
   from aws_cdk import aws_ecr as ecr

   class MeetingAnalysisStack(core.Stack):
       def __init__(self, scope: core.Construct, id: str, **kwargs):
           super().__init__(scope, id, **kwargs)
           ecr_repo = ecr.Repository.from_repository_name(self, 'WhisperRepo', 'meeting-analysis-whisper')
           model = sagemaker.CfnModel(
               self, 'WhisperModel',
               execution_role_arn='arn:aws:iam::your-account-id:role/SageMakerRole',
               primary_container=sagemaker.CfnModel.ContainerDefinitionProperty(
                   image=f'your-account-id.dkr.ecr.us-east-1.amazonaws.com/meeting-analysis-whisper:latest'
               )
           )
           endpoint_config = sagemaker.CfnEndpointConfig(
               self, 'WhisperEndpointConfig',
               production_variants=[sagemaker.CfnEndpointConfig.ProductionVariantProperty(
                   variant_name='AllTraffic',
                   model_name=model.ref,
                   instance_type='ml.g4dn.xlarge',
                   initial_instance_count=1
               )]
           )
           sagemaker.CfnEndpoint(self, 'WhisperEndpoint', endpoint_config_name=endpoint_config.ref)
   ```
   - Cursor: Update CDK app (`app.py` or `cdk_stack.py`), add SageMaker stack, save.

### Testing Instructions
- **Docker Build**: Run `docker run -it meeting-analysis-whisper` with a test audio URL; verify JSON output.
- **SageMaker Endpoint**: Deploy CDK stack (`cdk deploy`); invoke endpoint with AWS CLI:
  ```bash
  aws sagemaker-runtime invoke-endpoint --endpoint-name WhisperEndpoint --body '{"audio_url": "s3://your-bucket/test.opus"}' output.json
  ```
  Check `output.json` for transcription and diarization.
- **Fix Issues**: If build fails, check Docker dependencies; if endpoint fails, verify IAM role and ECR permissions.

---

## Phase 10: Step Functions and API Gateway
**Objective**: Implement Step Functions for workflow orchestration and API Gateway for UI interactions.
**Estimated Time**: 3–4 hours

### Commands
1. **Create Step Functions Definition**
   ```json
   # step_functions.json
   {
       "StartAt": "UploadAudio",
       "States": {
           "UploadAudio": {
               "Type": "Task",
               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:AudioUploadHandler",
               "Next": "TranscribeAudio"
           },
           "TranscribeAudio": {
               "Type": "Task",
               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:StartTranscriptionHandler",
               "Next": "ConsolidateTranscript"
           },
           "ConsolidateTranscript": {
               "Type": "Task",
               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:TranscriptConsolidationHandler",
               "Next": "AnalyzeMeeting"
           },
           "AnalyzeMeeting": {
               "Type": "Parallel",
               "Branches": [
                   {
                       "StartAt": "Agent1",
                       "States": {
                           "Agent1": {
                               "Type": "Task",
                               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:Agent1Handler",
                               "End": true
                           }
                       }
                   },
                   {
                       "StartAt": "Agent2",
                       "States": {
                           "Agent2": {
                               "Type": "Task",
                               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:Agent2Handler",
                               "End": true
                           }
                       }
                   },
                   {
                       "StartAt": "Agent3",
                       "States": {
                           "Agent3": {
                               "Type": "Task",
                               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:Agent3Handler",
                               "End": true
                           }
                       }
                   }
               ],
               "Next": "NotifyUsers"
           },
           "NotifyUsers": {
               "Type": "Task",
               "Resource": "arn:aws:lambda:us-east-1:your-account-id:function:NotifyUsersHandler",
               "End": true
           }
       }
   }
   ```
   - Cursor: Create `step_functions.json`, paste code, save. Replace `your-account-id` with your AWS account ID.

2. **Create NotifyUsersHandler**
   ```python
   # src/notify_users/lambda_function.py
   import json
   import boto3
   import logging

   logger = logging.getLogger()
   logger.setLevel(logging.INFO)

   def lambda_handler(event, context):
       try:
           client = boto3.client('apigatewaymanagementapi', endpoint_url=os.getenv('WEBSOCKET_URL'))
           meeting_id = event['meetingId']
           client.post_to_connection(
               ConnectionId=event['connectionId'],
               Data=json.dumps({'meetingId': meeting_id, 'status': 'Completed'})
           )
           return {'statusCode': 200}
       except Exception as e:
           logger.error(f"Error: {str(e)}")
           return {'statusCode': 500}
   ```
   - Cursor: Create `src/notify_users/lambda_function.py`, paste code, save.

3. **Update CDK Stack for Step Functions and API Gateway**
   ```python
   # cdk_stack.py (update)
   from aws_cdk import core
   from aws_cdk import aws_lambda as _lambda
   from aws_cdk import aws_apigateway as apigateway
   from aws_cdk import aws_stepfunctions as sfn
   from aws_cdk import aws_stepfunctions_tasks as tasks

   class MeetingAnalysisStack(core.Stack):
       def __init__(self, scope: core.Construct, id: str, **kwargs):
           super().__init__(scope, id, **kwargs)

           # Lambda functions (example for AudioUploadHandler)
           audio_upload = _lambda.Function(self, 'AudioUploadHandler', runtime=_lambda.Runtime.PYTHON_3_9, handler='lambda_function.lambda_handler', code=_lambda.Code.from_asset('src/audio_upload'))
           # Add other Lambda functions similarly

           # Step Functions
           with open('step_functions.json') as f:
               definition = json.load(f)
           state_machine = sfn.StateMachine(self, 'MeetingAnalysisStateMachine', definition=definition)

           # API Gateway
           api = apigateway.RestApi(self, 'MeetingAnalysisApi')
           meetings = api.root.add_resource('meetings')
           meeting_id = meetings.add_resource('{meetingId}')
           meeting_id.add_method('POST', apigateway.LambdaIntegration(audio_upload), authorization_type=apigateway.AuthorizationType.COGNITO)
           meeting_id.add_resource('analyze').add_method('POST', apigateway.LambdaIntegration(state_machine))
           meeting_id.add_resource('analysis').add_method('GET', apigateway.LambdaIntegration(get_analysis))
           meeting_id.add_resource('validate').add_method('POST', apigateway.LambdaIntegration(validate_access_code))

           # WebSocket API
           websocket = apigateway.WebSocketApi(self, 'MeetingAnalysisWebSocket')
           websocket.add_route('default', integration=apigateway.WebSocketLambdaIntegration('NotifyUsers', notify_users))
   ```
   - Cursor: Update `cdk_stack.py`, add Step Functions and API Gateway, save.

### Testing Instructions
- **Step Functions**: Deploy CDK stack; start execution with AWS CLI:
  ```bash
  aws stepfunctions start-execution --state-machine-arn arn:aws:states:us-east-1:your-account-id:stateMachine:MeetingAnalysisStateMachine --input '{"meetingId": "test123"}'
  ```
  Check AWS Console for execution status.
- **API Gateway**: Test endpoints with Postman or `curl`:
  ```bash
  curl -X POST https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/meetings/test123/audio -H "Authorization: Bearer your-token" --form audio=@test.opus
  ```
  Verify S3 upload and DynamoDB status.
- **WebSocket**: Connect with a WebSocket client (e.g., wscat); send mock notification; verify message receipt.
- **Fix Issues**: If Step Functions fail, check Lambda ARNs; if API Gateway fails, verify Cognito integration.

---

## Phase 11: Cognito and Security
**Objective**: Set up Amazon Cognito for authentication and secure API endpoints.
**Estimated Time**: 2–3 hours

### Commands
1. **Create Cognito User Pool**
   ```bash
   aws cognito-idp create-user-pool --pool-name MeetingAnalysisPool --auto-verified-attributes email
   ```
   - Cursor: Run in terminal; note the `UserPoolId`.

2. **Create Cognito Client**
   ```bash
   aws cognito-idp create-user-pool-client --user-pool-id your-user-pool-id --client-name MeetingAnalysisClient --callback-urls "https://your-domain.com" --allowed-o-auth-flows user_password implicit --allowed-o-auth-scopes email openid
   ```
   - Cursor: Run in terminal; note the `ClientId`.

3. **Update CDK Stack for Cognito**
   ```python
   # cdk_stack.py (update)
   from aws_cdk import aws_cognito as cognito

   class MeetingAnalysisStack(core.Stack):
       def __init__(self, scope: core.Construct, id: str, **kwargs):
           super().__init__(scope, id, **kwargs)
           user_pool = cognito.UserPool(self, 'MeetingAnalysisPool', auto_verify_attributes=[cognito.AutoVerifiedAttrs.EMAIL])
           client = user_pool.add_client('MeetingAnalysisClient', o_auth=cognito.OAuthSettings(
               callback_urls=['https://your-domain.com'],
               flows=cognito.OAuthFlows(implicit=True),
               scopes=[cognito.OAuthScope.EMAIL, cognito.OAuthScope.OPENID]
           ))
           # Add to API Gateway authorizer
           api = apigateway.RestApi(self, 'MeetingAnalysisApi', default_cors_preflight_options=apigateway.CorsOptions(
               allow_origins=apigateway.Cors.ALL_ORIGINS,
               allow_methods=['GET', 'POST']
           ))
           api.root.add_resource('meetings').add_resource('{meetingId}').add_method(
               'POST',
               apigateway.LambdaIntegration(audio_upload),
               authorizer=apigateway.CognitoUserPoolsAuthorizer(self, 'Authorizer', cognito_user_pools=[user_pool])
           )
   ```
   - Cursor: Update `cdk_stack.py`, add Cognito, save.

### Testing Instructions
- **Cognito Setup**: Run `aws cognito-idp describe-user-pool --user-pool-id your-user-pool-id` to verify pool exists.
- **Login**: Access Cognito Hosted UI (`https://your-domain.auth.us-east-1.amazoncognito.com/login?client_id=your-client-id&response_type=token&redirect_uri=https://your-domain.com`); sign up and log in; verify JWT token in URL.
- **API Security**: Call API with JWT token:
  ```bash
  curl -X POST https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/meetings/test123/audio -H "Authorization: Bearer your-jwt-token" --form audio=@test.opus
  ```
  Verify 200 or 403 response.
- **Fix Issues**: If login fails, check client ID and callback URL; if API fails, verify authorizer configuration.

---

## Phase 12: Deployment and Monitoring
**Objective**: Deploy the system to AWS, set up monitoring, and configure data retention.
**Estimated Time**: 3–4 hours

### Commands
1. **Deploy CDK Stack**
   ```bash
   cdk deploy
   ```
   - Cursor: Run in terminal; confirm stack creation in AWS Console.

2. **Set Up CloudWatch Alarms**
   ```bash
   aws cloudwatch put-metric-alarm --alarm-name LambdaErrors --metric-name Errors --namespace AWS/Lambda --threshold 10 --comparison-operator GreaterThanThreshold --period 300 --evaluation-periods 1 --alarm-actions arn:aws:sns:us-east-1:your-account-id:Notify
   ```
   - Cursor: Run in terminal; repeat for SageMaker latency, DynamoDB throttling.

3. **Configure S3 Lifecycle Policy**
   ```json
   # s3_lifecycle.json
   {
       "Rules": [
           {
               "ID": "Archive",
               "Status": "Enabled",
               "Prefix": "uploads/",
               "Transitions": [
                   {
                       "Days": 30,
                       "StorageClass": "GLACIER"
                   }
               ]
           },
           {
               "ID": "Delete",
               "Status": "Enabled",
               "Prefix": "uploads/",
               "Expiration": {
                   "Days": 90
               }
           }
       ]
   }
   aws s3api put-bucket-lifecycle-configuration --bucket your-uploads-bucket --lifecycle-configuration file://s3_lifecycle.json
   ```
   - Cursor: Create `s3_lifecycle.json`, paste code, run command.

4. **Configure DynamoDB TTL**
   ```bash
   aws dynamodb update-time-to-live --table-name MeetingAnalysisTable --time-to-live-specification "Enabled=true,AttributeName=ttl"
   ```
   - Cursor: Run in terminal; ensure `ttl` attribute is added to table schema.

### Testing Instructions
- **Deployment**: Check AWS Console for Lambda functions, S3 buckets, DynamoDB table, SageMaker endpoint, Step Functions, API Gateway, Cognito.
- **Monitoring**: Trigger a Lambda error (e.g., invalid S3 key); verify CloudWatch alarm triggers SNS notification.
- **Lifecycle Policy**: Upload a test file to S3; wait 30 days or use AWS Console to simulate; verify file moves to Glacier.
- **TTL**: Add a record with `ttl` attribute (Unix timestamp + 90 days); wait or simulate; verify record deletion.
- **Fix Issues**: If deployment fails, check CDK logs; if alarms fail, verify SNS topic; if lifecycle fails, check bucket permissions.

---

## Phase 13: End-to-End Testing and Documentation
**Objective**: Perform end-to-end tests and create user documentation.
**Estimated Time**: 2–3 hours

### Commands
1. **Create End-to-End Test Script**
   ```python
   # tests/e2e_test.py
   import boto3
   import requests
   import time

   def test_e2e():
       s3_client = boto3.client('s3')
       dynamodb = boto3.resource('dynamodb')
       api_base = 'https://your-api-id.execute-api.us-east-1.amazonaws.com/prod'
       token = 'your-jwt-token'
       meeting_id = 'test123'
       access_codes = {'client': 'client123', 'salesperson': 'sales123'}

       # Upload audio
       with open('test.opus', 'rb') as f:
           response = requests.post(f"{api_base}/meetings/{meeting_id}/audio", headers={'Authorization': f"Bearer {token}"}, files={'audio': f})
       assert response.status_code == 200

       # Trigger analysis
       response = requests.post(f"{api_base}/meetings/{meeting_id}/analyze", headers={'Authorization': f"Bearer {token}"}, json={'meetingId': meeting_id, 'accessCodes': access_codes})
       assert response.status_code == 200

       # Wait for completion
       table = dynamodb.Table('MeetingAnalysisTable')
       for _ in range(60):
           response = table.get_item(Key={'meetingId': meeting_id, 'agentId': 'agent1'})
           if 'Item' in response:
               break
           time.sleep(5)

       # Verify results
       response = requests.get(f"{api_base}/meetings/{meeting_id}/analysis?agent=agent1", headers={'Authorization': f"Bearer {token}"})
       assert response.status_code == 200
       assert 'analysis' in response.json()

   if __name__ == '__main__':
       test_e2e()
   ```
   - Cursor: Create `tests/e2e_test.py`, paste code, save. Replace `api_base` and `token`.

2. **Create User Guide**
   ```markdown
   # Meeting Analysis System User Guide

   ## Getting Started
   1. Open the Recorder UI at `https://your-domain.com/recorder`.
   2. Log in with your email and password.
   3. Click "Start" to record, grant microphone access.
   4. Note the Meeting ID and access codes for Client and Salesperson.

   ## Recording
   - Click "Pause" to pause, "Resume" to continue, "Finish" to process.
   - Wait for analysis (2–3 minutes for a 30-minute meeting).
   - Download the PDF summary from the Recorder UI.

   ## Client/Salesperson UI
   1. Open `https://your-domain.com/client` or `https://your-domain.com/salesperson`.
   2. Enter Meeting ID and access code.
   3. View analysis results.

   ## Troubleshooting
   - **Microphone Error**: Ensure browser permissions are enabled.
   - **Invalid Code**: Verify Meeting ID and access code.
   ```
   - Cursor: Create `docs/user_guide.md`, paste code, save.

### Testing Instructions
- **End-to-End Test**: Run `python tests/e2e_test.py` with a test audio file (`test.opus`); verify analysis in DynamoDB and PDF in S3.
- **User Guide**: Open `user_guide.md` in a Markdown viewer; verify content is clear and accurate.
- **Manual Test**: Follow user guide steps in a browser; record a 1-minute meeting, share codes, verify results in all UIs.
- **Fix Issues**: If tests fail, check API responses and logs; if guide is unclear, revise based on user feedback.

---

## Summary
- **Phases**: 13 modular phases cover setup, backend, frontend, deployment, security, and testing.
- **Commands**: Specific Cursor commands for file creation, code pasting, and terminal execution.
- **Testing**: Syntax checks, unit tests, manual tests, and end-to-end tests ensure correctness.
- **Total Time**: ~30–40 hours (spread over days, depending on experience).

Execute each phase sequentially, testing thoroughly before proceeding. If you need help with specific commands, test debugging, or AWS setup, let me know!