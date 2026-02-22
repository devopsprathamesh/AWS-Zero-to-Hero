# AWS IAM (Identity & Access Management) — Zero → Hero

This module makes you **job-ready for production IAM**: least-privilege design, safe credential patterns, cross-account access, and Kubernetes/EKS identity (IRSA). Treat this as your **runbook + lab guide + interview checklist**.

---

## Outcomes (what you should be able to do)
By the end, you can:
- Model any access problem as **Principal + Action + Resource + Condition** and debug it fast.
- Write **least-privilege policies** with conditions (MFA, tags, VPC, IP, TLS, org paths, session tags).
- Build secure patterns: **roles everywhere**, break-glass, CI/CD deploy roles, cross-account vendor access, short-lived STS creds.
- Operate IAM: key rotation, monitoring, Access Analyzer, credential reports, CloudTrail evidence.
- Implement **EKS IRSA / OIDC federation** correctly (trust policy, audience, sub, session policies).

---

## IAM mental model (don’t skip)
### The “authorization equation”
**Access = allow(Identity policies + Resource policies + Session policies) − explicit deny (anywhere)**
…constrained by outer guardrails:
- **SCPs** (Organizations) limit what accounts can do
- **Permissions boundaries** cap what identities can be granted
- **Session policies** cap assumed-role sessions
- **Resource policies** (S3/KMS/SQS/SNS/etc.) can grant cross-account access

### Policy evaluation order (practical)
1. **Explicit Deny** wins (any policy type).
2. If no explicit deny, AWS looks for an **Allow** that matches.
3. If no allow, it’s **Implicit Deny**.
4. Even with allow, **SCP/Boundary/Session policy** may block it.

### Vocabulary
- **Principal**: user/role/federated user/service principal
- **Identity-based policy**: attached to user/group/role
- **Resource-based policy**: attached to the resource (S3 bucket policy, KMS key policy, etc.)
- **Trust policy**: who can assume a role
- **STS**: AssumeRole/AssumeRoleWithWebIdentity, short-lived creds

---

## IAM components you must master
### Identities
- Users (avoid long-term usage; use for humans only if needed)
- Groups (human permission sets)
- Roles (preferred: workloads + automation + cross-account)

### Policy types (know where each applies)
- AWS managed policies (fast, rarely least-privilege)
- Customer managed policies (recommended)
- Inline policies (avoid unless exceptional)
- Permissions boundaries (caps identity permissions)
- SCPs (account-wide guardrails)
- Session policies (caps at assume-role time)

### Conditions (where senior engineers stand out)
Common high-signal conditions:
- `aws:MultiFactorAuthPresent`
- `aws:PrincipalArn`, `aws:PrincipalTag/*`, `aws:RequestTag/*`, `aws:TagKeys` (ABAC)
- `aws:SourceIp`, `aws:VpcSourceIp` (careful)
- `aws:SecureTransport` (TLS)
- `aws:CalledVia`, `aws:ViaAWSService`
- `aws:RequestedRegion`
- `sts:ExternalId`, `sts:RoleSessionName`, `sts:TagSession`
- Service keys: `s3:prefix`, `kms:ViaService`, `kms:EncryptionContext:*`

---

## Production best practices (non-negotiable)
- Prefer **roles + STS** over access keys.
- Require **MFA** for privileged actions and console access.
- Use **separate accounts** (prod/non-prod/security/log-archive/shared-services).
- Enforce **least privilege** with:
  - narrow actions
  - narrow resources (ARNs)
  - conditions
- Centralize audit: **CloudTrail** (org trail), config baseline, access analyzer.
- Separate duties: **break-glass** role with strict controls + alerting.
- Use **Identity Center (SSO)** for humans when possible.
- Rotate/disable unused credentials; monitor “last used”.

---

## EKS-focused IAM (IRSA essentials)
### What IRSA is
Pods assume an IAM Role via **OIDC** using `AssumeRoleWithWebIdentity`. No node role leakage if done right.

### Required trust policy claims
Your role trust should restrict:
- **OIDC provider**
- `aud` = `sts.amazonaws.com`
- `sub` = `system:serviceaccount:<namespace>:<serviceaccount>`

### Common IRSA pitfalls
- Wrong `sub` / namespace / SA name
- Missing OIDC provider in the account/cluster
- Pod using default SA (no annotation)
- Using global STS endpoints / partition mismatch in restricted environments
- Missing permissions on the target service (S3/KMS/etc.)

---

## “AccessDenied” debugging playbook (fast path)
When you see `AccessDenied` / `UnauthorizedOperation`:
1. Confirm **who** you are:
   - `aws sts get-caller-identity`
