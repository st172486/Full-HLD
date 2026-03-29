# Presigned URLs - Detailed Notes

## 1. What are Presigned URLs?

A **presigned URL** is a temporary, secure URL that allows access to a private resource without requiring authentication.

- Used with cloud storage systems like AWS S3, GCP, Azure
- Contains embedded permissions + expiration time
- Anyone with the URL can access the resource until it expires

---

## 2. Key Idea

Instead of sharing credentials, you share a **temporary access link**.

### Analogy:
- Private file = Locked room
- Presigned URL = Temporary access pass

---

## 3. Why Do We Need Presigned URLs?

### Problem (Without Presigned URLs):
Client → Backend → Storage

- Backend handles heavy file traffic
- Increased latency
- Higher infrastructure cost
- Poor scalability

### Solution (With Presigned URLs):
Client → Storage (Direct Upload)

- Backend only generates URL
- No heavy file handling
- Scalable and efficient

---

## 4. Use Cases

### 4.1 Secure File Upload
- Upload images/videos directly to cloud
- Backend generates upload URL
- Client uploads directly

### 4.2 Secure File Download
- Temporary access to private files
- Used in reports, resumes, paid content

### 4.3 Temporary File Sharing
- Share secure links via email or apps
- Links expire automatically

### 4.4 Mobile Applications
- Faster uploads
- Reduced backend dependency

---

## 5. Flow of Presigned URLs

### Step 1: Client requests URL
POST /generate-upload-url

### Step 2: Backend generates URL
- Includes:
  - Bucket name
  - File name
  - Expiry
  - Permissions

### Step 3: Backend sends URL
{
  "url": "signed-url"
}

### Step 4: Client uploads file
PUT signed-url

---

## 6. Example (Node.js)

```javascript
import AWS from "aws-sdk";

const s3 = new AWS.S3();

async function generatePresignedUrl() {
  const params = {
    Bucket: "my-bucket",
    Key: "uploads/file.jpg",
    Expires: 60,
    ContentType: "image/jpeg"
  };

  const url = await s3.getSignedUrlPromise("putObject", params);
  return url;
}
```

---

## 7. Key Features

- Time-limited access
- Secure (signed with credentials)
- No credential exposure
- Direct client-to-storage communication

---

## 8. Advantages

- Reduces backend load
- Improves scalability
- Faster uploads/downloads
- Cost-efficient

---

## 9. Important Considerations

- Keep expiry short (1–5 mins)
- Validate file type and size
- Use HTTPS only
- URL can be reused until expiry

---

## 10. Real-World Example (WhatsApp-like System)

1. User selects image
2. App requests presigned URL from backend
3. Backend returns URL
4. App uploads image directly to storage
5. Backend stores metadata (URL, sender, timestamp)

---

## 11. Interview Points

- Used for scalable file handling
- Avoids backend bottlenecks
- Improves performance
- Ensures security with temporary access

---

## 12. Summary

Presigned URLs allow:
- Secure
- Temporary
- Direct access

They are essential for building scalable systems involving file uploads/downloads.
