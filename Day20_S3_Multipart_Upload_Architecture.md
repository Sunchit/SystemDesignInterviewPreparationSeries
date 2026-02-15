# Block File Uploads: How Dropbox & Netflix Upload 5TB Files to S3
### Day 20 of 50 - System Design Interview Preparation Series

**By Sunchit Dudeja**

---

## 🎯 Welcome to Day 20!

Yesterday, we explored Bloom Filters for email uniqueness. Today, we dive into a critical infrastructure pattern: **How do applications like Dropbox, Netflix, and Google Drive upload massive files (100MB to 5TB) reliably to cloud storage?**

> "The secret isn't uploading one giant file. It's splitting it into chunks, uploading in parallel, and letting the cloud assemble them server-side."

---

## 🤔 THE PROBLEM WITH LARGE FILE UPLOADS

### The Challenge

```
┌─────────────────────────────────────────────────────────────────┐
│              THE LARGE FILE UPLOAD NIGHTMARE                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Scenario: User uploads 2GB video file                          │
│                                                                  │
│   TRADITIONAL APPROACH (Single PUT):                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Client ──────── 2GB in one request ──────────▶ Server  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Problems:                                                      │
│   ❌ Network drops at 1.8GB → Start over from 0                 │
│   ❌ 30-minute upload → Connection timeout                      │
│   ❌ Server memory → OOM with concurrent uploads                │
│   ❌ Progress bar → All or nothing                              │
│   ❌ Single point of failure → No parallelism                   │
│                                                                  │
│   MULTIPART APPROACH (Chunked Upload):                           │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Client ─┬─ 100MB ─▶ S3  ← Parallel!                    │  │
│   │          ├─ 100MB ─▶ S3                                  │  │
│   │          ├─ 100MB ─▶ S3                                  │  │
│   │          └─ ...    ─▶ S3                                  │  │
│   │                                                          │  │
│   │  S3: Assembles all parts into final object              │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Benefits:                                                      │
│   ✅ Network drops at 1.8GB → Resume from part 18               │
│   ✅ 4 parallel uploads → 4x faster                             │
│   ✅ Server never sees data → Scalable                          │
│   ✅ Progress bar → Per-part tracking                           │
│   ✅ S3 assembles → No re-upload                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ HIGH-LEVEL ARCHITECTURE

### The Hybrid Client-Backend-S3 Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│              S3 MULTIPART UPLOAD - ARCHITECTURE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────┐      ┌──────────────┐      ┌──────────────┐     │
│   │  CLIENT  │      │   BACKEND    │      │   AWS S3     │     │
│   │ (Browser │◄────▶│   SERVICE    │◄────▶│              │     │
│   │  / App)  │      │ (Credentials)│      │              │     │
│   └──────────┘      └──────────────┘      └──────────────┘     │
│        │                                         ▲              │
│        │                                         │              │
│        └──────── DIRECT UPLOAD ─────────────────┘              │
│                  (via Presigned URLs)                           │
│                                                                  │
│   KEY INSIGHT:                                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Backend handles: Credentials, URLs, Completion          │  │
│   │  Client handles:  Chunking, Parallel Upload, Progress    │  │
│   │  S3 handles:      Storage, Assembly, Integrity           │  │
│   │                                                          │  │
│   │  Backend NEVER sees file data → Infinitely scalable!     │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Why Presigned URLs?

```
┌─────────────────────────────────────────────────────────────────┐
│              WHY PRESIGNED URLs?                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   WITHOUT PRESIGNED URLs:                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Option A: Expose AWS credentials to client             │  │
│   │            → SECURITY DISASTER! ❌                       │  │
│   │                                                          │  │
│   │  Option B: Proxy all data through backend               │  │
│   │            → Backend becomes bottleneck! ❌              │  │
│   │            → Memory exhaustion with large files! ❌      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   WITH PRESIGNED URLs:                                           │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  1. Backend generates time-limited signed URL           │  │
│   │  2. URL contains: bucket, key, expiry, signature        │  │
│   │  3. Client uploads DIRECTLY to S3 with this URL         │  │
│   │  4. No credentials exposed, no backend proxy            │  │
│   │                                                          │  │
│   │  ✅ SECURE + SCALABLE                                    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Presigned URL Example:                                         │
│   https://bucket.s3.amazonaws.com/video.mp4                     │
│   ?X-Amz-Algorithm=AWS4-HMAC-SHA256                             │
│   &X-Amz-Credential=AKIAIOSFODNN7EXAMPLE...                     │
│   &X-Amz-Date=20240115T120000Z                                  │
│   &X-Amz-Expires=3600                                            │
│   &X-Amz-Signature=fe5f80f77d5fa3beca038a248f...                │
│                                                                  │
│   Valid for 1 hour, only for specified operation               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📋 THE 6-STEP WORKFLOW

