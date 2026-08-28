# Blob Storage & AWS S3 – Complete Notes

## 1. What is Blob Storage?

Blob (Binary Large Object) storage is used to store **unstructured data** such as:
- Images
- Videos
- PDFs
- Logs
- Backups

### Key Characteristics
- No schema required
- Highly scalable (TB → PB → EB)
- Cost-effective
- Access via key/URL

---

## 2. Blob Storage Providers

| Provider | Service |
|----------|--------|
| AWS | S3 |
| Azure | Blob Storage |
| GCP | Cloud Storage |

---

## 3. What is AWS S3?

Amazon S3 (Simple Storage Service) is an object storage system where:
- Container = Bucket
- File = Object

### Object Structure
- Key (path)
- Value (file data)
- Metadata
- Version (optional)

---

## 4. Uploading Data to S3

### Option 1: Backend Upload
Frontend → Backend → S3

Pros:
- Better control
- Validation possible

Cons:
- High backend load

---

### Option 2: Pre-Signed URL (Recommended)
Frontend → S3 (via pre-signed URL)

Steps:
1. Frontend requests URL
2. Backend generates URL
3. Frontend uploads directly

Pros:
- Scalable
- Faster
- Reduced backend load

---

## 5. When to Use S3

### Use Cases
- User uploads (profile pics, resumes)
- Media storage (videos/images)
- Logs & backups
- Static assets

### Avoid When
- Frequent updates needed
- Strong transactional requirements

---

## 6. Real-World Examples

### Profile Upload
Store image in S3 and save URL in DB.

### Netflix-like System
Videos in S3 + CDN (CloudFront)

### Recruitment System
Store resumes in S3 and save link in DB.

---

## 7. Important Concepts

### Security
- IAM roles
- Bucket policies

### Versioning
Maintain multiple versions

### Storage Classes
- Standard
- Infrequent Access
- Glacier

### Lifecycle Policies
Automate movement across storage classes

---

## 8. Architecture Pattern

Frontend → Backend → S3 → CDN

---

## 9. Common Mistakes

- Storing files in DB
- Public buckets without control
- Not using pre-signed URLs
- Ignoring lifecycle policies

---

## 10. Summary

- Blob storage stores unstructured data
- S3 is AWS's implementation
- Use pre-signed URLs for scalability
- Store only file URLs in DB
