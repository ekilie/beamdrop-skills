---
name: beamdrop
description: Interact with a Beamdrop file storage server — upload, download, and manage files via the S3-compatible API. Use when the user wants to store files, create buckets, generate presigned/shareable URLs, manage API keys, set up webhooks, connect via MCP, or integrate Beamdrop into their project. Covers Go SDK, TypeScript/JavaScript SDK (npm package), HTTP API, presigned URL strategies, webhooks, MCP server, and error handling.
license: MIT
metadata:
  author: "Tachera Sasi"
  repository: "https://github.com/ekilie/beamdrop-skills"
  version: "1.0.0"
  keywords: "beamdrop, ai, agent, skill, file-storage, s3, presigned-urls"
references:
  - references/agent-instructions.md
---

# Beamdrop File Storage Skill

## When to Use This Skill

Use this skill when the user asks to:

- Upload, download, or manage files on a Beamdrop server
- Create, list, or delete storage buckets
- Generate presigned or shareable download URLs
- Manage Beamdrop API keys (create, list, delete, scope permissions)
- Store AI-generated artifacts (code, images, documents, build outputs) on Beamdrop
- Set up Beamdrop integration in their Go, TypeScript/JavaScript (Node.js, Bun, Deno), Python, PHP, or any project
- Use the `beamdrop` npm package in Express.js, Elysia, Next.js, or any TS/JS framework
- Compare presigned URL types or decide on a file-sharing strategy
- Set up webhooks for real-time event notifications (object, bucket, share, presign events)
- Connect AI assistants via the built-in MCP server at `/mcp`
- Debug Beamdrop API errors (401, 404, 409, 429)
- Configure Beamdrop for production deployment

## Prerequisites

The user needs a running Beamdrop instance with API authentication enabled:

```bash
beamdrop -dir /path/to/share -api-auth
```

They need these environment variables or config values:

- `BEAMDROP_BASE_URL` — The server URL (e.g., `http://localhost:7777`)
- `BEAMDROP_ACCESS_KEY_ID` — API access key (format: `BDK_` + 16 hex chars)
- `BEAMDROP_SECRET_KEY` — API secret key (format: `sk_` + 40 hex chars)

If the user doesn't have API keys yet, they can create them:

```bash
curl -X POST http://localhost:7777/api/v1/keys
# Response includes accessKeyId and secretKey (secret shown ONCE — save immediately)
```

## Authentication Details

All S3 API requests at `/api/v1/` use HMAC-SHA256 signing:

```
StringToSign = METHOD + "\n" + PATH + "\n" + RFC3339_TIMESTAMP
Signature = Base64(HMAC-SHA256(StringToSign, SecretKey))
```

- PATH is the URL path only — no query string. Example: `/api/v1/buckets/my-bucket`
- TIMESTAMP is RFC3339 UTC (e.g., `2024-01-15T10:30:00Z`). Must be within ±15 minutes of server time.
- Base64 is standard encoding (not URL-safe)

Headers required on every request:

```
Authorization: Bearer BDK_xxxx:SIGNATURE
X-Beamdrop-Date: 2024-01-15T10:30:00Z
```

API key properties:

- `permissions`: `"read"`, `"write"`, or `"read,write"`
- `bucketScope`: Optional — restricts key to a single bucket
- `expiresAt`: Optional — key auto-expires after this time
- `disabled`: Can be set to temporarily disable without deleting

## Go Client SDK

When generating Go code, always use the official client SDK at `github.com/ekilie/beamdrop/pkg/client`:

```go
import "github.com/ekilie/beamdrop/pkg/client"

// Initialize client (validates BaseURL, creates HTTP client with 2-min timeout)
c, err := client.New(client.Config{
    BaseURL:     os.Getenv("BEAMDROP_BASE_URL"),   // Required, must have scheme+host
    AccessKeyID: os.Getenv("BEAMDROP_ACCESS_KEY_ID"),
    SecretKey:   os.Getenv("BEAMDROP_SECRET_KEY"),
    // HTTPClient: &http.Client{Timeout: 5*time.Minute},  // Optional custom client
    // UserAgent:  "my-app/1.0",                           // Optional, default "beamdrop-go-client/0.1"
})
if err != nil {
    log.Fatal(err)  // ErrInvalidBaseURL or ErrMissingCredentials
}
```