### Complete Interaction Flow

```
┌─────────────────────────────────────────────────────────────────┐
│              6-STEP MULTIPART UPLOAD WORKFLOW                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CLIENT              BACKEND              AWS S3                │
│     │                    │                    │                  │
│     │ ──── STEP 1 ──────▶│                    │                  │
│     │ POST /upload/init  │                    │                  │
│     │ {fileName, size}   │                    │                  │
│     │                    │ ──── STEP 2 ──────▶│                  │
│     │                    │ CreateMultipart    │                  │
│     │                    │ Upload             │                  │
│     │                    │◀────────────────── │                  │
│     │                    │     UploadId       │                  │
│     │                    │                    │                  │
│     │                    │ Generate Presigned │                  │
│     │                    │ URLs for each part │                  │
│     │◀──── STEP 3 ───────│                    │                  │
│     │ {uploadId, urls[]} │                    │                  │
│     │                    │                    │                  │
│     │ Split file into    │                    │                  │
│     │ chunks locally     │                    │                  │
│     │                    │                    │                  │
│     │ ═══════════════════════ STEP 4 ════════▶│                  │
│     │   PUT Part 1 (Direct via Presigned URL) │                  │
│     │ ════════════════════════════════════════▶│                  │
│     │   PUT Part 2 (Parallel!)                │                  │
│     │ ════════════════════════════════════════▶│                  │
│     │   PUT Part N                            │                  │
│     │◀════════════════════════════════════════ │                  │
│     │   ETags for each part                   │                  │
│     │                    │                    │                  │
│     │ ──── STEP 5 ──────▶│                    │                  │
│     │ POST /upload/done  │                    │                  │
│     │ {uploadId, parts[]}│                    │                  │
│     │                    │ ──── STEP 6 ──────▶│                  │
│     │                    │ CompleteMultipart  │                  │
│     │                    │ Upload             │                  │
│     │                    │◀────────────────── │                  │
│     │                    │   Final Object     │                  │
│     │◀───────────────────│   Created!         │                  │
│     │      SUCCESS!      │                    │                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 STEP-BY-STEP IMPLEMENTATION

### Step 1: Client Initiates Upload

```javascript
// Client-side: Request to start upload
async function initiateUpload(file) {
    const response = await fetch('/api/upload/initiate', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            fileName: file.name,
            fileSize: file.size,
            contentType: file.type
        })
    });
    
    return await response.json();
    // Returns: { uploadId, presignedUrls, key }
}
```

### Step 2: Backend Creates Multipart Upload in S3

```java
// Backend: Spring Boot Controller
@PostMapping("/api/upload/initiate")
public UploadInitResponse initiateUpload(@RequestBody UploadRequest request) {
    // 1. Create multipart upload in S3
    CreateMultipartUploadRequest createRequest = CreateMultipartUploadRequest.builder()
        .bucket(bucketName)
        .key(request.getFileName())
        .contentType(request.getContentType())
        .build();
    
    CreateMultipartUploadResponse response = s3Client.createMultipartUpload(createRequest);
    String uploadId = response.uploadId();
    
    // 2. Calculate number of parts
    long partSize = 10 * 1024 * 1024; // 10MB chunks
    int totalParts = (int) Math.ceil((double) request.getFileSize() / partSize);
    
    // 3. Generate presigned URLs for each part
    List<PresignedUrl> presignedUrls = new ArrayList<>();
    for (int partNumber = 1; partNumber <= totalParts; partNumber++) {
        UploadPartRequest uploadPartRequest = UploadPartRequest.builder()
            .bucket(bucketName)
            .key(request.getFileName())
            .uploadId(uploadId)
            .partNumber(partNumber)
            .build();
        
        UploadPartPresignRequest presignRequest = UploadPartPresignRequest.builder()
            .signatureDuration(Duration.ofHours(1))
            .uploadPartRequest(uploadPartRequest)
            .build();
        
        PresignedUploadPartRequest presigned = s3Presigner.presignUploadPart(presignRequest);
        presignedUrls.add(new PresignedUrl(partNumber, presigned.url().toString()));
    }
    
    return new UploadInitResponse(uploadId, presignedUrls, request.getFileName());
}
```

### Step 3 & 4: Client Chunks and Uploads in Parallel

```javascript
// Client-side: Parallel chunk upload
const CHUNK_SIZE = 10 * 1024 * 1024; // 10MB
const CONCURRENCY = 4; // 4 parallel uploads

