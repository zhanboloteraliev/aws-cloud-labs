# Lab 01 — IAM Users, Groups & Custom Policies

## Objective
Set up a production-style IAM structure for a fictional company with two teams (Developers and Admins), using least-privilege access control and a custom policy with explicit Deny rules.

---

## Architecture

```
AWS Account (root — MFA enabled)
├── IAM Group: Developers
│   ├── Policy: AmazonEC2ReadOnlyAccess (AWS managed)
│   ├── Policy: AmazonS3FullAccess (AWS managed)
│   ├── Policy: ReadOnly-NoDeletion-Policy (custom — see below)
│   └── User: dev-alice
└── IAM Group: Admins
    ├── Policy: AdministratorAccess (AWS managed)
    └── User: admin-bob
```

---

## What I Built

### 1. Root Account Security
- Enabled MFA on the root account using an authenticator app (Google Authenticator)
- Root account is never used for daily tasks — all work is done through IAM users

### 2. IAM Groups
| Group | Policies Attached |
|-------|------------------|
| Developers | AmazonEC2ReadOnlyAccess, AmazonS3FullAccess, ReadOnly-NoDeletion-Policy (custom) |
| Admins | AdministratorAccess |

### 3. IAM Users
| User | Group | Console Access |
|------|-------|----------------|
| dev-alice | Developers | Yes |
| admin-bob | Admins | Yes |

### 4. Custom Least-Privilege Policy
Created a custom policy named `ReadOnly-NoDeletion-Policy` that explicitly denies destructive actions, even if other policies would allow them.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowEC2ReadOnly",
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:List*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AllowS3BucketListOnly",
      "Effect": "Allow",
      "Action": [
        "s3:ListAllMyBuckets",
        "s3:GetBucketLocation"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyDeleteActions",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "s3:DeleteBucket"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## Testing & Validation

Logged in as `dev-alice` in an incognito browser window and tested the following:

| Action | Expected Result | Actual Result |
|--------|----------------|---------------|
| View EC2 instances | ✅ Allowed | ✅ Allowed |
| Terminate an EC2 instance | ❌ Denied | ❌ Denied |
| View S3 buckets | ✅ Allowed | ✅ Allowed |
| Delete an S3 bucket | ❌ Denied | ❌ Denied |

---

## What Went Wrong & How I Fixed It

**Problem:** After logging in as `dev-alice`, the EC2 instance created by root was not visible.

**Cause:** EC2 instances are region-specific. The instance was created in `us-east-1 (N. Virginia)` under the root account, but after logging in as `dev-alice`, the console was defaulting to a different region (`us-west-2`).

**Fix:** Switched the region dropdown in the top-right corner of the AWS Console to `us-east-1`. The instance appeared immediately.

**Lesson learned:** This is one of the most common issues in real cloud support tickets. Always verify region context when a resource appears "missing." The resource is not gone — it's just in a different region.

---

## Key Concepts Learned

- **IAM Groups** simplify permission management at scale — attach policies to groups, not individual users
- **Least privilege principle** — grant only the permissions required, nothing more
- **Explicit Deny always wins** — even if another policy grants access, an explicit Deny overrides it
- **EC2 instances are region-scoped** — a resource created in one region is not visible in another
- **Root account should never be used for daily operations** — always use IAM users

---

## AWS Services Used

- AWS IAM (Identity and Access Management)
- Amazon EC2 (for testing access)
- Amazon S3 (for testing access)

---

## AWS Documentation Referenced

- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [Policies and Permissions in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html)
- [IAM JSON Policy Reference](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies.html)

---

## Screenshots

| Screenshot | Description |
|------------|-------------|
| `my_groups.png` | IAM dashboard showing groups and users |
| `custom_policy.png` | Custom policy JSON in the IAM policy editor |
| `instance_deletion_denied.png` | dev-alice denied when trying to terminate EC2 instance |
| `s3_bucket_deletion_denied.png` | dev-alice denied when trying to delete S3 bucket |