### Bucket Operations

```go
// List all buckets → *BucketList{Buckets []BucketInfo, Count int}
buckets, err := c.ListBuckets(ctx)

// Create bucket — errors with 409 BUCKET_EXISTS if already exists
created, err := c.CreateBucket(ctx, "my-bucket")   // → *BucketCreated{Bucket, Created, Location}

// Create bucket (IDEMPOTENT — preferred for automation)
// Returns 201 if new, 200 with Exists=true if already existed
created, err := c.CreateBucketIfNotExists(ctx, "my-bucket")

// Check existence via HEAD (returns false on 404, not an error)
exists, err := c.BucketExists(ctx, "my-bucket")   // → bool

// Delete (must be empty — 409 BUCKET_NOT_EMPTY if objects remain)
err = c.DeleteBucket(ctx, "my-bucket")
```

Bucket name rules: 3-63 chars, regex `^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$`, no IP-like names.

### Object Operations

```go
// Upload from bytes → *ObjectCreated{Bucket, Key, ETag, Size, URL}
uploaded, err := c.PutObject(ctx, "bucket", "path/to/file.txt", []byte("content"))

// Upload from io.Reader (streaming — use for large files, avoids loading into memory)
f, _ := os.Open("large-file.bin")
defer f.Close()
uploaded, err = c.PutObjectReader(ctx, "bucket", "large-file.bin", f)

// Download → *ObjectBody{ObjectMetadata, Body []byte}
obj, err := c.GetObject(ctx, "bucket", "path/to/file.txt")
fmt.Println(string(obj.Body))                    // file content
fmt.Println(obj.ContentType)                     // "text/plain"
fmt.Println(obj.ETag)                            // MD5 hex hash (quotes trimmed)

// Metadata only (no body download) → *ObjectMetadata
meta, err := c.HeadObject(ctx, "bucket", "path/to/file.txt")
fmt.Printf("Size: %d, Type: %s\n", meta.ContentLength, meta.ContentType)

// Check existence (false on 404, not an error)
exists, err := c.ObjectExists(ctx, "bucket", "key")

// Delete
err = c.DeleteObject(ctx, "bucket", "path/to/file.txt")

// List objects with S3-style prefix/delimiter → *ObjectList
list, err := c.ListObjects(ctx, "bucket", client.ListObjectsOptions{
    Prefix:    "folder/",    // only keys starting with "folder/"
    Delimiter: "/",          // group into "directories" at "/"
    MaxKeys:   100,          // default 1000
})
// list.Contents → []ObjectInfo{Key, Size, LastModified, ETag, ContentType}
// list.CommonPrefixes → []CommonPrefix{Prefix} (virtual subdirectories)
// list.IsTruncated → bool (more results available)
```

Object key rules: max 1024 bytes, no `..`, no leading `/`. Forward slashes create virtual directory hierarchies.
Max upload size: 5GB. Writes are atomic (crash-safe). ETag = MD5 hex hash of content.

## TypeScript / JavaScript SDK (`beamdrop` npm package)

When generating TypeScript or JavaScript code, always use the official npm package:

```bash
# npm / pnpm / yarn
npm install beamdrop

# Bun
bun add beamdrop
```

Works in Node.js (≥18), Bun, Deno, and any runtime with Web Crypto + Fetch APIs.

### Initialization

```ts
import { Beamdrop, BeamdropException } from "beamdrop";

const client = new Beamdrop({
  baseUrl: process.env.BEAMDROP_BASE_URL!, // e.g. "http://localhost:7777"
  accessKey: process.env.BEAMDROP_ACCESS_KEY_ID!, // "BDK_xxxx"
  secretKey: process.env.BEAMDROP_SECRET_KEY!, // "sk_xxxx"
  // connectTimeout: 10_000,  // Optional, reserved for future use (default 10s)
  // timeout: 120_000,        // Optional, total request timeout (default 2min)
});
```

All methods are `async` and throw `BeamdropException` on failure.

### Bucket Operations