async function uploadFile(file, initResponse) {
    const { uploadId, presignedUrls, key } = initResponse;
    const totalParts = presignedUrls.length;
    const completedParts = [];
    
    // Upload parts with controlled concurrency
    for (let i = 0; i < totalParts; i += CONCURRENCY) {
        const batch = presignedUrls.slice(i, i + CONCURRENCY);
        
        const batchPromises = batch.map(async ({ partNumber, url }) => {
            const start = (partNumber - 1) * CHUNK_SIZE;
            const end = Math.min(start + CHUNK_SIZE, file.size);
            const chunk = file.slice(start, end);
            
            // Upload with retry logic
            const etag = await uploadPartWithRetry(url, chunk, 3);
            
            completedParts.push({ PartNumber: partNumber, ETag: etag });
            updateProgress(completedParts.length, totalParts);
        });
        
        await Promise.all(batchPromises);
    }
    
    // Sort by part number for completion
    completedParts.sort((a, b) => a.PartNumber - b.PartNumber);
    
    return { uploadId, key, parts: completedParts };
}

async function uploadPartWithRetry(url, chunk, maxRetries) {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
        try {
            const response = await fetch(url, {
                method: 'PUT',
                body: chunk,
                headers: { 'Content-Type': 'application/octet-stream' }
            });
            
            if (!response.ok) throw new Error(`HTTP ${response.status}`);
            
            // ETag is returned in response header
            return response.headers.get('ETag');
        } catch (error) {
            if (attempt === maxRetries) throw error;
            // Exponential backoff
            await new Promise(r => setTimeout(r, 1000 * Math.pow(2, attempt)));
        }
    }
}
```

### Step 5 & 6: Complete the Multipart Upload

```java
// Backend: Complete the upload
@PostMapping("/api/upload/complete")
public UploadCompleteResponse completeUpload(@RequestBody CompleteRequest request) {
    List<CompletedPart> completedParts = request.getParts().stream()
        .map(p -> CompletedPart.builder()
            .partNumber(p.getPartNumber())
            .eTag(p.getETag())
            .build())
        .collect(Collectors.toList());
    
    CompleteMultipartUploadRequest completeRequest = CompleteMultipartUploadRequest.builder()
        .bucket(bucketName)
        .key(request.getKey())
        .uploadId(request.getUploadId())
        .multipartUpload(CompletedMultipartUpload.builder()
            .parts(completedParts)
            .build())
        .build();
    
    CompleteMultipartUploadResponse response = s3Client.completeMultipartUpload(completeRequest);
    
    return new UploadCompleteResponse(
        response.location(),
        response.eTag(),
        response.key()
    );
}
```

---

## 🔧 S3 INTERNAL OPERATIONS

### What Happens Inside S3

```
┌─────────────────────────────────────────────────────────────────┐
│              S3 INTERNAL PROCESSING                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   STEP 1: CreateMultipartUpload                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • Generates unique UploadId (UUID)                      │  │
│   │  • Creates metadata entry in S3 index                    │  │
│   │  • Allocates temporary storage space                     │  │
│   │  • NO data transfer yet                                  │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   STEP 2: UploadPart (for each part)                            │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • Receives: PUT /bucket/key?partNumber=1&uploadId=xyz  │  │
│   │  • Validates: UploadId exists, PartNumber 1-10000       │  │
│   │  • Stores in TEMPORARY location (not final bucket)      │  │
│   │  • Computes MD5 hash of content                         │  │
│   │  • Returns ETag = "md5hash" (quoted!)                    │  │
│   │  • Updates metadata: {PartNumber, ETag, Size, Location} │  │
│   │                                                          │  │
│   │  Parts can be uploaded in ANY order!                     │  │
│   │  Parts can be OVERWRITTEN (same PartNumber)!             │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   Temporary Storage:                                             │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  UploadId: VXBsb2FkSWQtMTIz...                          │  │
│   ├─────────────────────────────────────────────────────────┤  │
│   │  Part 1: ETag="abc123", Size=10MB, Loc=/tmp/p1          │  │
│   │  Part 2: ETag="def456", Size=10MB, Loc=/tmp/p2          │  │
│   │  Part 3: ETag="ghi789", Size=10MB, Loc=/tmp/p3          │  │
│   │  Part 4: ETag="jkl012", Size=5MB,  Loc=/tmp/p4 (last)   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   STEP 3: CompleteMultipartUpload (THE MAGIC!)                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  1. Receives: POST with ordered {PartNumber, ETag} list │  │
│   │                                                          │  │
│   │  2. VALIDATES:                                           │  │
│   │     • All parts exist                                    │  │
│   │     • ETags match (integrity check!)                     │  │
│   │     • Parts in ascending order                           │  │
│   │     • All parts >= 5MB (except last)                     │  │
│   │                                                          │  │
│   │  3. SERVER-SIDE ASSEMBLY:                                │  │
│   │     • Concatenates parts into single object              │  │
│   │     • Uses internal pointers (NOT byte copying!)        │  │
│   │     • NO data re-upload required!                        │  │
│   │                                                          │  │
│   │  4. Creates final object at s3://bucket/key              │  │
│   │  5. Deletes temporary parts storage                      │  │
│   │  6. Returns final ETag: "abc123-4" (4 = part count)      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   SERVER-SIDE ASSEMBLY (No Re-Upload!):                         │
│                                                                  │
│     [Part 1: 10MB] ─┐                                            │
│     [Part 2: 10MB] ─┼──▶ [Final Object: 35MB]                   │
│     [Part 3: 10MB] ─┤    s3://bucket/video.mp4                  │
│     [Part 4: 5MB]  ─┘    ETag: "xyz789-4"                       │
│                                ▲                                 │
│                                │                                 │
│                          "-4" = 4 parts                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚠️ S3 CONSTRAINTS

