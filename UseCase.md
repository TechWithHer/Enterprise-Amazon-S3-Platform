# MindStrength Enterprise S3 Platform — Production Capstone Project

---

## 🎯 **The Real-World Storyline**

**MindStrength** is a Singapore-based mental wellness SaaS startup with 8 employees. You've scaled from a hobby project to serving corporate clients across SE Asia. Your infrastructure was never formally designed—it evolved organically.

**The Crisis:** Last month, you lost 3 days of user session data due to accidental deletion. A developer mistakenly deleted an S3 bucket. Your HR files are accessible to everyone. Your finance invoices have no encryption. Your AWS bill jumped 40% because nobody's managing storage efficiently.

**The Assignment (from your CTO):**
> "Design a production-grade S3 platform for MindStrength. Make it secure, automated, cost-effective, and recoverable. You have 2 weeks. After this, we're on-boarding 5 corporate clients—their data depends on this."

**Your Role:** Jr. Cloud Engineer responsible for this end-to-end delivery.

---

## 📊 **MindStrength's Real Data Landscape**

| Data Type | Volume | Sensitivity | Access Pattern | Owner |
|-----------|--------|-------------|-----------------|-------|
| User Session Videos | 50 GB | High | Developers (read) | Dev Team |
| Client Invoices & Contracts | 5 GB | Critical | Finance team only | HR/Operations |
| HR Records (Salary, Reviews) | 2 GB | Critical | HR only | HR |
| Application Logs | 100 GB/month | Medium | Ops team (read-only) | Operations |
| Client Data Backups | 80 GB | High | Ops (backup/restore) | Operations |
| Marketing Assets | 15 GB | Low | Marketing + Developers | Marketing |
| Deleted User Data (archive) | 20 GB | Medium | Legal hold (2 years) | Operations |

**Current Problem:** Everything's in one bucket with one set of credentials shared among the team. 😱

---

## 🏗️ **Your Project — Phased Approach**

### **Phase 1: Environment Setup (Days 1-2)**

#### **Create Secure Bucket Architecture**

```bash
# Production buckets for MindStrength
mindstrength-sessions         # User video sessions
mindstrength-finance          # Invoices, contracts
mindstrength-hr-records       # Payroll, reviews (most sensitive)
mindstrength-logs             # Application logs
mindstrength-backups          # Database/config backups
mindstrength-marketing        # Marketing assets
mindstrength-legal-archive    # Deleted data with Object Lock
mindstrength-cloudtrail-logs  # Audit trail (separate bucket)
```

#### **Enable Security Baseline on All Buckets**
```yaml
✓ Versioning (protection against accidental deletion)
✓ Block All Public Access (even if misconfigured)
✓ Default Encryption (AES-256)
✓ Disable ACLs (use only policies)
```

**Why This Matters:** When your developer deleted data last month, there was no versioning. This prevents that forever.

---

### **Phase 2: Identity & Access Control (Days 3-4)**

#### **Create IAM Roles for MindStrength Team**

```yaml
Team Structure:
├── HR (1 person)
│   └── Access: mindstrength-hr-records/* (full)
│   └── Access: mindstrength-finance/* (read-only)
│
├── Developers (3 people)
│   └── Access: mindstrength-sessions/* (read/write)
│   └── Access: mindstrength-marketing/assets/* (read)
│   └── Access: mindstrength-logs/* (read-only)
│
├── Operations (2 people)
│   └── Access: ALL buckets (limited to specific actions)
│   └── Access: mindstrength-backups/* (full)
│   └── Access: mindstrength-logs/* (read-only)
│
└── Marketing (2 people)
    └── Access: mindstrength-marketing/* (read/write)
    └── Access: mindstrength-sessions/thumbnails/* (read)
```

