# ☁️ AWS Cloud Labs Portfolio

Hands-on AWS labs and troubleshooting scenarios built to prepare for Cloud Support Engineer roles.
Each lab simulates a real support ticket — including mistakes made and how they were fixed.

**Goal:** Cloud Support Associate / Cloud Support Engineer

- LinkedIn: https://www.linkedin.com/in/zhanbolot-erali-317a28173/
- Email: ezhanbolot@gmail.com
---

### 🔧 [IAM Troubleshooting](iam-troubleshooting/README.md)
 
Real-world IAM support scenarios — break, diagnose, and fix.
 
| Scenario | Description |
|----------|-------------|
| [T-01 — Policy Not Working as Expected](iam-troubleshooting/README.md#t-01--policy-not-working-as-expected) | User has S3 policy attached but still gets AccessDenied — diagnosing why |
| [T-02 — Conflicting Policies Allow vs Deny](iam-troubleshooting/README.md#t-02--conflicting-policies-allow-vs-deny) | User is in two groups with conflicting permissions — understanding evaluation order |
| [T-03 — Wrong Resource ARN Bucket vs Object](iam-troubleshooting/README.md#t-03--wrong-resource-arn-bucket-vs-object) | User has `s3:GetObject` allowed but download still fails because the policy points to the bucket ARN instead of the object ARN |
 
---

### 💻 Compute — EC2 *(coming soon)*
### 🌐 Networking — VPC *(coming soon)*
### 🗄️ Storage — S3 *(coming soon)*
