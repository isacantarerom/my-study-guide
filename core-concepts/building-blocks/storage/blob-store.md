# 📦 Blob Store

> *"A database stores structure. A blob store stores everything else."*

**⏱ Reading time:** ~11 minutes &nbsp;|&nbsp; **📍 Series:** Building Blocks &nbsp;|&nbsp; **🗂 Group:** Storage

---

## Table of Contents

1. [What a Blob Store Is](#1-what-a-blob-store-is)
2. [Why Not Just Use a Database?](#2-why-not-just-use-a-database)
3. [How a Blob Store Works](#3-how-a-blob-store-works)
4. [The Flat Namespace — No Real Folders](#4-the-flat-namespace--no-real-folders)
5. [Consistency Model](#5-consistency-model)
6. [Durability — How Blob Stores Survive Failures](#6-durability--how-blob-stores-survive-failures)
7. [Access Control](#7-access-control)
8. [Blob Store + CDN — The Standard Pattern](#8-blob-store--cdn--the-standard-pattern)
9. [Common Use Cases](#9-common-use-cases)
10. [Multipart Upload — Handling Large Files](#10-multipart-upload--handling-large-files)
11. [How Blob Stores Connect to Other Building Blocks](#11-how-blob-stores-connect-to-other-building-blocks)
12. [Self-Check](#12-self-check)
13. [References](#13-references)

---

## 1. What a Blob Store Is

**BLOB** stands for **Binary Large Object** — any large chunk of binary data without a predefined structure. Images, videos, audio files, PDFs, software packages, ML model weights, database backups, log archives — these are all blobs.

A blob store is a storage system designed specifically for these large, unstructured files. It stores them reliably at massive scale, serves them on demand, and handles the hard problems of durability and distribution without you managing any of it.

The three most widely used blob stores:
- **Amazon S3** (Simple Storage Service) — the original and most widely used
- **Google Cloud Storage (GCS)**
- **Azure Blob Storage**

All three work on the same fundamental model: you store a file at a key (called an **object key**), and you retrieve it by that key. The interface is simple; the infrastructure behind it is enormously complex.

---

## 2. Why Not Just Use a Database?

The natural question: if you already have a database, why not store files in it?

Some databases do support binary storage (PostgreSQL has `bytea` columns, for example). For small files, this occasionally makes sense. For large files at scale, it's almost always the wrong choice.

```
Storing a 5MB photo in PostgreSQL:

Write path:
  - Photo data written to database row
  - Transaction log updated
  - B-tree indexes updated
  - Data replicated to all replicas
  
Problems:
  - Database rows are loaded entirely into memory for processing
  - A 5MB column bloats every query that touches the row
  - Database backups now include all photo data (much larger, slower)
  - Database is optimized for structured queries, not binary streaming
  - Serving the photo requires going through your application server
  - You can't put a CDN in front of a database efficiently
```

Blob stores are purpose-built for this workload:
- Stream large files without loading into memory
- Serve directly to clients via URLs (no application server needed)
- Integrate natively with CDNs
- Scale storage independently from compute
- Achieve 11 nines durability (99.999999999%) through internal redundancy
- Charge per GB stored and GB transferred (no compute cost for pure storage)

The pattern is universal: **store metadata in the database, store the actual file in the blob store.**

```
Database row:
  photo_id:    12345
  user_id:     678
  caption:     "System design notes"
  blob_key:    "photos/user/678/photo_12345.jpg"  ← pointer to blob store
  created_at:  2024-01-15T10:30:00Z
  size_bytes:  3145728

Blob Store:
  Key: "photos/user/678/photo_12345.jpg"
  Value: [3MB of binary image data]
```

The database answers "what photos does user 678 have?" The blob store answers "give me the bytes for photo 12345."

---

## 3. How a Blob Store Works

The interface is intentionally simple:

```
PUT  /bucket/object-key  → upload a file, returns URL or confirmation
GET  /bucket/object-key  → download a file
DELETE /bucket/object-key → delete a file
HEAD /bucket/object-key  → get metadata without downloading content
LIST /bucket/prefix      → list all object keys with a given prefix
```

**Buckets** are the top-level containers — like a folder that contains all objects for one application or purpose. Objects live inside buckets.

**Object keys** are the unique identifier within a bucket — analogous to a file path, though there are no real folders (more on this in Section 4).

**The actual storage** is handled internally by the blob store — your files are chunked, replicated across multiple physical locations, and served from redundant infrastructure. You never manage disks, RAID arrays, or replication. You just PUT and GET objects.

---

## 4. The Flat Namespace — No Real Folders

This is the most important thing to understand about blob stores that trips people up: **there are no actual folders or directories.** The namespace is completely flat — every object in a bucket has a key, and keys are just strings.

What looks like a folder structure is an illusion created by using `/` as a delimiter in key names:

```
"photos/user/678/2024/01/photo1.jpg"
"photos/user/678/2024/01/photo2.jpg"
"photos/user/679/2024/02/photo3.jpg"
"videos/user/678/intro.mp4"
```

These look like files in a folder hierarchy, but they're actually just four keys in a flat namespace. The `/` is just a character in the key string — the blob store treats it as a delimiter only for listing operations.

**Why this matters:** You can't "move a folder" atomically — you'd have to copy each object individually and update all keys. You can't enforce that a "folder" exists before storing objects in it. And listing by prefix scans keys alphabetically, not by actual directory structure.

**The benefit:** The flat namespace is what enables massive parallelism. There are no directory locks, no inode tables, no tree structures to contend on. Every object is independent. This is part of why blob stores can handle millions of concurrent operations.

---

## 5. Consistency Model

S3 and most major blob stores now offer **strong read-after-write consistency** for new objects:

```
PUT object → success response
GET same object → immediately returns the new data (guaranteed)
```

This wasn't always the case — S3 originally had eventual consistency for some operations, which caused subtle bugs when code assumed a freshly uploaded file was immediately readable. The modern behavior is strongly consistent.

**However:** List operations (listing all objects with a prefix) may still have slight consistency delays in large buckets. In practice this is rarely a problem for typical use cases.

---

## 6. Durability — How Blob Stores Survive Failures

Amazon S3's advertised durability is **99.999999999% (11 nines)**. This means if you store 10 million objects, you can statistically expect to lose one object every 10,000 years. In practice this durability comes from aggressive redundancy:

```
You upload a photo
            │
            ▼
S3 internally:
  - Splits file into chunks
  - Stores copies across minimum 3 Availability Zones (separate data centers)
  - Uses erasure coding (like RAID) so data can be reconstructed even if multiple chunks are lost
  - Continuously verifies stored data via checksums (bit rot detection)
  - Automatically repairs any detected corruption
            │
            ▼
Object is durably stored
```

This durability is why blob stores replaced tape backups and on-premise file servers for most use cases. Building this level of durability yourself would require an enormous engineering investment.

**Important distinction:** Durability (data survives hardware failures) is different from availability (data is accessible). S3's availability SLA is 99.99% — you might occasionally get a 503 trying to read a file, but the file itself is almost certainly not lost.

---

## 7. Access Control

Blob stores have sophisticated access control since they may contain private user data.

**Bucket-level policies** — rules that apply to all objects in a bucket.
```
"This bucket is private — no public access allowed"
"Only users in this AWS account can access this bucket"
```

**Object-level ACLs** — access rules per individual object.
```
"This specific object is public-read"
"This object is accessible only by user X"
```

**Pre-signed URLs** — a powerful pattern for temporary access:

```
Your server generates a pre-signed URL:
  URL = sign("GET", "photos/user/678/photo_12345.jpg", expires=3600)
  → https://bucket.s3.amazonaws.com/photos/user/678/photo_12345.jpg
      ?X-Amz-Signature=abc123
      &X-Amz-Expires=3600

You give this URL to the user.
The user can access the photo for 1 hour — directly from S3, no server needed.
After 1 hour, the URL is invalid.
```

Pre-signed URLs are the standard pattern for serving private user content:
- User requests to view their private photo
- Your server checks authorization
- Your server generates a 1-hour pre-signed URL
- Your server returns the URL to the client
- Client fetches the photo directly from S3 (your server isn't in the data path)

This keeps your servers out of the bandwidth-intensive work of serving large files while still enforcing access control.

---

## 8. Blob Store + CDN — The Standard Pattern

For any content that's accessed frequently, the blob store alone isn't enough — you need a CDN in front of it to handle global distribution and reduce latency.

```
Standard architecture for user-generated content:

Upload path:
  Client → Your API server → Blob Store (S3)
           (validates, authorizes, then stores)

Serve path:
  Client → CDN Edge Node → (cache hit: serve immediately)
                        → (cache miss: fetch from S3, cache, serve)
```

The CDN caches objects at edge nodes globally. Users get images and videos from a server 20ms away instead of S3's origin servers potentially 200ms away.

**For public content** (profile photos, public posts): set objects to public-read in S3, point CDN at S3 bucket as origin.

**For private content**: use pre-signed URLs that include CDN cache authorization, or serve private content through your application servers (accepting the bandwidth cost for truly private data).

---

## 9. Common Use Cases

### User-Generated Media
Photos, videos, audio uploaded by users. The canonical use case. Instagram, YouTube, Spotify all use blob storage for the actual media files — databases hold only metadata.

```
Instagram user uploads photo:
  1. Client uploads to Instagram's API
  2. API validates (size, format, content policy)
  3. Stores original in S3: "originals/user/678/photo_12345.jpg"
  4. Kicks off async processing: resize to thumbnail, medium, large
  5. Stores processed versions: "processed/user/678/photo_12345_{size}.jpg"
  6. Stores metadata in database
  7. CDN serves all sizes globally
```

### Application Assets
JavaScript bundles, CSS, web fonts, static HTML. Your build pipeline uploads new versions on each deploy; users get files from CDN.

### Backups and Archives
Database backups, log archives, audit trails. Written periodically, rarely read, must be durable and cheap to store.

### ML Model Weights
Large binary files (gigabytes to terabytes) that need to be distributed to inference servers globally. Blob store + CDN or blob store + direct download to GPU servers.

### Software Distribution
Binary installers, container images, package repositories. GitHub Releases, npm, PyPI all use blob storage for the actual package files.

---

## 10. Multipart Upload — Handling Large Files

Uploading a 10GB video file as a single HTTP request is fragile — any network interruption means starting over. Blob stores provide **multipart upload** to handle large files reliably.

```
Multipart upload for a 10GB video:

1. Initiate multipart upload → get upload_id
2. Split file into 100MB chunks
3. Upload each chunk independently:
   PUT part 1 (100MB) → etag_1
   PUT part 2 (100MB) → etag_2
   ...
   PUT part 100 (100MB) → etag_100
   (chunks can be uploaded in parallel)
4. Complete multipart upload with list of etags
   → S3 assembles into final object

Benefits:
  - Resume after failure (re-upload only failed parts)
  - Parallel upload (upload multiple chunks simultaneously → faster)
  - Progress tracking (know which parts succeeded)
  - Memory-efficient (process one chunk at a time, not entire file)
```

For any file over ~100MB, multipart upload is the right approach.

---

## 11. How Blob Stores Connect to Other Building Blocks

```
Databases ──────────────────────────────────────────────────────────────►
  Store metadata (file name, size, user, created_at, blob_key)
  Blob key is the pointer from database row to blob store object

Blob Store ──────────────────────────────────────────────────────────────►
  Stores the actual binary content
  Source of truth for files

CDN ──────────────────────────────────────────────────────────────────────►
  Sits in front of blob store for public/semi-public content
  Caches at edge nodes globally — blob store only serves cache misses

Message Queue ────────────────────────────────────────────────────────────►
  After upload, queue a processing job
  (video transcoding, image resizing, virus scanning)
  Worker reads from queue, processes file in blob store, writes result back

Task Scheduler ───────────────────────────────────────────────────────────►
  Periodic cleanup jobs (delete expired files, archive old content)
  Reads object listing from blob store, processes in batches
```

---

## 12. Self-Check

1. What is a blob, and why is a blob store better than a database for storing large files?
2. What is the flat namespace model? Why are there no real folders in a blob store?
3. What does "11 nines durability" mean, and how do blob stores achieve it?
4. What is a pre-signed URL? Walk through how it enables private content delivery without your server being in the data path.
5. Why is a CDN almost always placed in front of a blob store for user-facing content?
6. A user uploads a 500MB video file. Your API tries to accept the entire upload and forward it to S3 in one request, but users on slow connections keep timing out. How do you fix this?
7. You're designing Instagram. Walk through the complete path of a photo from the moment a user taps "post" to the moment another user sees it in their feed. Which building blocks are involved at each step?

---

## 13. References

| Resource | Why it's worth it |
|----------|-------------------|
| 🔧 [AWS S3 Documentation](https://docs.aws.amazon.com/s3/) | The authoritative reference for S3 — especially access control and multipart upload |
| 📬 [ByteByteGo — Designing a Blob Store](https://bytebytego.com) | Visual walkthrough of blob store architecture from scratch |
| 📝 [AWS S3 Consistency Model](https://aws.amazon.com/blogs/storage/amazon-s3-update-strong-read-after-write-consistency/) | The announcement when S3 moved to strong consistency — explains the original model too |
| 📊 [Google Cloud — Object Storage Best Practices](https://cloud.google.com/storage/docs/best-practices) | Practical patterns for blob storage at scale |

---

*⬅️ Previous: [Key-Value Store](key-value-store.md) &nbsp;|&nbsp; ➡️ Next Group: [Speed](../speed/Speed.md)*

---

<sub>Part of the <a href="../../README.md">System Design Foundations</a> study guide series — Building Blocks: Storage.</sub>