2. Confirm the **exact action** + **resource ARN** in the error.
3. Check if it’s blocked by **SCP** / **permissions boundary** / **session policy**.
4. Check **conditions**: region, IP, MFA, tags, encryption context.
5. For roles: validate **trust policy** + external ID / OIDC claims.
6. Validate the service’s **resource policy** (S3/KMS are frequent offenders).
7. Use evidence:
   - CloudTrail event (who called what)
   - IAM policy simulator (when applicable)
   - Access Analyzer findings

---

## Hands-on Labs (production-grade)
> Each lab includes: Goal → Build → Validate → Expected failure → Fix → Cleanup.

### Lab 01 — IAM policy evaluation basics (explicit vs implicit deny)
**Goal:** Prove that explicit deny overrides allow.
- Create a role/user with an allow for `s3:ListAllMyBuckets`
- Add an explicit deny for `s3:*`
- **Validate:** `aws s3 ls` fails even though allow exists
- **Fix:** remove/limit deny statement

---

### Lab 02 — Least-privilege S3 access (read-only to one prefix)
**Goal:** Allow `s3:GetObject` only for `s3://my-bucket/app/*`
- Use resource ARN: `arn:aws:s3:::my-bucket/app/*`
- Add `s3:ListBucket` only with condition `s3:prefix = "app/"`
- **Validate:** can list/get under `app/`, cannot access other prefixes
- **Expected failure:** missing `ListBucket` on bucket ARN (not object ARN)
- **Fix:** add `s3:ListBucket` on `arn:aws:s3:::my-bucket` with prefix condition

---

### Lab 03 — MFA-gated admin actions
**Goal:** Require MFA for sensitive actions (IAM, KMS, org changes).
- Add condition: `"Bool": {"aws:MultiFactorAuthPresent": "true"}`
- **Validate:** actions fail without MFA, succeed with MFA session
- **Gotcha:** API access using access keys won’t magically have MFA; you need MFA session creds via STS.

---

### Lab 04 — Cross-account access (CI/CD deploy role)
**Goal:** Account A (CI) assumes role in Account B (prod deploy).
- In Account B: role trust allows Account A principal to `sts:AssumeRole`
- In Account A: permission to assume that role
- **Validate:** `aws sts assume-role --role-arn ...`
- **Expected failure:** trust policy missing principal / external id mismatch
- **Fix:** correct trust + add `sts:ExternalId` requirement for vendors

---

### Lab 05 — Vendor access with External ID
**Goal:** Prevent confused-deputy attacks.
- Trust policy requires `sts:ExternalId`
- **Validate:** assume-role fails without correct external ID
- **Fix:** supply external ID; keep it secret-ish (treat like shared token)

---

### Lab 06 — Permissions boundary (cap what admins can grant)
**Goal:** Even “admin-ish” role can’t exceed boundary.
- Attach broad allow policy
- Attach boundary that denies/omits `iam:*`, `kms:*`, etc.
- **Validate:** attempts to create privileged policies fail
- **Gotcha:** boundaries **don’t grant**, they only cap.

---

### Lab 07 — ABAC with tags (PrincipalTag + ResourceTag)
**Goal:** Team-based access without per-team policies.
- Require `aws:PrincipalTag/Team == resource tag Team`
- **Validate:** user tagged Team=payments can access only payments-tagged resources
- **Expected failure:** missing tag on principal/resource → implicit deny
- **Fix:** ensure consistent tagging + enforce with `aws:TagKeys` rules

---

### Lab 08 — KMS key policy + IAM policy (two-layer model)
**Goal:** Understand why KMS often denies even with IAM allow.
- Add IAM allow for `kms:Encrypt/Decrypt`
- Ensure **key policy** also allows the principal (or grants via AWS account root + IAM)
- **Validate:** encryption succeeds only when both align
- **Expected failure:** “not authorized to perform kms:Decrypt” due to key policy
- **Fix:** update key policy properly (avoid overly-open principals)

---

### Lab 09 — Resource policy cross-account (S3 bucket policy)
**Goal:** Grant read to a role in another account via bucket policy.
- Bucket policy principal = role ARN
- Constrain with conditions (TLS, source VPC endpoint, prefix)
- **Validate:** cross-account reads work only under constraints

---

### Lab 10 — CloudTrail evidence for IAM investigations
**Goal:** Prove *who did what*.
- Enable org/account trail
- Perform actions: assume-role, access denied, policy changes
- **Validate:** locate events in CloudTrail by `eventName`, `userIdentity.arn`
- **Output:** keep a “forensics checklist” in notes

---

### Lab 11 — IAM Access Analyzer (unintended public/cross-account access)
**Goal:** Detect overly-broad resource policies.
- Turn on Access Analyzer
- Create a bucket policy that accidentally allows `Principal:"*"`
- **Validate:** finding appears; remediate by narrowing principal + conditions

---