```ts
// List all buckets → ListBucketsResponse { buckets: BucketInfo[], count: number }
const { buckets, count } = await client.listBuckets();

// Create bucket — throws 409 if already exists
const created = await client.createBucket("my-bucket");
// → CreateBucketResponse { bucket, created, location }

// Create bucket (IDEMPOTENT — preferred for automation)
const result = await client.createBucketIfNotExists("my-bucket");
if ("exists" in result) {
  console.log("Bucket already existed");
} else {
  console.log("Bucket created at", result.created);
}

// Check existence via HEAD (returns false on 404, not an error)
const exists = await client.bucketExists("my-bucket"); // → boolean

// Delete (must be empty — throws 409 if objects remain)
await client.deleteBucket("my-bucket");
```

### Object Operations

```ts
// Upload — body accepts string, Blob, ArrayBuffer, ReadableStream, etc.
const uploaded = await client.putObject(
  "bucket",
  "path/to/file.txt",
  "content",
);
// → PutObjectResponse { bucket, key, etag, size, url }

// Upload binary (e.g. from file in Node.js/Bun)
import { readFileSync } from "fs";
const buffer = readFileSync("photo.jpg");
await client.putObject("bucket", "photos/photo.jpg", buffer);

// Download → GetObjectResponse { body (UTF-8 string), content_type, content_length, etag, last_modified }
const obj = await client.getObject("bucket", "path/to/file.txt");
console.log(obj.body); // file content as string
console.log(obj.content_type); // "text/plain"
console.log(obj.etag); // MD5 hex hash
// NOTE: body is returned as UTF-8 string — for binary files, use presigned URLs

// Metadata only (no body) → ObjectMetadata { content_type, content_length, etag, last_modified }
const meta = await client.headObject("bucket", "path/to/file.txt");

// Check existence (false on 404, not an error)
const exists = await client.objectExists("bucket", "key"); // → boolean

// Delete
await client.deleteObject("bucket", "path/to/file.txt");

// List objects with S3-style prefix/delimiter
const list = await client.listObjects("bucket", "folder/", "/", 100);
// list.contents  → ObjectInfo[] { key, size, lastModified, etag }
// list.commonPrefixes → string[] (virtual subdirectories)
// list.isTruncated → boolean
// NOTE: contents and commonPrefixes may be null — use ?? [] for safety
for (const obj of list.contents ?? []) {
  console.log(obj.key, obj.size);
}
```

### Presigned URLs (Client-side HMAC)

Generated locally — no server call needed:

```ts
// Generate a download URL valid for 1 hour (3600 seconds)
const url = await client.presignedUrl("bucket", "file.txt", 3600);
// → "http://server/api/v1/buckets/bucket/file.txt?token=...&expires=...&access_key=..."

// With custom method
const putUrl = await client.presignedUrl("bucket", "file.txt", 3600, "PUT");
```

### Pretty Presigned URLs (Server-side)

Registered on the server, short `/dl/{token}` URLs with tracking:

```ts
// Create — expires in 7 days, max 100 downloads
const link = await client.createPrettyPresignedUrl(
  "bucket",
  "report.pdf",
  7 * 86400, // expiresIn (seconds), null for no expiry
  100, // maxDownloads, null for unlimited
);
// → CreatePrettyPresignedUrlResponse { token, url, bucket, key, method, expiresAt, maxDownloads, createdAt }
console.log(link.url); // "https://server/dl/a1b2c3d4..."

// List all active presigned URLs
const { urls, count } = await client.listPrettyPresignedUrls();

// Revoke — download link immediately stops working
await client.revokePrettyPresignedUrl(link.token);
```

### Error Handling

```ts
import { Beamdrop, BeamdropException } from "beamdrop";

try {
  await client.getObject("bucket", "missing.txt");
} catch (err) {
  if (err instanceof BeamdropException) {
    console.error(`HTTP ${err.status}: ${err.message}`);
    // err.status — HTTP status code (0 for network/timeout errors)
    // err.body   — parsed JSON error body from server (optional)
    // err.body?.error?.code — machine-readable error code

    switch (err.status) {
      case 404:
        console.log("Not found");
        break;
      case 409:
        console.log("Conflict");
        break;
      case 429:
        console.log("Rate limited — retry later");
        break;
    }
  }
}
```

### TypeScript Types Reference

All types are exported from the package:

```ts
import type {
  BucketInfo, // { name, createdAt }
  ListBucketsResponse, // { buckets, count }
  CreateBucketResponse, // { bucket, created, location }
  CreateBucketIfNotExistsResponse, // CreateBucketResponse | { bucket, exists, location }
  PutObjectResponse, // { bucket, key, etag, size, url }
  ObjectMetadata, // { content_type, content_length, etag, last_modified }
  GetObjectResponse, // ObjectMetadata & { body }
  ObjectInfo, // { key, size, lastModified, etag }
  ListObjectsResponse, // { bucket, prefix, delimiter, maxKeys, isTruncated, contents, commonPrefixes }
  PrettyPresignedUrlInfo, // { token, url, bucket, key, method, expiresAt, maxDownloads, createdAt }
  CreatePrettyPresignedUrlResponse, // same as PrettyPresignedUrlInfo
  ListPrettyPresignedUrlsResponse, // { urls, count }
} from "beamdrop";
```

### Framework Integration Examples

#### Express.js

```ts
import express from "express";
import multer from "multer";
import { Beamdrop, BeamdropException } from "beamdrop";

const client = new Beamdrop({ baseUrl, accessKey, secretKey });
const upload = multer({ storage: multer.memoryStorage() });
const app = express();

// Upload endpoint
app.post("/upload/:key", upload.single("file"), async (req, res) => {
  const result = await client.putObject(
    "my-bucket",
    req.params.key,
    req.file!.buffer,
  );
  res.status(201).json(result);
});

// Error middleware
app.use((err, _req, res, _next) => {
  if (err instanceof BeamdropException) {
    res.status(err.status || 502).json({ error: err.message });
    return;
  }
  res.status(500).json({ error: "internal server error" });
});
```

#### Elysia (Bun)

```ts
import { Elysia, t } from "elysia";
import { Beamdrop, BeamdropException } from "beamdrop";

const client = new Beamdrop({ baseUrl, accessKey, secretKey });

const app = new Elysia()
  .onError(({ error, set }) => {
    if (error instanceof BeamdropException) {
      set.status = error.status || 502;
      return { error: error.message };
    }
  })
  .post(
    "/upload/:key",
    async ({ params, body, set }) => {
      const content = await (body.file as File).arrayBuffer();
      const result = await client.putObject("my-bucket", params.key, content);
      set.status = 201;
      return result;
    },
    { body: t.Object({ file: t.File() }) },
  )
  .listen(3000);
```

### Presigned URLs — Choosing the Right Type

Beamdrop has two presigned URL mechanisms. **Always choose deliberately** — they have different trade-offs:

#### Client-side HMAC presigned URLs

Generated locally using your secret key. No server API call needed. Self-contained URL.

```go
url, err := c.PresignObjectURL("GET", "bucket", "file.txt", time.Now().Add(24*time.Hour))
// Returns: "http://server/api/v1/buckets/bucket/file.txt?access_key=BDK_xxx&expires=...&token=..."
```

**Use when:**

- You need zero server overhead (URL computed locally)
- Generating many links in a batch
- Embedding in automated emails or notifications
- Temporary access with no tracking needed

**Limitations:**

- Cannot be revoked (valid until expiry)
- No download counting or limits
- Key rotation invalidates ALL outstanding client-side URLs
- URL is long (contains bucket/key path + query params)

#### Server-side presigned URLs (RECOMMENDED for most use cases)

Created via API call, stored in database with a short 32-char hex token. Clean `/dl/{token}` URLs.

```go
presigned, err := c.CreatePresignedURL(ctx, client.CreatePresignedURLRequest{
    Bucket:       "bucket",
    Key:          "file.txt",
    Method:       "GET",
    ExpiresIn:    int64Ptr(3600),    // 1 hour (seconds). Omit for no expiry.
    MaxDownloads: intPtr(10),         // Max 10 downloads. Omit for unlimited.
})
// presigned.URL = "https://server/dl/a1b2c3d4..."  (short, clean)
// presigned.Token = "a1b2c3d4..."
// presigned.DownloadCount = 0

// List all presigned URLs
urls, err := c.ListPresignedURLs(ctx)   // → *PresignedURLList{URLs, Count}

// Check download stats
details, err := c.GetPresignedURL(ctx, "token")
fmt.Printf("Downloaded %d/%d times\n", details.DownloadCount, *details.MaxDownloads)

// Revoke — /dl/token immediately returns 404
err = c.DeletePresignedURL(ctx, "token")
```

