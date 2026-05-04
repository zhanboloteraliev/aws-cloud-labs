# 🔧 IAM Troubleshooting

These are real-world IAM scenarios I worked through — each one based on the kind of support tickets cloud engineers deal with every day. I set up the broken state myself, diagnosed it using AWS tools, and documented what I found including things that surprised me or didn't work the way I expected.

| Scenario | Ticket |
|----------|--------|
| [T-01](#t-01--policy-not-working-as-expected) | "I gave my user S3 access but it still says denied" |
| [T-02](#t-02--conflicting-policies-allow-vs-deny) | "My user is in two groups with different permissions, which one wins?" |
| [T-03](#t-03--wrong-resource-arn-bucket-vs-object) | "I gave the user `s3:GetObject`, but downloading from S3 still says AccessDenied" |

---

## T-01 — Policy Not Working as Expected

### The Ticket
> *"I attached an S3 full access policy to my user but they are still getting AccessDenied when trying to upload files. I checked and the policy is definitely attached."*

This was the first real troubleshooting scenario I worked through and it immediately taught me something important — the customer is not wrong when they say the policy is attached. Something else is blocking it. My job is to find what.

---

### Why This Happens — The 5 Reasons I Check First

Before touching anything I run through this checklist mentally:

| # | Reason | How Common |
|---|--------|------------|
| 1 | Wrong resource ARN in the policy | Very common |
| 2 | An explicit Deny somewhere overriding the Allow | Common |
| 3 | Policy attached to wrong user or wrong group | Common |
| 4 | Action name in policy doesn't match what the app calls | Common |
| 5 | S3 bucket policy is blocking the request separately | Often missed |

---

### How I Set Up the Broken State

I attached `AmazonS3FullAccess` directly to dev-alice, then created a hidden Deny policy and attached it to her group so she would have conflicting permissions without obviously knowing why.

Hidden deny policy I created:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "BlockS3Uploads",
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*"
    }
  ]
}
```

Named it `Hidden-Deny-Policy` and attached it to the Developers group. Alice now has `AmazonS3FullAccess` on her user but a silent Deny on her group. This is exactly what happens in real environments — old policies from months ago quietly affect new users.

---

### How I Diagnosed It

Used the **IAM Policy Simulator** — this tool shows you the final verdict on any action for any user and tells you exactly which policy statement caused the result.

Steps:
1. IAM → Policy Simulator → selected dev-alice
2. Service: S3 → Action: PutObject → Run simulation
3. Result: ❌ explicitly denied
4. Clicked the result → expanded it → showed `Hidden-Deny-Policy` from Developers group as the cause

This is the tool I reach for first on any "policy not working" ticket. Guessing wastes time.

---

### Unexpected Finding

While testing I noticed alice couldn't delete S3 buckets either — even after fixing the upload issue. Traced it back to `ReadOnly-NoDeletion-Policy` that I had created in Lab 01 and left attached to the Developers group. That policy had an explicit Deny on `s3:DeleteBucket` which was still active.

This taught me something real — in production environments old policies accumulate and silently affect users. A support engineer has to look at the full picture of where permissions are coming from, not just the policy the customer mentions.

Ran the simulator on `s3:DeleteBucket` → confirmed `ReadOnly-NoDeletion-Policy` was the cause → removed it from the group → retested → bucket deletion worked.

---

### The Fix

Removed `Hidden-Deny-Policy` from the Developers group. Confirmed fix by running the simulator again — PutObject now shows ✅ allowed.

---

### Diagnosis Mental Model I Use Now

```
Is there an explicit Deny anywhere?
└── Yes → find it, decide if it should be removed or scoped down
└── No → is the Allow on the right resource ARN?
    └── No → fix the ARN
    └── Yes → is the action name exactly right?
        └── No → fix the action
        └── Yes → is there a bucket policy blocking it?
            └── Check the S3 bucket policy separately