### Lab 12 — EKS IRSA for a controller (realistic)
**Goal:** Pod assumes role to access AWS API (e.g., S3 or Route53).
- Create IAM role with OIDC trust restricted by `sub`
- Annotate Kubernetes service account with role ARN
- Deploy a pod that calls `aws sts get-caller-identity`
- **Validate:** caller identity is the IRSA role
- **Expected failure:** `InvalidIdentityToken` or access denied due to trust mismatch
- **Fix:** correct OIDC issuer/provider, `aud`, `sub`

---

## Terraform snippets (minimal, production-shaped)

### IAM role trust for IRSA (template)
```hcl
data "aws_iam_openid_connect_provider" "eks" {
  arn = var.oidc_provider_arn
}

resource "aws_iam_role" "irsa" {
  name = "irsa-${var.name}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = data.aws_iam_openid_connect_provider.eks.arn }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "${replace(var.oidc_issuer, "https://", "")}:aud" = "sts.amazonaws.com"
          "${replace(var.oidc_issuer, "https://", "")}:sub" = "system:serviceaccount:${var.namespace}:${var.serviceaccount}"
        }
      }
    }]
  })
}
```

## 50 senior-level IAM scenarios (prompts you should be able to answer)

1. “I have AdministratorAccess but still AccessDenied” → boundary/SCP/session policy/resource policy
2. S3 access works in console but fails in CLI → different principal/session, missing MFA session creds
3. Cross-account S3 read fails despite bucket policy allow → KMS encryption/key policy issue
4. KMS decrypt denied though IAM allows → key policy missing allow
5. AssumeRole denied though trust exists → caller lacks sts:AssumeRole permission
6. Vendor can assume role without external id → confused deputy risk; fix trust condition
7. Role chaining breaks after N hops → session duration/limits; redesign
8. You want per-team access without per-team policies → ABAC tags
9. Prevent deleting CloudTrail/Config → SCP explicit deny
10. CI/CD can deploy but not create IAM roles → boundary or explicit deny on iam:*
11. EKS pod uses node role instead of IRSA → service account annotation missing
12. InvalidIdentityToken in IRSA → wrong OIDC provider/issuer/audience
13. Restrict S3 to VPC endpoint only → bucket policy aws:SourceVpce
14. Force TLS to S3 → aws:SecureTransport
15. Restrict actions to one region → aws:RequestedRegion
16. Allow RDS snapshot copy cross-account → KMS + snapshot sharing
17. Deny privilege escalation paths → block iam:PassRole + sensitive iam actions
18. iam:PassRole best practice for ECS/EKS/Glue → restrict to exact role ARNs + conditions
19. Rotate access keys without downtime → dual keys, staged rotation, enforce last-used checks
20. Detect unused creds → credential report + last accessed
21. Someone attached AWS managed admin policy by mistake → SCP guardrail + detection
22. Implement break-glass admin safely → separate account, MFA, alerts, time-bound access
23. Prevent public S3 buckets org-wide → SCP + S3 Block Public Access + analyzer
24. Grant EKS controller access to Route53 only → IRSA + narrow actions/resources
25. Debug “works locally, fails in pipeline” → pipeline role session policy/permissions boundary
26. Allow Lambda to assume role → service principal trust lambda.amazonaws.com + passrole chain
27. Use session tags for tenant isolation → sts:TagSession + condition on aws:PrincipalTag
28. Secure third-party GitHub OIDC to assume role → trust conditions on repo/branch claims
29. “User in group but no access” → explicit deny somewhere; policy simulator
30. “Role has allow but API still denied” → resource policy denies or missing
31. Deny kms:DisableKey everywhere except security → SCP + exceptions
32. Delegate IAM admin to platform team but prevent org-level changes → boundary + SCP
33. Use Access Analyzer to validate external sharing → findings workflow
34. Avoid wildcard resources in IAM → when required and how to constrain with conditions
35. S3 ListBucket permission confusion → bucket ARN vs object ARN
36. STS session duration mismatch → MaxSessionDuration and IdP settings
37. AssumeRole fails with MFA requirement → enforce MFA in trust via aws:MultiFactorAuthPresent
38. Prevent root usage → no root keys; MFA; alerts; SCPs won’t restrict root in same way
39. KMS encryption context conditions → lock decrypt to specific context
40. Troubleshoot EC2 instance profile access → instance role vs credentials chain
41. IAM policy size limits workaround → managed policies, split statements
42. “Deny all except X” patterns safely → explicit denies with careful exceptions
43. SQS/SNS cross-account publish/consume → resource policy + kms
44. ECR pull across accounts → repository policy + role permissions
45. Restrict sts:AssumeRole by aws:PrincipalArn/org path → condition keys
46. Use organizations aws:PrincipalOrgID in resource policies → allow only within org
47. Prevent accidental iam:CreateAccessKey for users → deny except break-glass
48. Hardening console access → MFA, password policy, session duration, IP conditions (with caution)
49. Audit policy changes → CloudTrail + alert on Put*Policy, Attach*Policy
50. Detect privilege escalation risks → review PassRole + IAM write actions; analyzer patterns