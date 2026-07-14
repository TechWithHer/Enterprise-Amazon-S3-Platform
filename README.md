```markdown
# MindStrength Enterprise Amazon S3 Platform

---

## Project Overview

**MindStrength** is a Singapore-based mental wellness SaaS startup scaling rapidly across SE Asia. This project delivers a **production-grade Amazon S3 platform** designed to securely store, automate, monitor, optimize, and recover critical business data.

This platform addresses real-world challenges faced by MindStrength, including accidental data loss, compliance with Singapore PDPA, cost overruns, and operational inefficiencies.

---

## Use Case & Business Context

MindStrength manages diverse data types with varying sensitivity and access needs:

| Data Type               | Sensitivity | Access Team       | Volume    | Key Requirements                          |
|------------------------|-------------|-------------------|-----------|------------------------------------------|
| User Session Videos     | High        | Developers        | 50 GB     | Secure, versioned, lifecycle-managed     |
| Client Invoices & Contracts | Critical    | Finance           | 5 GB      | Encrypted, automated processing, audit  |
| HR Records             | Critical    | HR                | 2 GB      | Strict access, object lock, compliance   |
| Application Logs       | Medium      | Operations        | 100 GB/mo | Read-only access, lifecycle archiving    |
| Client Data Backups    | High        | Operations        | 80 GB     | Disaster recovery, replication           |
| Marketing Assets       | Low         | Marketing, Devs   | 15 GB     | Shared access, cost-optimized             |
| Deleted User Data (Archive) | Medium      | Legal/Operations  | 20 GB     | Object lock, long-term retention         |

---

## Project Objectives

- Design and implement multiple secure, versioned S3 buckets tailored to data sensitivity.
- Enforce least-privilege IAM roles and bucket policies.
- Automate workflows using Lambda, SNS, and SQS.
- Optimize storage costs with lifecycle rules and storage classes.
- Enable cross-region replication for disaster recovery.
- Monitor activity with CloudTrail, CloudWatch, and Storage Lens.
- Document recovery procedures and cost optimization strategies.

---

## Architecture Overview

```text
Users (HR, Finance, Developers, Marketing, Operations)
        │
        ▼
IAM Roles & Policies (Least Privilege)
        │
        ▼
Amazon S3 Buckets (Versioning, Encryption, Object Lock)
        │
        ├── Event Notifications → Lambda / SNS / SQS
        │
        ├── Lifecycle Rules (Transition & Expiration)
        │
        ├── Cross-Region Replication (Singapore → Sydney)
        │
        └── Access Points (Department-specific access)
        │
        ▼
Monitoring & Auditing (CloudTrail, CloudWatch, Storage Lens)
```

---

## Bucket List

| Bucket Name                 | Purpose                          | Key Features                          |
|----------------------------|---------------------------------|-------------------------------------|
| mindstrength-sessions       | User session videos             | Versioning, lifecycle, encryption   |
| mindstrength-finance        | Invoices & contracts            | Encryption, event-driven automation |
| mindstrength-hr-records     | Payroll & reviews               | Object Lock, strict access           |
| mindstrength-logs           | Application logs               | Lifecycle archiving                  |
| mindstrength-backups        | Backups                        | Replication, lifecycle               |
| mindstrength-marketing      | Marketing assets               | Shared access, cost optimization     |
| mindstrength-legal-archive  | Deleted user data (legal hold) | Object Lock, long retention          |
| mindstrength-cloudtrail-logs| CloudTrail audit logs          | Secure, separate bucket              |

---

## Key Features & Implementations

### Security

- **Versioning** enabled on all buckets to protect against accidental deletion.
- **Block Public Access** enforced globally.
- **Default Encryption** (AES-256) on all buckets.
- **Object Lock** enabled on legal archive bucket for compliance.
- **IAM Roles** with least privilege and folder-level access.
- **Bucket Policies** enforcing HTTPS-only and denying unencrypted uploads.
- **CloudTrail Data Events** enabled for audit logging.

### Automation

- **S3 Event Notifications** trigger Lambda, SNS, and SQS for invoice processing, virus scanning, and notifications.
- **Lambda Functions** automate metadata extraction and integration with internal systems.

### Cost Optimization

- Lifecycle rules transition data through Standard, Standard-IA, Glacier Flexible Retrieval, and Deep Archive.
- Cleanup of incomplete multipart uploads.
- Intelligent tiering recommendations documented.

### Disaster Recovery

- Cross-region replication from Singapore (ap-southeast-1) to Sydney (ap-southeast-2).
- Documented recovery procedures for accidental deletion, ransomware, and compliance holds.

### Monitoring

- CloudWatch dashboards track storage growth, API calls, replication status, and security events.
- CloudTrail insights detect anomalous access patterns.
- Storage Lens and inventory reports provide detailed usage analytics.

---

## Getting Started

### Prerequisites

- AWS Account with appropriate permissions.
- AWS CLI configured.
- Basic knowledge of IAM, S3, Lambda, SNS, SQS.

### Deployment Steps

1. **Create S3 Buckets** with versioning, encryption, and block public access.
2. **Define IAM Roles** for HR, Finance, Developers, Marketing, and Operations.
3. **Apply Bucket Policies** to enforce security controls.
4. **Set up Folder Structure** and upload sample files.
5. **Configure Event Notifications** for automation workflows.
6. **Implement Lifecycle Rules** for cost optimization.
7. **Enable Object Lock** on legal archive bucket.
8. **Set up Cross-Region Replication** for disaster recovery.
9. **Create Access Points** for simplified access management.
10. **Enable Monitoring and Logging** with CloudTrail and CloudWatch.
11. **Test Disaster Recovery** scenarios.
12. **Document Cost Optimization** and operational procedures.

---

## Disaster Recovery Playbook (Summary)

- **Accidental Deletion:** Use versioning to restore previous object versions within minutes.
- **Ransomware Attack:** Failover to replicated bucket in Sydney region.
- **Legal Compliance:** Object Lock prevents deletion of protected data.
- **Audit:** CloudTrail logs all access and changes for forensic analysis.

---

## Cost Optimization Summary

- Transition infrequently accessed data to cheaper storage classes.
- Delete incomplete multipart uploads after 7 days.
- Use lifecycle policies to automate data archiving.
- Monitor storage usage and costs regularly via CloudWatch and Storage Lens.

---

## Contact & Support

For questions or support, contact:

- **Cloud Engineering Team**: cloudeng@mindstrength.com
- **Operations**: ops@mindstrength.com
- **Finance**: finance@mindstrength.com

---

## License

This project is for internal use at MindStrength and is not publicly licensed.

---

## Acknowledgments

This project is inspired by AWS best practices and the Amazon S3 Production Capstone course.

---

*Prepared by: Jr. Cloud Engineer, MindStrength*  
*Date: July 2026*  
*Location: Singapore*  
```
