# Module 01 — Project 4: Store and Retrieve Files Using Amazon S3

## Project Overview

| Field | Details |
|---|---|
| **Level** | 100 (Beginner) |
| **Project** | 04 |
| **Service** | Amazon S3 (Simple Storage Service) |
| **Completion** | ✅ Complete |
| **Repository Reference** | [AWS Projects Portfolio](https://github.com/shubhammurti/AWS-Projects-Portfolio) |



## What I Built

A foundational Amazon S3 workflow covering bucket creation, file uploads, access control configuration, and object retrieval — including both public access via bucket policy and secure access via pre-signed URLs.



## Architecture Diagram

```
┌─────────────────────────────────────────────┐
│              Amazon S3                       │
│                                              │
│  ┌─────────────────────────────────────┐    │
│  │           S3 Bucket                  │    │
│  │   (Block Public Access: ON/OFF)      │    │
│  │                                      │    │
│  │  ┌─────────────┐ ┌───────────────┐  │    │
│  │  │  folder-1/  │ │  object.txt   │  │    │
│  │  │  file.jpg   │ │  image.png    │  │    │
│  │  └─────────────┘ └───────────────┘  │    │
│  └─────────────────────────────────────┘    │
│                                              │
│  Access Methods:                             │
│  ① Pre-signed URL (Temporary, Secure)        │
│  ② Bucket Policy (Public Read — Learning)    │
└─────────────────────────────────────────────┘
```



## Key Concepts Learned

### 1. S3 is Object Storage — Not a File System

S3 uses a flat namespace. There are no real folders — what appears as a "folder" in the console is simply a key prefix ending in `/`. For example, an object stored as `images/photo.jpg` has a key of `images/photo.jpg`, not a file inside a directory.

### 2. S3 Storage Classes

Choosing the right storage class is a core cost optimization decision:

| Storage Class | Use Case | Retrieval Speed |
|---|---|---|
| S3 Standard | Frequently accessed data | Instant |
| S3 Standard-IA | Infrequent access | Instant |
| S3 Glacier Instant Retrieval | Archives accessed occasionally | Instant |
| S3 Glacier Flexible Retrieval | Long-term cold archives | Minutes–Hours |

> **SA Insight:** Using S3 Standard for cold archive data can cost 4–5x more than necessary. Always ask: *"How often will this data be accessed?"*

### 3. Access Control — Two-Layer Model

```
Layer 1: Block Public Access (Guardrail)
    ON  → No public access regardless of policies
    OFF → Policies and ACLs can grant public access

Layer 2: Bucket Policy / ACL (Defines who gets what)
    Principal, Action, Resource, Effect
```

Disabling Block Public Access does **not** automatically make objects public — it only allows policies to do so. This distinction catches many beginners off guard.

### 4. Understanding Bucket Policies

Every bucket policy answers 5 questions:

| JSON Key | Question | Example |
|---|---|---|
| `Effect` | Allow or Deny? | `"Allow"` |
| `Principal` | Who? | `"*"` (everyone) |
| `Action` | What action? | `"s3:GetObject"` |
| `Resource` | Which objects? | `"arn:aws:s3:::bucket-name/*"` |
| `Condition` | Under what conditions? | *(optional)* |

**Public Read Bucket Policy (for learning purposes):**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::your-bucket-name/*"
    }
  ]
}
```

Read it as a sentence: *"Allow everyone to GetObject from all objects in my bucket."*


## Steps Completed

### Step 1: Create S3 Bucket
- Chose a globally unique bucket name following convention: `name-purpose-date`
- Selected region consistent with previous projects
- Left Block Public Access **enabled** (secure default)
- Noted SSE-S3 encryption is **on by default** for all new buckets (since 2023)

### Step 2: Upload Files
- Uploaded files using the S3 console
- Selected **S3 Standard** storage class for this project
- Observed the flat key structure of uploaded objects

### Step 3: Attempt Direct Object URL Access
- Copied the Object URL and opened it in browser
- Received **403 Access Denied** — expected and correct behavior
- Understood this is not an error but the secure default working as intended

### Step 4: Access via Pre-signed URL (Secure Method)
- Navigated to object → Object actions → Share with a presigned URL
- Set expiry time → Generated URL
- Successfully opened object in browser via the temporary signed URL

### Step 5: Access via Public Bucket Policy (Learning Exercise)
- Disabled Block Public Access at bucket level
- Applied public read bucket policy
- Verified object accessible via Object URL in browser
- **Restored Block Public Access and removed bucket policy after testing**

### Step 6: Explore "Folder" Structure
- Created a prefix-based "folder" in the console
- Uploaded a file inside it
- Confirmed object key is `folder-name/filename.txt` — proving flat namespace


## Solutions Architect Lens

### Cost
- S3 pricing has three components: **storage** (per GB/month), **requests** (per GET/PUT), and **data transfer out** (egress)
- Uploads to S3 are free; downloads to the internet incur egress charges
- Choosing the wrong storage class is one of the most common unnecessary cost drivers in AWS accounts

### Security
- **Deny by default, grant explicitly** — the golden rule for S3
- Never make a bucket public without a deliberate, documented reason
- Pre-signed URLs are the preferred pattern for granting temporary object access
- Exposed S3 buckets are among the most common causes of cloud data breaches (e.g., Capital One 2019)

### Scalability
- S3 is infinitely scalable — no capacity planning required
- Handles thousands of requests per second per prefix automatically
- No servers to manage, no storage limits to worry about

### Reliability
- S3 Standard provides **99.999999999% (11 nines) durability** by replicating objects across a minimum of 3 Availability Zones
- Enable **Versioning** to protect against accidental overwrites and deletions
- Enable **Cross-Region Replication (CRR)** for disaster recovery across regions


## Common Mistakes (And How to Avoid Them)

| Mistake | Why It Happens | Fix |
|---|---|---|
| Making buckets public carelessly | Trying to get quick browser access | Use pre-signed URLs instead |
| Treating S3 like a file system | Familiar mental model from local storage | Learn the flat key/prefix model |
| Ignoring storage classes | Default to S3 Standard for everything | Match class to access frequency |
| Not enabling versioning | Seems unnecessary at first | Enable it on any bucket with important data |
| Forgetting egress costs | Uploads are free, downloads aren't | Design architectures to keep data within AWS |


## Production Architecture Patterns

In real-world architectures, S3 objects are rarely served directly. Common patterns include:

**Pattern 1: CloudFront + S3 (Static Content Delivery)**
```
User → CloudFront (CDN) → S3 Bucket (Private)
```
Bucket stays private; CloudFront serves content globally with caching. Reduces cost and improves performance.

**Pattern 2: Pre-signed URL (User File Access)**
```
User → Application Server → Generate Pre-signed URL → User Downloads from S3
```
Application controls access logic; S3 bucket remains private throughout.

**Pattern 3: EC2/Lambda IAM Role Access**
```
EC2 Instance (with IAM Role) → S3 Bucket
```
No credentials stored on the instance. IAM role grants access. This is how Project 3 (EC2) and Project 4 (S3) connect architecturally.


## Services & Features Used

- Amazon S3 Bucket
- S3 Storage Classes (Standard)
- Block Public Access settings
- Bucket Policies (IAM policy language)
- Pre-signed URLs
- S3 Object key / prefix structure
- Server-Side Encryption (SSE-S3, default)


## Key Takeaways

1. S3 is object storage with a flat namespace — "folders" are key prefixes
2. Block Public Access is a guardrail, not an access grant
3. Pre-signed URLs are the secure, production-grade way to share private objects
4. Bucket policies are just structured sentences answering: Who, What, Which resource, Allow or Deny
5. You never need to memorize policy JSON — understand what it expresses, and use the AWS Policy Generator or documentation to write it
6. S3 Standard durability is 11 nines — reliability is built in
7. Storage class selection is a cost decision, not just a technical one


## Resources

- [Amazon S3 Documentation](https://docs.aws.amazon.com/s3/)
- [S3 Storage Classes](https://aws.amazon.com/s3/storage-classes/)
- [AWS Policy Generator](https://awspolicygen.s3.amazonaws.com/policygen.html)
- [S3 Pre-signed URLs Guide](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