```

Explicit Deny is always first because it overrides everything else.

---

## T-02 — Conflicting Policies (Allow vs Deny)

### The Ticket
> *"My user is in two groups. One group gives S3 access, the other restricts it. I don't know which one wins and my user can't do anything now."*

This scenario is where most people get confused because the instinct is to think the more permissive policy wins or the more specific one wins. Neither is true. Working through this made the evaluation order click for me permanently.

---

### The Rule I Memorized Before Starting

```
1. Explicit Deny anywhere → DENIED, nothing overrides it
2. SCP at organization level blocks it → DENIED
3. Resource-based policy allows it → might be allowed
4. Identity-based policy allows it → ALLOWED
5. Nothing matches → implicit DENY
```

Deny always wins. Always.

---

### How I Set Up the Broken State

Created two groups with conflicting permissions:
- `DataTeam` group → `AmazonS3FullAccess` → allows everything in S3
- `Developers` group → `S3-Total-Deny-Policy` (custom) → denies everything in S3

Added alice to both groups. She now has an Allow from one group and a Deny from another.

The deny policy I created:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAllS3",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

Logged in as alice → went to S3 → AccessDenied even for listing buckets.

---

### How I Diagnosed It

IAM Policy Simulator → dev-alice → S3 → ListBuckets → Run simulation.

Result: ❌ explicitly denied → expanded the result → showed `S3-Total-Deny-Policy` from the Developers group as the cause.

Then I tested every S3 action to understand the scope:

| Action | Result | Reason |
|--------|--------|--------|
| `s3:ListBuckets` | ❌ Denied | Explicit Deny covers `s3:*` |
| `s3:PutObject` | ❌ Denied | Same |
| `s3:GetObject` | ❌ Denied | Same |
| `ec2:DescribeInstances` | ✅ Allowed | Deny is scoped to S3 only |

This confirmed the Deny was broad (`s3:*`) and affected all S3 actions but nothing outside S3.

---

### The Three Ways I Fixed It

I deliberately tried all three fixes so I could understand the difference. In a real ticket the right fix depends on what the customer actually needs.

**Option A — Remove alice from the conflicting group:**
Removed alice from Developers → S3 works. Use this when the user should not be in that group.

**Option B — Remove the bad policy from the group:**
Detached `S3-Total-Deny-Policy` from Developers → S3 works for everyone in the group. Use this when the policy was wrong to begin with.

**Option C — Narrow the Deny to a specific resource:**
Edited the policy so it only denies access to one confidential bucket instead of all S3:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenySpecificBucketOnly",
      "Effect": "Deny",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::confidential-bucket",
        "arn:aws:s3:::confidential-bucket/*"
      ]
    }
  ]
}
```

Use this when the Deny intent was right but the scope was too broad.

---

### What This Taught Me

Adding more Allows does nothing when a Deny exists. I tested this — gave alice five different S3 Allow policies while the Deny was still there. Still denied every time. The only way to fix a Deny is to remove it or narrow its scope. This is the thing that confuses customers most and now I understand exactly why.

---

### Key Interview Answer I Prepared From This

**Q: User is in two groups — one allows S3, one denies S3. What happens?**

Deny wins. AWS evaluates all policies that apply to a user together. If any statement anywhere says Deny for that action, the request is denied — it does not matter how many Allows exist elsewhere. The only exception is if an AWS Organizations SCP is involved, which adds another layer above IAM policies entirely.

---

## T-03 — Wrong Resource ARN: Bucket vs Object

### The Ticket
> *"I gave the user `s3:GetObject`, but they still get AccessDenied when trying to download a file from S3."*

This lab helped me understand one of the most common IAM mistakes: the policy action can be correct, but the `Resource` ARN can still point to the wrong level of S3.

---

### What I Was Trying to Understand

