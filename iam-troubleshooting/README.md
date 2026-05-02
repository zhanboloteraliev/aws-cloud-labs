# 🔧 IAM Troubleshooting

These are real-world IAM scenarios I worked through — each one based on the kind of support tickets cloud engineers deal with every day. I set up the broken state myself, diagnosed it using AWS tools, and documented what I found including things that surprised me or didn't work the way I expected.

| Scenario | Ticket |
|----------|--------|
| [T-01](#t-01--policy-not-working-as-expected) | "I gave my user S3 access but it still says denied" |
| [T-02](#t-02--conflicting-policies-allow-vs-deny) | "My user is in two groups with different permissions, which one wins?" |

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