### Critical Limits to Know

```
┌─────────────────────────────────────────────────────────────────┐
│              S3 MULTIPART UPLOAD CONSTRAINTS                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┬─────────────┬─────────────┬─────────────────┐│
│   │ Part Size   │ Part Count  │ Object Size │ Additional      ││
│   ├─────────────┼─────────────┼─────────────┼─────────────────┤│
│   │ MIN: 5 MB   │ MIN: 1      │ MAX: 5 TB   │ UploadId never  ││
│   │ MAX: 5 GB   │ MAX: 10,000 │ = 5GB×10K   │ expires (until  ││
│   │ Last: any   │ Ascending   │             │ complete/abort) ││
│   └─────────────┴─────────────┴─────────────┴─────────────────┘│
│                                                                  │
│   PRACTICAL RECOMMENDATIONS:                                     │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • Part size: 10-100MB (balance between retries & speed) │  │
│   │  • Concurrency: 4-10 parallel uploads                    │  │
│   │  • Presigned URL expiry: 1 hour (configurable)           │  │
│   │  • Retry with exponential backoff                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   PART SIZE TRADE-OFFS:                                          │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Small parts (5-10MB):                                   │  │
│   │    ✅ Fast retry on failure (less data to re-upload)    │  │
│   │    ❌ More HTTP overhead                                 │  │
│   │    ❌ More presigned URLs to generate                    │  │
│   │                                                          │  │
│   │  Large parts (50-100MB):                                 │  │
│   │    ✅ Less HTTP overhead                                 │  │
│   │    ✅ Fewer parts to manage                              │  │
│   │    ❌ Slow retry on failure                              │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🚨 FAILURE HANDLING

### What Can Go Wrong (And How to Fix It)

```
┌─────────────────────────────────────────────────────────────────┐
│              FAILURE SCENARIOS & SOLUTIONS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   SCENARIO 1: Part Upload Fails                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Problem: Network drops during part 5 upload             │  │
│   │                                                          │  │
│   │  Solution:                                               │  │
│   │  • Retry with exponential backoff                        │  │
│   │  • Same presigned URL still valid (within expiry)        │  │
│   │  • Only re-upload part 5, not entire file!               │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   SCENARIO 2: Client Crashes Mid-Upload                          │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Problem: App crashes at 80% completion                  │  │
│   │                                                          │  │
│   │  Solution:                                               │  │
│   │  • Store {uploadId, completedParts} in localStorage      │  │
│   │  • On restart, call ListParts API to verify              │  │
│   │  • Resume from where you left off                        │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   SCENARIO 3: Incomplete Uploads (COST DANGER!)                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Problem: Upload started but never completed             │  │
│   │           Parts remain in S3, INCURRING CHARGES!         │  │
│   │                                                          │  │
│   │  Solutions:                                              │  │
│   │  1. AbortMultipartUpload API - Manual cleanup            │  │
│   │  2. S3 Lifecycle Rule - Auto-abort after N days:         │  │
│   │                                                          │  │
│   │     {                                                    │  │
│   │       "Rules": [{                                        │  │
│   │         "Status": "Enabled",                             │  │
│   │         "AbortIncompleteMultipartUpload": {              │  │
│   │           "DaysAfterInitiation": 7                       │  │
│   │         }                                                │  │
│   │       }]                                                 │  │
│   │     }                                                    │  │
│   │                                                          │  │
│   │  ⚠️  CRITICAL: Always set lifecycle rule in production! │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   SCENARIO 4: ETag Mismatch                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  Problem: Part corrupted during upload                   │  │
│   │           ETag stored by client ≠ ETag stored by S3     │  │
│   │                                                          │  │
│   │  Solution:                                               │  │
│   │  • CompleteMultipartUpload will FAIL                     │  │
│   │  • Re-upload corrupted part                              │  │
│   │  • S3's integrity check protects against corruption      │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Abort Incomplete Upload

