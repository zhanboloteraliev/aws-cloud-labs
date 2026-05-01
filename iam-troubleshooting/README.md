# 🔧 IAM Troubleshooting
 
Real-world IAM support scenarios — each one based on actual support tickets.
Every scenario includes the symptom, diagnosis process, root cause, and fix.
 
| Scenario | Ticket |
|----------|--------|
| [T-01](#t-01--policy-not-working-as-expected) | "I gave my user S3 access but it still says denied" |
 
---
 
## T-01 — Policy Not Working as Expected
 
### The Ticket
> *"I attached an S3 full access policy to my user but they are still getting AccessDenied when trying to upload files. I checked and the policy is definitely attached."*
 
This is the most common IAM support ticket. The customer is not lying — the policy IS attached. Something else is blocking it.
 
---
 
### Why This Happens — The 5 Reasons
 
Before touching anything, a support engineer mentally runs through this checklist:
 
| # | Reason | How Common |
|---|--------|------------|
| 1 | Wrong resource ARN in the policy | Very common |
| 2 | An explicit Deny somewhere is overriding the Allow | Common |
| 3 | Policy attached to wrong user or wrong group | Common |
| 4 | Action in policy doesn't match what app is calling | Common |
| 5 | S3 bucket policy is blocking the request | Often missed |
 
You need to rule out all five before you can tell the customer what is wrong.
 
---
 
### Step-by-Step — Reproduce and Diagnose
 
#### Step 1 — Set Up the Broken State
 
**Why:** Never debug from assumptions. Always reproduce the exact broken state yourself so you can confirm when it is fixed.
 
1. Log in as **root**
2. Go to **IAM → Users → dev-alice**
3. Click **Add permissions → Attach policies directly**
4. Search for and attach `AmazonS3FullAccess`
5. Now create a policy that silently blocks uploads. Go to **IAM → Policies → Create policy → JSON**:
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
 
6. Name it `Hidden-Deny-Policy`
7. Attach it to the `Developers` group (alice is in this group)
Now alice has `AmazonS3FullAccess` directly on her user but a hidden Deny on her group. This is the broken state.
 
Screenshot alice's permissions page showing both policies.
 
---
 
#### Step 2 — Confirm the Problem
 
1. Open an **incognito window** → log in as `dev-alice`
2. Go to **S3** → click any bucket → try to upload any file
3. You will get `AccessDenied`
Screenshot the error.
 
---
 
#### Step 3 — Use the IAM Policy Simulator
 
**Why:** This is the tool that answers "what does this user actually have permission to do?" without guessing. Every cloud support engineer uses this daily.
 
1. Log in as **root**
2. Go to: **IAM → Policy Simulator** (or search "IAM Policy Simulator" in Google — it opens at `https://policysim.aws.amazon.com`)
3. Under **Users, Groups, and Roles** → select `dev-alice`
4. Under **Select service** → choose **S3**
5. Under **Select actions** → check `PutObject`
6. Click **Run simulation**
Result shows: ❌ **implicitly denied** or ❌ **explicitly denied**
 
If it shows **explicitly denied** — there is a Deny somewhere. This narrows it down immediately.
 
Screenshot the simulation result.
 
---
 
#### Step 4 — Find the Deny
 
1. Still in Policy Simulator → click on the **denied** result
2. It expands and shows which policy caused the denial
3. You will see `Hidden-Deny-Policy` listed as the reason
In a real support ticket the customer would not know this policy exists — maybe another admin added it, maybe it came from a group they forgot alice was in. Now you know exactly what to fix.
 
Screenshot the expanded denial showing the policy name.
 
---
 
#### Step 5 — Check All Policy Sources
 
**Why:** You need to understand the full picture before telling the customer what to fix. Alice might have policies from multiple places.
 
1. Go to **IAM → Users → dev-alice → Permissions tab**
2. You will see policies from three sources:
   - **Directly attached** to alice — `AmazonS3FullAccess`
   - **Inherited from Developers group** — `Hidden-Deny-Policy`, `AmazonEC2ReadOnlyAccess`, etc
3. Expand each policy and read what it does
This is the manual version of what the Policy Simulator does automatically. Knowing both methods matters — sometimes the simulator is not available, sometimes you need to explain it to a customer step by step.
 
Screenshot the full permissions view showing all sources.
 
---
 
#### Step 6 — Fix It
 
1. Go to **IAM → Policies → Hidden-Deny-Policy**
2. Click **Policy usage** tab → you can see which groups or users it is attached to
3. Go to **IAM → User groups → Developers → Permissions**
4. Find `Hidden-Deny-Policy` → click **Remove**
Now test again as alice — the upload works. ✅
 
Screenshot the successful upload.
 
---
 
#### Step 7 — Verify With Policy Simulator Again
 
**Why:** Always confirm the fix worked using the same tool you used to diagnose it. This is good support practice — you do not close a ticket until you have verified the resolution.
 
1. Go back to **IAM Policy Simulator**
2. Select `dev-alice` → S3 → PutObject → Run simulation
3. Now shows: ✅ **allowed**
Screenshot the green allowed result.
 
---
 
### The Diagnosis Mental Model
 
When a customer says "my policy is not working" always think in this order:
 
```
Is there an explicit Deny anywhere?
└── Yes → find it and decide if it should be removed
└── No → is the Allow on the right resource ARN?
    └── No → fix the ARN
    └── Yes → is the action name exactly right?
        └── No → fix the action
        └── Yes → is there a bucket policy blocking it?
            └── Check the S3 bucket policy separately
```
 
Explicit Deny is always the first thing to rule out because it overrides everything else.
 
---
 
### Safe Terminal Output to Screenshot
 
```
Policy Simulator Result:
Service: Amazon S3
Action: PutObject
Resource: *
Final result: DENIED
 
Matched statement:
  Policy: Hidden-Deny-Policy (attached via Developers group)
  Effect: Deny
  Action: s3:PutObject
 
After fix:
Action: PutObject
Final result: ALLOWED
 
Matched statement:
  Policy: AmazonS3FullAccess (attached directly to user)
  Effect: Allow
  Action: s3:*
```
 
---
 
### Key Concepts for Interviews
 
**Q: A user has AdministratorAccess but still gets denied on one action — how is that possible?**
> An explicit Deny somewhere overrides AdministratorAccess. Could be a second policy, a group policy, an SCP at the organization level, or a resource-based policy on the target resource.
 
**Q: What is the difference between an implicit deny and explicit deny?**
> Implicit deny — the action is simply not mentioned anywhere so AWS defaults to deny. Explicit deny — a policy statement says `"Effect": "Deny"` for that action. Explicit deny is stronger — it cannot be overridden by any Allow.
 
**Q: What tool do you use to test IAM permissions without making changes?**
> IAM Policy Simulator — lets you simulate any action for any user, group or role and shows exactly which policy statement is responsible for the result.
 
---