**Use when:**

- You need to revoke access after sharing
- You want download counting or limits
- You need an audit trail (createdBy, createdAt tracked)
- Link must survive API key rotation
- You want short, clean URLs for users
- Sharing sensitive files

**Decision matrix:**

| Scenario                          | Client-side | Server-side |
| --------------------------------- | :---------: | :---------: |
| Quick temporary link, no tracking |     ✅      |             |
| Need to revoke after sharing      |             |     ✅      |
| Limit number of downloads         |             |     ✅      |
| Track download count              |             |     ✅      |
| Batch-generate 1000 links         |     ✅      |             |
| Share in download portal          |             |     ✅      |
| Must survive key rotation         |             |     ✅      |
| Sensitive files with audit trail  |             |     ✅      |
| Embed in automated emails         |     ✅      |             |

## HTTP API Quick Reference

When generating code in other languages, use the HTTP API directly with HMAC signing:

### Buckets

- `GET /api/v1/buckets` → `{"buckets":[{name, createdAt}], "count":N}`
- `PUT /api/v1/buckets/{name}` → 201 `{bucket, created, location}` | 409 BUCKET_EXISTS
- `PUT /api/v1/buckets/{name}?createIfNotExists=true` → 201 (new) or 200 `{exists:true}` (existed)
- `HEAD /api/v1/buckets/{name}` → 200 or 404
- `DELETE /api/v1/buckets/{name}` → 204 | 409 BUCKET_NOT_EMPTY

### Objects

- `PUT /api/v1/buckets/{bucket}/{key}` (body = raw bytes) → 200 `{bucket, key, etag, size, url}`
- `GET /api/v1/buckets/{bucket}/{key}` → raw file (headers: Content-Type, Content-Length, ETag, Last-Modified). Supports Range header
- `HEAD /api/v1/buckets/{bucket}/{key}` → headers only, no body
- `DELETE /api/v1/buckets/{bucket}/{key}` → 204
- `GET /api/v1/buckets/{bucket}?list=true&prefix=X&delimiter=/&max-keys=N` → `{contents, commonPrefixes, isTruncated}`

### Presigned URLs

- `POST /api/v1/presign` (JSON: `{bucket, key, method, expires_in, max_downloads}`) → 201 `{token, url, ...}`
- `GET /api/v1/presign` → `{urls, count}`
- `GET /api/v1/presign/{token}` → presigned URL details with current downloadCount
- `DELETE /api/v1/presign/{token}` → 200 (immediate revocation)
- `GET /dl/{token}` → public download (no auth needed). 404 if expired/revoked/max-reached

### API Keys

- `POST /api/v1/keys` (JSON: `{name, permissions, bucket_scope}`) → 201 `{accessKeyId, secretKey, ...}` — secret shown ONCE
- `GET /api/v1/keys` → `{keys, count}` — no secrets
- `DELETE /api/v1/keys?accessKeyId=BDK_xxxx` → 204

## Error Handling

All errors return consistent JSON: `{"error":{"code":"CODE","category":"CATEGORY","message":"...","details":{}}}`

Handle these errors in generated code:

| HTTP | Code                | What to Do                                                                                                       |
| ---- | ------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 400  | INVALID_BUCKET_NAME | Fix bucket name: 3-63 chars, lowercase a-z0-9 + hyphens/dots, start/end with letter/digit                        |
| 400  | INVALID_OBJECT_KEY  | Fix key: no `..`, no leading `/`, max 1024 bytes                                                                 |
| 401  | UNAUTHORIZED        | Check: (1) API key exists and not disabled, (2) timestamp within ±15 min, (3) signing path matches request path  |
| 403  | PERMISSION_DENIED   | API key lacks required permission or bucket scope doesn't match                                                  |
| 404  | BUCKET_NOT_FOUND    | Create bucket first with `CreateBucketIfNotExists`                                                               |
| 404  | OBJECT_NOT_FOUND    | Object doesn't exist — check key spelling, verify bucket                                                         |
| 409  | BUCKET_EXISTS       | Use `?createIfNotExists=true` to avoid this                                                                      |
| 409  | BUCKET_NOT_EMPTY    | Delete all objects before deleting bucket                                                                        |
| 413  | FILE_TOO_LARGE      | File exceeds 5GB limit — split or compress                                                                       |
| 423  | OBJECT_LOCKED       | Another operation holds the lock — **retry after brief delay** (lock timeout is 30s)                             |
| 429  | RATE_LIMIT_EXCEEDED | **Retry after `Retry-After` header seconds**. Response has `X-Retryable: true`. General: 100/min, upload: 10/min |
| 507  | STORAGE_FULL        | Server reached `-max-storage` limit — cannot upload                                                              |