```java
// Cleanup incomplete uploads
@DeleteMapping("/api/upload/abort")
public void abortUpload(@RequestBody AbortRequest request) {
    AbortMultipartUploadRequest abortRequest = AbortMultipartUploadRequest.builder()
        .bucket(bucketName)
        .key(request.getKey())
        .uploadId(request.getUploadId())
        .build();
    
    s3Client.abortMultipartUpload(abortRequest);
    // All parts for this uploadId are deleted
    // No storage charges after abort
}
```

---

## 📊 REAL-WORLD USE CASES

### Who Uses This Pattern?

```
┌─────────────────────────────────────────────────────────────────┐
│              COMPANIES USING S3 MULTIPART UPLOAD                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   DROPBOX                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • 2+ billion files synced daily                         │  │
│   │  • Max file size: 2GB (web), 50GB (desktop)             │  │
│   │  • Uses chunked upload with deduplication                │  │
│   │  • Resume interrupted uploads                            │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   NETFLIX                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • 4K HDR content: 20-50GB per movie                     │  │
│   │  • Multiple bitrate versions: 100GB+ total per title    │  │
│   │  • Global CDN population from S3 origin                  │  │
│   │  • Parallel regional uploads                             │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   YOUTUBE                                                        │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • 500+ hours uploaded every minute                      │  │
│   │  • Max upload: 256GB or 12 hours                        │  │
│   │  • Resumable uploads for reliability                     │  │
│   │  • Background processing while uploading                 │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   GOOGLE DRIVE                                                   │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │  • Max file size: 5TB (matches S3 limit!)               │  │
│   │  • Resumable uploads API                                 │  │
│   │  • Automatic retry on failure                            │  │
│   │  • Cross-platform sync                                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔧 USEFUL S3 APIs

### APIs for Managing Multipart Uploads

```
┌─────────────────────────────────────────────────────────────────┐
│              S3 MULTIPART UPLOAD APIs                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   CreateMultipartUpload                                          │
│   POST /bucket/key?uploads                                       │
│   → Returns: UploadId                                            │
│                                                                  │
│   UploadPart                                                     │
│   PUT /bucket/key?partNumber=N&uploadId=xyz                     │
│   → Returns: ETag                                                │
│                                                                  │
│   ListParts (for resuming)                                       │
│   GET /bucket/key?uploadId=xyz                                   │
│   → Returns: [{PartNumber, ETag, Size, LastModified}]           │
│                                                                  │
│   ListMultipartUploads (for cleanup monitoring)                  │
│   GET /bucket?uploads                                            │
│   → Returns: All incomplete uploads in bucket                    │
│                                                                  │
│   CompleteMultipartUpload                                        │
│   POST /bucket/key?uploadId=xyz                                  │
│   Body: Ordered list of {PartNumber, ETag}                      │
│   → Returns: Final object location, ETag                        │
│                                                                  │
│   AbortMultipartUpload                                           │
│   DELETE /bucket/key?uploadId=xyz                                │
│   → Deletes all parts, stops charges                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 💡 KEY ARCHITECTURAL INSIGHTS

