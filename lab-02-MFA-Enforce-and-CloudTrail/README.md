# Lab 02 — MFA Enforcement + CloudTrail Audit Logging

## Objective
Enforce MFA for all IAM users and enable organization-wide 
audit logging with CloudTrail.

## What I Built
- MFA enabled on dev-alice
- Custom IAM policy that blocks all actions without MFA
- CloudTrail trail storing logs to S3
- Verified logs by finding my own login event in Event History

## Key Concepts Learned
- MFA adds a second layer that makes stolen passwords useless
- IAM policies can use Conditions to check MFA status
- CloudTrail is the answer to "who did this and when"
- Event history has a delay — not real-time
- ConsoleLogin events show whether MFA was used

## CloudTrail Findings
The ConsoleLogin event confirmed:
- MFAUsed: Yes
- Login timestamp was recorded
- Source IP was captured
- Username matched dev-alice
