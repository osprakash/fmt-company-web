# File & Asset Service - Technical Specification

> **Service**: File & Asset Service  
> **Version**: 1.0  
> **Owner**: Platform Team  
> **Last Updated**: November 2025

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        File Service                                      │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────────────────┐  │
│  │ Upload Handler │  │ Image Processor│  │   Access Controller      │  │
│  └────────────────┘  └────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
       ┌──────────┐     ┌──────────┐     ┌──────────┐
       │ S3/Blob  │     │   CDN    │     │  Antivirus│
       │ Storage  │     │(CloudFront)│   │ (ClamAV) │
       └──────────┘     └──────────┘     └──────────┘
```

---

## 2. Data Model

```sql
CREATE TABLE files (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    tenant_id UUID NOT NULL,
    uploaded_by UUID NOT NULL,
    
    filename VARCHAR(255) NOT NULL,
    original_filename VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100) NOT NULL,
    size_bytes BIGINT NOT NULL,
    
    storage_path VARCHAR(500) NOT NULL,
    storage_provider VARCHAR(20) NOT NULL DEFAULT 's3',
    
    -- Image specific
    width INTEGER,
    height INTEGER,
    
    -- Security
    checksum_sha256 CHAR(64) NOT NULL,
    virus_scanned_at TIMESTAMP WITH TIME ZONE,
    virus_scan_result VARCHAR(20),
    
    -- Access
    visibility VARCHAR(20) NOT NULL DEFAULT 'private',
    
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_files_tenant ON files(tenant_id);
CREATE INDEX idx_files_uploader ON files(uploaded_by);
```

---

## 3. API Contracts

### 3.1 Upload File

```yaml
POST /v1/files/upload

Request:
  Content-Type: multipart/form-data
  Body:
    file: binary
    visibility: string (private, public)
    folder: string (optional)

Response: 201 Created
  {
    "success": true,
    "data": {
      "id": "uuid",
      "filename": "document.pdf",
      "mimeType": "application/pdf",
      "sizeBytes": 102400,
      "url": "https://cdn.flowmind.io/files/xxx",
      "createdAt": "2025-11-29T10:00:00Z"
    }
  }
```

### 3.2 Get Signed URL

```yaml
POST /v1/files/{id}/signed-url

Request:
  {
    "expiresIn": 3600,
    "disposition": "attachment"
  }

Response: 200 OK
  {
    "success": true,
    "data": {
      "url": "https://s3.../file?signature=xxx",
      "expiresAt": "2025-11-29T11:00:00Z"
    }
  }
```

### 3.3 Image Transformations

```yaml
GET /v1/files/{id}/transform

Query Parameters:
  w: integer (width)
  h: integer (height)
  fit: string (cover, contain, fill)
  format: string (webp, jpeg, png)
  quality: integer (1-100)

Response: 302 Redirect to transformed image URL
```

---

## 4. Storage Strategy

```typescript
// Storage path: /{tenant_id}/{year}/{month}/{uuid}.{ext}
function generateStoragePath(file: UploadedFile, tenantId: string): string {
  const now = new Date();
  const ext = path.extname(file.originalName);
  const id = uuid();
  return `${tenantId}/${now.getFullYear()}/${now.getMonth() + 1}/${id}${ext}`;
}

// Upload to S3
async function uploadToS3(file: Buffer, path: string): Promise<string> {
  await s3.putObject({
    Bucket: BUCKET_NAME,
    Key: path,
    Body: file,
    ContentType: mimeType,
    ServerSideEncryption: 'AES256'
  });
  return path;
}
```

---

*FlowMind Technologies | File & Asset Service Technical Specification v1.0*