#### **IAM Policy Example (Developers)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SessionVideoAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::mindstrength-sessions/*"
    },
    {
      "Sid": "DenyDelete30Days",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::mindstrength-sessions/*",
      "Condition": {
        "NumericLessThan": {
          "aws:CurrentTime": "2024-01-30T00:00:00Z"
        }
      }
    }
  ]
}
```

**Business Logic:** Developers can't delete videos uploaded in the last 30 days (prevents accidental loss).

---

### **Phase 3: Bucket Policies & Security (Days 5-6)**

#### **Master Bucket Policy (All Buckets)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::mindstrength-*/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::mindstrength-*",
        "arn:aws:s3:::mindstrength-*/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyRootAccountExceptSpecificActions",
      "Effect": "Deny",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-ID:root"
      },
      "Action": "s3:*",
      "Resource": "arn:aws:s3:::mindstrength-hr-records/*",
      "Condition": {
        "StringNotEquals": {
          "aws:username": "HR-IAM-Role"
        }
      }
    }
  ]
}
```

**What You're Preventing:**
- Unencrypted data uploads (compliance requirement)
- HTTP traffic (Singapore's fintech regulations)
- HR data access by anyone except HR role

---

### **Phase 4: Event-Driven Automation (Days 7-8)**

#### **Scenario: Auto-Process Client Invoices**

When Finance uploads a new invoice, automatically:
1. Extract metadata
2. Trigger virus scan
3. Send notification to accounting system

```yaml
S3 Event Configuration:

Bucket: mindstrength-finance
Trigger Event: s3:ObjectCreated:*
Prefix: invoices/

Destinations:
├── Lambda Function: InvoiceProcessor
│   └── Extracts: Invoice Number, Amount, Date
│   └── Posts to: Internal Accounting API
│
├── SNS Topic: InvoiceNotification
│   └── Email: finance@mindstrength.com
│   └── Message: "New invoice uploaded: INV-2024-001"
│
└── SQS Queue: VirusScanQueue
    └── Integration: ClamAV Lambda
    └── Action: Quarantine if infected
```

#### **Lambda Function Example**

```python
import json
import boto3
import requests

s3_client = boto3.client('s3')
sns_client = boto3.client('sns')

def lambda_handler(event, context):
    # Extract file details
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download invoice
    response = s3_client.get_object(Bucket=bucket, Key=key)
    invoice_data = response['Body'].read()
    
    # Parse (simplified - use textract in production)
    invoice_num = key.split('_')[1]  # INV_2024_001.pdf
    
    # Send to accounting system
    accounting_api = "https://accounting.mindstrength.com/api/invoices"
    requests.post(accounting_api, json={
        "invoice_number": invoice_num,
        "s3_location": f"s3://{bucket}/{key}",
        "upload_timestamp": event['Records'][0]['eventTime']
    })
    
    # Notify finance team
    sns_client.publish(
        TopicArn='arn:aws:sns:ap-southeast-1:ACCOUNT:InvoiceNotification',
        Subject='New Invoice Received',
        Message=f'Invoice {invoice_num} uploaded successfully'
    )
    
    return {"statusCode": 200, "body": "Invoice processed"}
```

**Real Impact:** Manual invoice entry eliminated. Finance team saves 5 hours/week.

---

### **Phase 5: Lifecycle Management (Days 9-10)**

#### **Cost Optimization for MindStrength**

Current problem: Storing 20 GB of deleted user data in Standard storage (expensive).

```yaml
Bucket: mindstrength-sessions
Rule: Archive Old Sessions

Age 0-30 days:     Standard (hot data, developers access daily)
Age 30-90 days:    Standard-IA (warm data, occasional access)
Age 90-365 days:   Glacier Flexible Retrieval (cold data, legal hold)
Age 365+ days:     Glacier Deep Archive (compliance, rarely accessed)

