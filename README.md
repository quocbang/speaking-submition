# Submit Flow Documentation

## Upload Sequence Flow

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant N as N8N
    participant S as Supabase
    participant G as Google Docs

    U->>F: Submit form with files
    F->>F: Validate form data
    F->>F: Generate sessionId

    loop For each file
        F->>N: GET presigned URL<br/>/webhook/speaking/submit<br/>?sessionId=X&email=Y&fileName=Z
        N-->>F: {url, token}
        F->>S: PUT file<br/>storage/v1{url}<br/>Content-Type: audio/*
        S-->>F: 200 OK
        F->>F: Update UI status
    end

    F->>N: POST finalize<br/>/webhook/speaking/submit<br/>{sessionId, email, fullName, phone, topic, from, class}
    N->>S: Process uploaded files
    N->>G: Create Google Doc
    G-->>N: Doc URL
    N-->>F: {doc_link}

    F->>U: Show success with doc link
    F->>F: Reset form (30s)
```

### Legacy Flow (for comparison)

```
User submits form
    ↓
Validate form data
    ↓
POST all files to N8N
    ↓
N8N uploads to Google Drive
    ↓
N8N creates Google Doc
    ↓
Return doc links
```

## Key Functions in Code

### `getPresignedUrl(sessionId, email, fileName)`
- **Purpose**: Retrieves presigned URL for file upload
- **Method**: GET
- **Endpoint**: `https://n8n.dungenglishspeaking.com/webhook/speaking/submit`
- **Parameters**: sessionId, email, fileName (query params)
- **Response**: `{ "url": "/object/upload/sign/.../filename?token=...", "token": "..." }`

### `uploadToSupabase(file, presignedData, fileIndex)`
- **Purpose**: Uploads file directly to Supabase storage
- **Method**: PUT
- **Endpoint**: `https://hwqxigmopqkiqmogkaxt.storage.supabase.co/storage/v1{url}`
- **Headers**: `Content-Type: file.type`
- **Body**: File blob data
- **Progress**: Tracks upload progress per file

### `finalizeSubmission(sessionId, email, fullName, phone, topic, from, class)`
- **Purpose**: Completes submission and generates document
- **Method**: POST
- **Endpoint**: `https://n8n.dungenglishspeaking.com/webhook/speaking/submit`
- **Headers**: `Content-Type: application/json`
- **Body** (when accessed via `/lop-4658` path):
  ```json
  {
    "sessionId": "submit_1234567890_abc123def",
    "email": "user@example.com",
    "fullName": "Nguyễn Văn A",
    "phone": "0901234567",
    "topic": "Giới thiệu bản thân",
    "from": "lop-4658",
    "class": "t46 chiều (dl thứ 4 hàng tuần)"
  }
  ```
- **Body** (when accessed via `/lop-275878` path):
  ```json
  {
    "sessionId": "submit_1234567890_abc123def",
    "email": "user@example.com",
    "fullName": "Nguyễn Văn A",
    "phone": "0901234567",
    "topic": "Giới thiệu bản thân",
    "from": "lop-275878",
    "class": "t27 (dl t7 hằng tuần)"
  }
  ```
- **Body** (when accessed via `/lop-243578` path):
  ```json
  {
    "sessionId": "submit_1234567890_abc123def",
    "email": "user@example.com",
    "fullName": "Nguyễn Văn A",
    "phone": "0901234567",
    "topic": "Giới thiệu bản thân",
    "from": "lop-243578",
    "class": "t24 (dl thứ 4 hàng tuần)"
  }
  ```
- **Body** (when accessed via regular path):
  ```json
  {
    "sessionId": "submit_1234567890_abc123def",
    "email": "user@example.com",
    "fullName": "Nguyễn Văn A",
    "phone": "0901234567",
    "topic": "Giới thiệu bản thân",
    "from": "default",
    "class": ""
  }
  ```
- **Response**: `{ "doc_link": "https://docs.google.com/document/d/..." }`

### `uploadSubmission(formData, sessionId)`
- **Purpose**: Orchestrates the entire upload process
- **Flow**:
  1. Loop through selected files
  2. For each file: getPresignedUrl → uploadToSupabase
  3. After all files: finalizeSubmission
  4. Return result with docLink

## Error Handling

- **Presigned URL failure**: Stops process, shows error
- **Supabase upload failure**: Stops process, shows error
- **Final submission failure**: Shows error message
- **Progress tracking**: Per-file progress bars
- **User protection**: Warns before leaving during upload

## UI States

- **File status**: "Chờ tải lên" → "Đang tải lên..." → "Hoàn thành ✓" / "Lỗi tải lên ✗"
- **Overall progress**: Combined progress bar (completed_files / total_files * 100)
- **Success message**: Shows file count + doc link
- **Form reset**: Automatic after 30 seconds

## Technical Details

- **File validation**: MP3, WAV, M4A only, max 50MB each
- **Progress tracking**: XMLHttpRequest upload progress events
- **Session management**: Unique sessionId per submission batch
- **Error isolation**: Failed file doesn't affect others (but stops batch)
- **Network handling**: Proper error responses and timeouts