Go client error handling pattern:

```go
result, err := c.GetObject(ctx, "bucket", "key")
if err != nil {
    var apiErr *client.APIError
    if errors.As(err, &apiErr) {
        switch apiErr.Code {
        case "OBJECT_NOT_FOUND":
            // handle missing object
        case "RATE_LIMIT_EXCEEDED":
            time.Sleep(time.Duration(apiErr.RetryAfter) * time.Second)
            // retry
        }
    }
}
```

## Common Workflows

### Store AI-generated artifacts

```go
c.CreateBucketIfNotExists(ctx, "ai-artifacts")

// Use structured key paths: {purpose}/{session}/{filename}
key := fmt.Sprintf("generations/%s/%s", sessionID, "output.json")
c.PutObject(ctx, "ai-artifacts", key, resultBytes)

// Generate a shareable link (server-side, revocable, 24h expiry)
presigned, _ := c.CreatePresignedURL(ctx, client.CreatePresignedURLRequest{
    Bucket:    "ai-artifacts",
    Key:       key,
    Method:    "GET",
    ExpiresIn: int64Ptr(86400),
})
fmt.Println("Download:", presigned.URL)
```

### Upload and share with download limits

```go
c.CreateBucketIfNotExists(ctx, "shared")
c.PutObject(ctx, "shared", "report.pdf", pdfBytes)

// Server-side URL — max 5 downloads, expires in 7 days
presigned, _ := c.CreatePresignedURL(ctx, client.CreatePresignedURLRequest{
    Bucket:       "shared",
    Key:          "report.pdf",
    Method:       "GET",
    ExpiresIn:    int64Ptr(7 * 24 * 3600),
    MaxDownloads: intPtr(5),
})
```

### List and organize files

```go
// List top-level "directories" in a bucket
list, _ := c.ListObjects(ctx, "bucket", client.ListObjectsOptions{
    Delimiter: "/",
})
// list.CommonPrefixes → ["folder1/", "folder2/"]

// List files within a "directory"
list, _ = c.ListObjects(ctx, "bucket", client.ListObjectsOptions{
    Prefix:    "folder1/",
    Delimiter: "/",
})
// list.Contents → [{Key:"folder1/file.txt", ...}]
// list.CommonPrefixes → ["folder1/sub/"]
```

### Deduplicate with ETags

```go
// ETags are MD5 hashes — use for deduplication
meta, _ := c.HeadObject(ctx, "bucket", "file.txt")
if meta.ETag == computeMD5Hex(newContent) {
    fmt.Println("Content unchanged, skipping upload")
} else {
    c.PutObject(ctx, "bucket", "file.txt", newContent)
}
```

### Scoped API keys for limited access

```bash
# Create a read-only key scoped to one bucket
curl -X POST http://localhost:7777/api/v1/keys \
  -H "Content-Type: application/json" \
  -d '{"name":"readonly-reports","permissions":"read","bucket_scope":"reports"}'
```

## Validation Rules Summary

| Rule                      | Constraint                               |
| ------------------------- | ---------------------------------------- |
| Bucket name length        | 3-63 characters                          |
| Bucket name regex         | `^[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]$`     |
| Bucket name chars         | Lowercase a-z, digits 0-9, hyphens, dots |
| Bucket name start/end     | Must be letter or digit                  |
| IP-like bucket names      | Rejected (e.g., `192.168.1.1`)           |
| Object key max length     | 1024 bytes                               |
| Object key no-go patterns | Empty, contains `..`, starts with `/`    |
| Max upload size           | 5GB (5,242,880,000 bytes)                |
| HMAC clock skew tolerance | ±15 minutes                              |
| Object lock timeout       | 30 seconds                               |
| Rate limit — general      | 100 req/min per IP                       |
| Rate limit — upload       | 10 req/min per IP                        |
| Rate limit — auth         | 5 req/min per IP                         |