Estimated Savings:
├── Current: 100 GB × $0.023/GB = $2,300/month
├── Optimized: 
│   ├── 20 GB Standard × $0.023 = $460
│   ├── 30 GB Standard-IA × $0.0125 = $375
│   ├── 40 GB Glacier × $0.004 = $160
│   └── 10 GB Deep Archive × $0.00099 = $10
└── New Total: $1,005/month (56% savings!)
```

```json
{
  "Rules": [
    {
      "ID": "TransitionVideoSessions",
      "Filter": {"Prefix": "sessions/"},
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 2555
      }
    },
    {
      "ID": "DeleteIncompleteUploads",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
```

**Business Outcome:** CFO is happy. Monthly AWS bill drops from $5K to $3.2K.

---

### **Phase 6: Security & Compliance (Days 11-12)**

#### **Enable Object Lock for Legal Archive**

```bash
# Create bucket WITH Object Lock enabled (cannot be added later)
aws s3api create-bucket \
  --bucket mindstrength-legal-archive \
  --region ap-southeast-1 \
  --object-lock-enabled-for-bucket

# Set retention policy on legal data
aws s3api put-object-retention \
  --bucket mindstrength-legal-archive \
  --key deleted_users/user_2024_001.json \
  --retention "Mode=GOVERNANCE,RetainUntilDate=2026-01-01T00:00:00Z"
```

**Compliance Logic:** Deleted user data must be retained for 2 years per Singapore PDPA regulations. Even if AWS account is compromised, nobody can delete this data.

#### **Enable CloudTrail for Audit**

```yaml
CloudTrail Data Events:
├── Monitor: s3:GetObject, s3:PutObject, s3:DeleteObject
├── Buckets: mindstrength-hr-records, mindstrength-finance
├── Log Destination: mindstrength-cloudtrail-logs bucket
└── Alert: CloudWatch alarm if 10+ DeleteObject calls in 5 mins

Sample CloudTrail Log:
{
  "eventTime": "2024-01-15T10:30:45Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteObject",
  "sourceIPAddress": "203.0.113.45",
  "userAgent": "aws-cli/2.0",
  "requestParameters": {
    "bucketName": "mindstrength-hr-records",
    "key": "payroll_2024.xlsx"
  },
  "userIdentity": {
    "principalId": "AIDAI1234567890EXAMPLE",
    "arn": "arn:aws:iam::ACCOUNT:role/HR-Role"
  },
  "responseElements": null
}
```

---

### **Phase 7: Disaster Recovery (Days 13-14)**

#### **Cross-Region Replication**

```yaml
Primary Region: ap-southeast-1 (Singapore)
Backup Region: ap-southeast-2 (Sydney)

Critical Buckets to Replicate:
├── mindstrength-finance (RTO: 1 hour, RPO: 15 mins)
├── mindstrength-hr-records (RTO: 2 hours, RPO: 1 hour)
└── mindstrength-backups (RTO: 4 hours, RPO: 2 hours)

Replication Configuration:
{
  "Role": "arn:aws:iam::ACCOUNT:role/s3-replication",
  "Rules": [
    {
      "ID": "ReplicateFinance",
      "Filter": {"Prefix": ""},
      "Status": "Enabled",
      "Priority": 1,
      "Destination": {
        "Bucket": "arn:aws:s3:::mindstrength-finance-backup",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {"Minutes": 15}
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {"Minutes": 15}
        }
      }
    }
  ]
}
```

#### **Recovery Procedures (Documented)**

**Scenario 1: Accidental Object Deletion**
```bash
# List previous versions
aws s3api list-object-versions \
  --bucket mindstrength-sessions \
  --prefix deleted_session.mp4

# Restore from version ID
aws s3api get-object \
  --bucket mindstrength-sessions \
  --key deleted_session.mp4 \
  --version-id "abc123xyz" \
  recovered_session.mp4

# Time to recover: < 5 minutes
```

**Scenario 2: Ransomware Attack**
```bash
# Failover to Sydney replica
# Update DNS/application config to use Sydney bucket
# Verify data integrity

# Time to failover: < 30 minutes
```

**Scenario 3: Object Lock Protection**
```bash
# Try to delete protected legal document
aws s3api delete-object \
  --bucket mindstrength-legal-archive \
  --key user_data.json

# Result: Access Denied (object locked until 2026-01-01)
```

---

### **Phase 8: Monitoring & Cost Optimization (Days 15)**

#### **CloudWatch Dashboard for MindStrength**

```yaml
Key Metrics to Track:

1. Storage Growth
   ├── mindstrength-sessions: 50 GB
   ├── mindstrength-finance: 5 GB
   ├── mindstrength-hr-records: 2 GB
   └── Total: 100 GB (with breakdown by storage class)

2. API Calls
   ├── PutObject rate: ~500 calls/day (developers uploading videos)
   ├── GetObject rate: ~2,000 calls/day (clients downloading)
   └── DeleteObject rate: 10-20 calls/day (normal)

3. Cost Tracking
   ├── Standard storage: $460/month
   ├── Standard-IA: $375/month
   ├── Glacier: $170/month
   └── Total: $1,005/month (vs $2,300 before optimization)

4. Replication Metrics
   ├── Replication latency: 14.5 mins (SLA: 15 mins) ✓
   ├── Failed operations: 0
   └── Bytes replicated: 2.3 GB/day

5. Security Events
   ├── Unauthorized access attempts: 3 (blocked)
   ├── HTTP requests (should be 0): 0 ✓
   └── Objects deleted in last 7 days: 5 (within normal range)
```

#### **CloudTrail Insights**

```json
{
  "insight_type": "ApiCallRateInsight",
  "bucket": "mindstrength-hr-records",
  "event_name": "GetObject",
  "baseline_rate": 5.2,
  "current_rate": 487.3,
  "timestamp": "2024-01-15T14:32:00Z",
  "alert_severity": "HIGH",
  "message": "HR data accessed 93x above normal (possible data breach)"
}

Action Taken:
├── Automatic SNS alert to operations team
├── Lambda function blocks further access from anomalous IP
└── CloudTrail logs preserved for forensic analysis
```

---

### **Phase 9: Access Points (Bonus - Days 16+)**

#### **Simplified Access Management**

Instead of managing individual IAM policies per person, create simplified Access Points.

```yaml
Access Point: finance-ap
├── Alias: arn:s3:ap-southeast-1:ACCOUNT:accesspoint/finance-ap
├── Bucket: mindstrength-finance
├── Policy: Read/write invoices/, read-only reports/
├── Networks: IP whitelist (office + VPN only)

Access Point: hr-ap
├── Alias: arn:s3:ap-southeast-1:ACCOUNT:accesspoint/hr-ap
├── Bucket: mindstrength-hr-records
├── Policy: Full access (HR only)
├── MFA Required: Yes (extra security for sensitive data)

Access Point: developer-ap
├── Alias: arn:s3:ap-southeast-1:ACCOUNT:accesspoint/developer-ap
├── Bucket: mindstrength-sessions
├── Policy: Read/write sessions/, read marketing assets
├── Rate limit: 1,000 requests/minute
```

---

## 📋 **Final Deliverables Checklist**

```yaml
✓ Phase 1: Created 8 production buckets with versioning & encryption
✓ Phase 2: Defined IAM roles for all 5 team member types
✓ Phase 3: Implemented bucket policies (deny unencrypted, deny HTTP, deny unauthorized)
✓ Phase 4: Automated invoice processing (Lambda + SNS + SQS)
✓ Phase 5: Implemented lifecycle rules (60% cost reduction documented)
✓ Phase 6: Enabled Object Lock for 2-year legal compliance
✓ Phase 7: Configured cross-region replication (Singapore to Sydney)
✓ Phase 8: Set up CloudTrail auditing & CloudWatch monitoring
✓ Phase 9: Created access points for simplified team access
✓ Documentation: Disaster recovery playbook written
✓ Testing: Tested accidental deletion recovery (works in 4 mins)
✓ Security Review: Bucket policies validated, zero public exposure
```

---

## 🎓 **Interview Questions You Can Now Answer**

1. **"How would you recover from accidental deletion?"**
   - Answer: Versioning + CloudTrail. I restored a deleted invoice in 4 minutes.

2. **"Why did you choose that storage class?"**
   - Answer: Application logs are accessed for 30 days, then rarely. Standard-IA after 30 days saves 45% cost.

3. **"How do you secure HR data?"**
   - Answer: Object Lock, IAM role restriction, HTTPS-only, encrypted at rest, CloudTrail auditing.

4. **"What happens if the Singapore region goes down?"**
   - Answer: Cross-region replication to Sydney. RTO < 30 minutes.

5. **"How do you detect if someone's stealing data?"**
   - Answer: CloudTrail anomaly detection. Alerts if GetObject rate is 90x above baseline.

---

## 💡 **Real Business Impact**

| Metric | Before | After | Owner |
|--------|--------|-------|-------|
| Monthly AWS S3 Cost | $2,300 | $1,005 | CFO ✓ |
| Data Loss Risk | High | None | CTO ✓ |
| Compliance (PDPA) | Failing | Passing | Legal ✓ |
| Manual Invoice Entry | 5 hrs/week | 0 hrs | Finance ✓ |
| Disaster Recovery Plan | None | Tested | Ops ✓ |
| Security Incidents | 0 but risky | 0 and protected | Security ✓ |

---

## 🚀 **Your Next Steps**

1. **This week:** Build this project in AWS free tier (you have 12 months, enough for testing)
2. **Document as you go:** Each phase, screenshot your CloudTrail logs and billing
3. **Practice disaster recovery:** Actually delete something and recover it 3 times
4. **Prepare for interviews:** You now have a real project to discuss

**MindStrength's CTO:** "Great work. We're confident in our S3 platform now. Let's onboard those 5 corporate clients."

---

This is production-grade. You're ready.