### Why This Pattern Works

| Insight | Benefit |
|---------|---------|
| **Backend never sees file data** | Infinitely scalable, no memory issues |
| **Presigned URLs** | Secure, no credential exposure |
| **Parallel uploads** | 4-10x faster than sequential |
| **Server-side assembly** | No re-upload, S3 handles concatenation |
| **ETag verification** | Data integrity guaranteed |
| **Resumable** | Network failures don't lose progress |

---

## ❓ Interview Practice

### Question 1:
> "How would you design a file upload system that handles 5TB files?"

**Answer:**
> "I'd use S3 Multipart Upload with a hybrid architecture. The backend generates presigned URLs (for security) and the client uploads directly to S3 (for scalability). Files are split into 10-100MB chunks, uploaded in parallel (4-10 concurrent), and S3 assembles them server-side. This means the backend never sees file data, making it infinitely scalable. I'd add retry logic with exponential backoff, store progress in localStorage for resume capability, and set S3 lifecycle rules to auto-abort incomplete uploads after 7 days."

### Question 2:
> "Why use presigned URLs instead of proxying through the backend?"

**Answer:**
> "Two reasons: security and scalability. Exposing AWS credentials to the client is a security disaster. Proxying through the backend means every byte of every file flows through your servers - for a 5TB file, that's unsustainable. Presigned URLs give clients time-limited, operation-specific access to S3 directly. The backend handles metadata only, making it stateless and horizontally scalable. The actual file data goes client-to-S3 directly."

### Question 3:
> "What happens if a part upload fails at 80%?"

**Answer:**
> "Only that one part needs to be re-uploaded, not the entire file. The presigned URL is still valid within its expiry window. We implement retry with exponential backoff - wait 1s, 2s, 4s between attempts. If the client crashes completely, we store uploadId and completed parts in localStorage. On restart, we call S3's ListParts API to verify which parts are already uploaded, then resume from there. The key insight is that S3 keeps partial uploads indefinitely until we call Complete or Abort."

---

## 🔗 Connecting to Previous Days

| Day | Concept | How It Connects |
|-----|---------|-----------------|
| Day 15 | Redis Single-Threaded | Presigned URL caching |
| Day 18 | Redis Timeouts | Backend service resilience |
| Day 19 | Bloom Filters | Deduplication of uploaded chunks |

---

## ✅ Day 20 Action Items

1. **Implement a multipart uploader** with parallel chunking in your project
2. **Set up S3 Lifecycle Rules** to auto-abort incomplete uploads
3. **Add progress tracking** and resume capability
4. **Test failure scenarios** - network drops, client crashes

---

## 💡 Key Takeaways

| Insight | Why It Matters |
|---------|----------------|
| Presigned URLs = Secure + Scalable | No credential exposure, no backend bottleneck |
| Parallel uploads | 4-10x faster than sequential |
| Server-side assembly | No re-upload after parts complete |
| ETag = MD5 hash | Integrity verification built-in |
| Lifecycle rules | Prevent cost leakage from incomplete uploads |

---

## 🎯 The Architect's Principle

> **Junior:** "I'll just POST the entire file to my backend and upload to S3."
>
> **Architect:** "For a 5TB file? Your server will OOM and timeout. Instead, use presigned URLs so clients upload directly to S3. Split into 10MB chunks for parallel upload and resume capability. The backend only handles metadata - it never sees file bytes. S3 assembles parts server-side, so there's no re-upload. And always set lifecycle rules to auto-abort incomplete uploads, or you'll have mystery S3 bills from orphaned parts."

---

*— Sunchit Dudeja*  
*Day 20 of 50: System Design Interview Preparation Series*

---

> 🎯 **Interview Edge:** When asked about large file uploads, immediately mention: "Presigned URLs for security, chunked parallel upload for speed, server-side assembly for efficiency, and lifecycle rules for cost management."

> 📢 **Real Impact:** Dropbox uploads 2+ billion files daily using this pattern. Netflix uploads 50GB+ 4K content per title. YouTube handles 500+ hours uploaded every minute. The pattern is industry standard.

---

> 💡 **Tomorrow (Day 21):** We'll explore **Consistent Hashing** — how Netflix, Discord, and Amazon distribute data across thousands of servers without rehashing everything when nodes join or leave.