Before this lab I knew that IAM policies have an `Action` and a `Resource`, but I did not fully understand why this fails:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket"
}
```

The mistake is that `s3:GetObject` is an object-level action. It does not apply to the bucket container itself. It applies to files inside the bucket.

The mental model that made it click:

```text
arn:aws:s3:::my-bucket      = the bucket itself
arn:aws:s3:::my-bucket/*    = all objects/files inside the bucket
```

So this is the difference:

| Action | Correct Resource Level |
|--------|------------------------|
| `s3:ListBucket` | `arn:aws:s3:::my-bucket` |
| `s3:GetObject` | `arn:aws:s3:::my-bucket/*` |
| `s3:PutObject` | `arn:aws:s3:::my-bucket/*` |
| `s3:DeleteObject` | `arn:aws:s3:::my-bucket/*` |

---

### How I Set Up the Lab

Created an S3 bucket and uploaded a small test file:

```text
hello.txt
```

Then created a test IAM user for the AWS CLI lab:

```text
iam-arn-lab-user
```

Configured a separate CLI profile so I would not affect my normal AWS credentials:

```bash
aws configure --profile arn-lab
```

At first I accidentally entered this as the default region:

```text
global
```

Then `sts get-caller-identity` failed with this error:

```bash
aws sts get-caller-identity --profile arn-lab

aws: [ERROR]: Could not connect to the endpoint URL: "https://sts.global.amazonaws.com/"
```

The fix was to use a real AWS region code instead of `global`:

```bash
aws configure set region us-west-2 --profile arn-lab
aws sts get-caller-identity --profile arn-lab
```

Lesson: IAM may feel global in the console, but the AWS CLI profile still needs a real region such as `us-west-2` or `us-east-1`.

---

### How I Created the Broken State

I attached this wrong inline policy to the test IAM user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WrongBucketArnForGetObject",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
    }
  ]
}
```

Then I tried to download the file:

```bash
aws s3 cp s3://YOUR-BUCKET-NAME/hello.txt ./downloaded.txt --profile arn-lab
```

Result:

```text
AccessDenied
```

At first this looks confusing because the policy clearly says `Allow` for `s3:GetObject`. The problem is not the action. The problem is the resource ARN.

---

### How I Diagnosed It

The request was trying to access this object:

```text
arn:aws:s3:::YOUR-BUCKET-NAME/hello.txt
```

But the policy only allowed this bucket-level ARN:

```text
arn:aws:s3:::YOUR-BUCKET-NAME
```

Those are not the same resource.

The policy was basically saying:

```text
You can GetObject on the bucket itself.
```

But the actual request was:

```text
Can I GetObject on hello.txt inside the bucket?
```

Since the object ARN did not match the policy resource, AWS denied the request.

---

### The Fix

Changed the policy from the bucket ARN:

```json
"Resource": "arn:aws:s3:::YOUR-BUCKET-NAME"
```

To the object ARN pattern:

```json
"Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
```

Final working policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowGetObjectFromBucketObjects",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```

Retested:

```bash
aws s3 cp s3://YOUR-BUCKET-NAME/hello.txt ./downloaded.txt --profile arn-lab
cat downloaded.txt
```

Expected result:

```text
hello iam lab
```

---

### What This Taught Me

This lab taught me that IAM troubleshooting is not only about asking, "Does the user have the action allowed?" I also need to ask, "Is the action allowed on the exact resource AWS is checking?"

For S3 specifically, I now check bucket-level and object-level ARNs separately:

```text
Bucket-level permission needed?  Use arn:aws:s3:::bucket-name
Object-level permission needed?  Use arn:aws:s3:::bucket-name/*
```

This is why `s3:ListBucket` and `s3:GetObject` often need two different statements in the same policy.

---

### Ticket-Style Root Cause

Root cause: The IAM policy allowed `s3:GetObject`, but the `Resource` used the bucket ARN instead of the object ARN. `GetObject` applies to files inside the bucket, so AWS evaluated the request against `arn:aws:s3:::bucket-name/hello.txt`, which did not match `arn:aws:s3:::bucket-name`. Updating the resource to `arn:aws:s3:::bucket-name/*` fixed the download issue.

---

### Cleanup

After the lab I removed the temporary resources:

- Deleted the access key for `iam-arn-lab-user`
- Deleted the test IAM user
- Deleted `hello.txt`
- Deleted the test S3 bucket

I should never leave lab IAM users or access keys active after testing.
