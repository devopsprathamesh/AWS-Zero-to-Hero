# AWS STS (Security Token Service) — Production Guide

> Suggested repo path: `AWS-ZeroToHero/1_Identity_Access/2_AWS_STS/README.md`
STS is the backbone of **short-lived credentials** in AWS: cross-account access, SSO federation, **EKS IRSA**, CI/CD, and OIDC (GitHub Actions).

---

## Table of contents
- [Why STS](#why-sts)
- [STS mental model](#sts-mental-model)
- [Core STS APIs](#core-sts-apis)
- [Session concepts](#session-concepts)
- [Endpoints (private EKS best practice)](#endpoints-private-eks-best-practice)
- [Production patterns](#production-patterns)
- [CLI cheat sheet](#cli-cheat-sheet)
- [Trust policy templates](#trust-policy-templates)
- [Caller permission policy](#caller-permission-policy)
- [Session policies (runtime caps)](#session-policies-runtime-caps)
- [Troubleshooting playbook](#troubleshooting-playbook)
- [Hands-on labs](#hands-on-labs)
- [Terraform snippets](#terraform-snippets)
- [Senior DevOps checklist](#senior-devops-checklist)
- [Interview scenarios](#interview-scenarios)
- [Quick reference](#quick-reference)

---

## Why STS
**STS helps you avoid long-lived access keys** by issuing temporary credentials that expire automatically.

---

## STS mental model
Any AWS authorization decision can be reduced to:

- **Principal** (who)
- **Action** (what)
- **Resource** (which)
- **Condition** (context)

STS is how you **change “who” you are** (by obtaining temporary credentials for a different identity, typically an IAM Role).

### The golden rule (AssumeRole success criteria)
To assume a role successfully you need **both**:
1. **Caller permission**: caller is allowed to call the STS API (e.g., `sts:AssumeRole`) for that role ARN
2. **Role trust policy**: the role allows that caller (and any trust conditions match)

If either fails → `AccessDenied`.

---

## Core STS APIs

### `AssumeRole` (default for cross-account + role switching)
Use when an AWS principal (user/role/service) needs credentials for another **IAM role**.
- Common uses: **CI/CD deploy roles**, **break-glass**, **admin switch roles**, **cross-account access**
- Supports: **MFA**, **ExternalId**, **session tags**, **session policies**, **source identity**

### `AssumeRoleWithWebIdentity` (OIDC)
Use when you have an **OIDC JWT**:
- **EKS IRSA** (pods)
- **GitHub Actions OIDC**
- Cognito / external IdPs

### `AssumeRoleWithSAML` (SAML)
Use for enterprise SSO where your IdP issues SAML assertions.

### `GetSessionToken` (MFA-backed session for IAM users)
Use if you must keep IAM users (legacy), but want MFA session credentials.

### `GetCallerIdentity` (debug)
Use everywhere to confirm **who you are**.

---

## Session concepts

### Temporary credentials are always 3 parts
STS returns:
- `AccessKeyId`
- `SecretAccessKey`
- `SessionToken` ✅ required for STS-issued creds
Plus an `Expiration`.

> If you forget the `SessionToken`, AWS calls will fail (invalid token / auth errors).

### Session duration
- Sessions are **time-bound**
- The effective max is limited by the **role’s MaxSessionDuration** and the calling method (SSO/OIDC/etc.)
- Choose durations intentionally (CI/CD usually ~1h; humans longer if your policy allows)

### Role chaining
Assuming a role **from another assumed role** can create stricter limits and operational complexity.
Rule of thumb: keep chains short; prefer direct assumption patterns where possible.

---

## Endpoints (private EKS best practice)

### Prefer regional STS endpoints
Regional endpoints reduce latency and avoid edge cases.
Recommended:
- CLI/SDK: enable **regional STS endpoints**
- Private subnets: use an **Interface VPC Endpoint (PrivateLink) for STS** to avoid NAT dependency

**EKS gotcha:** misaligned STS endpoint behavior (global vs regional expectations) can cause region-scoping errors. Prefer regional endpoints + STS VPC endpoint for private clusters.

---

## Production patterns

### 1) CI/CD cross-account deploy role (recommended)
**Account A (CI)** assumes **Account B (Prod)** deploy role.

- **Prod (Account B):** role trust allows CI principal (Account A role) to assume it
- **CI (Account A):** CI role is permitted to call `sts:AssumeRole` on prod role ARN

### 2) Vendor / third-party access (must use ExternalId)
Prevent confused deputy by requiring **ExternalId** in the trust policy.

### 3) EKS IRSA (pods → IAM role)
Pods assume role via OIDC using `AssumeRoleWithWebIdentity`.
Trust policy must lock to:
- OIDC provider
- `aud` = `sts.amazonaws.com`
- `sub` = `system:serviceaccount:<namespace>:<serviceaccount>`

### 4) GitHub Actions OIDC (no AWS keys)
Trust GitHub OIDC provider and restrict claims:
- repo
- branch/ref
- workflow
- environment (if used)

---

## CLI cheat sheet

### Who am I?
```bash
aws sts get-caller-identity
```

### Assume role (basic)
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> \
  --role-session-name devops-session
```

### Assume role with MFA
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> \
  --role-session-name mfa-session \
  --serial-number arn:aws:iam::<ACCOUNT_ID>:mfa/<MFA_DEVICE_NAME> \
  --token-code 123456
```

### Assume role with session tags (ABAC / traceability)
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> \
  --role-session-name deploy-prod \
  --tags Key=Team,Value=platform Key=Env,Value=prod \
  --transitive-tag-keys Team Env \
  --source-identity prathamesh
```

### Assume role with web identity (OIDC token file)
```bash
aws sts assume-role-with-web-identity \
  --role-arn arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME> \
  --role-session-name oidc-session \
  --web-identity-token file://token.jwt
```

### Get a session token for IAM user (MFA)
```bash
aws sts get-session-token \
  --serial-number arn:aws:iam::<ACCOUNT_ID>:mfa/<MFA_DEVICE_NAME> \
  --token-code 123456
```

---

## Trust policy templates

### A) Cross-account role trust (CI role from another account)
Replace `<CI_ACCOUNT_ID>` and `<CI_ROLE_NAME>`:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCIRoleAssume",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::<CI_ACCOUNT_ID>:role/<CI_ROLE_NAME>"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

### B) Third-party vendor trust with ExternalId (confused deputy protection)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowVendorWithExternalId",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::<VENDOR_ACCOUNT_ID>:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "YOUR-EXTERNAL-ID"
      }
    }
  }]
}
```

### C) EKS IRSA trust policy (OIDC) — lock to service account
Replace:
- `<OIDC_PROVIDER_ARN>`
- `<OIDC_ISSUER_HOSTPATH>` (issuer without `https://`)
- `<NAMESPACE>`, `<SERVICEACCOUNT>`

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "IRSAWebIdentityAssume",
    "Effect": "Allow",
    "Principal": { "Federated": "<OIDC_PROVIDER_ARN>" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "<OIDC_ISSUER_HOSTPATH>:aud": "sts.amazonaws.com",
        "<OIDC_ISSUER_HOSTPATH>:sub": "system:serviceaccount:<NAMESPACE>:<SERVICEACCOUNT>"
      }
    }
  }]
}
```

### D) GitHub Actions OIDC trust (restrict repo + ref)
Replace:
- `<GITHUB_OIDC_PROVIDER_ARN>`
- `<ORG>/<REPO>` and branch/ref pattern

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "GithubOidcAssume",
    "Effect": "Allow",
    "Principal": { "Federated": "<GITHUB_OIDC_PROVIDER_ARN>" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:<ORG>/<REPO>:ref:refs/heads/main"
      }
    }
  }]
}
```

---

## Caller permission policy
Attach this to the **caller** (CI role/user/etc.):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowAssumeSpecificRole",
    "Effect": "Allow",
    "Action": "sts:AssumeRole",
    "Resource": "arn:aws:iam::<TARGET_ACCOUNT_ID>:role/<TARGET_ROLE_NAME>"
  }]
}
```

**Best practice:** avoid `Resource: "*"`. If you must, constrain with strong conditions (org, principal tags, source identity, role naming conventions, etc.).

---

## Session policies (runtime caps)
You can pass an inline **session policy** during `AssumeRole` to **reduce** permissions for that session.
- Useful for: “deploy-only”, “break-glass with strict cap”, “debug sessions”
- **Caps** permissions; does not grant additional permissions

---

## Troubleshooting playbook

### 1) `AccessDenied` on `AssumeRole`
Checklist:
1. Confirm principal: `aws sts get-caller-identity`
2. Caller policy allows `sts:AssumeRole` on the role ARN?
3. Role trust policy allows your principal?
4. Any trust conditions failing? (MFA, ExternalId, IP, tags)
5. Guardrails blocking? (**SCP**, **permissions boundary**, **session policy**)

### 2) `InvalidIdentityToken` (OIDC / IRSA / GitHub)
Common causes:
- Wrong OIDC provider / issuer mismatch
- Wrong `aud` (must match trust)
- Wrong `sub` (namespace/serviceaccount/repo/ref mismatch)
- Token expired or malformed

### 3) “MissingAuthenticationToken” / “The security token included in the request is invalid”
Common causes:
- Forgot to export `AWS_SESSION_TOKEN`
- Using expired STS creds
- Mixed env vars / profile confusion

### 4) Works locally, fails in CI
Common causes:
- CI runs under a different role than expected
- CI assume-role uses a **session policy** that caps permissions
- SCP / boundary differences between CI and prod accounts

---

## Hands-on labs

### Lab 01 — AssumeRole basics (local)
**Goal:** Assume role and call AWS APIs with temp creds.
- Create role + trust for your user/role
- Allow caller `sts:AssumeRole`
- Run `aws sts assume-role ...`
- Export creds and validate:
  - `aws sts get-caller-identity`

### Lab 02 — ExternalId enforcement (vendor pattern)
**Goal:** Prove assume-role fails without ExternalId.
- Add `sts:ExternalId` condition in trust
- Attempt assume-role without it (fail)
- Attempt with correct ExternalId (success)

### Lab 03 — MFA-required assume role
**Goal:** Enforce MFA at assume time.
- Require MFA in the trust policy (or via operational policy)
- Try assume-role without MFA params (fail)
- Try with MFA device + token (success)

### Lab 04 — Session tags + ABAC
**Goal:** Use session tags to control access.
- Trust allows `sts:TagSession`
- Role policy checks tags (PrincipalTag/session tag based conditions)
- Assume role with `--tags Team=platform`
- Validate access only to matching tagged resources

### Lab 05 — IRSA identity proof (EKS)
**Goal:** Pod uses IRSA role (not node role).
- Create IRSA role trust locked to service account
- Annotate Kubernetes service account with role ARN
- Pod runs: `aws sts get-caller-identity`
- Verify role identity is the IRSA role

### Lab 06 — GitHub OIDC restricted deploy
**Goal:** Only main branch workflow can assume deploy role.
- Configure GitHub OIDC provider + role trust conditions
- Run workflow from `main` (success)
- Run from a feature branch (fail)

---

## Terraform snippets

### IRSA role trust (Terraform template)
```hcl
variable "oidc_provider_arn" {}
variable "oidc_issuer" {} # full URL, e.g. https://oidc.eks.<region>.amazonaws.com/id/XXXX
variable "namespace" {}
variable "serviceaccount" {}

resource "aws_iam_role" "irsa" {
  name = "irsa-${var.namespace}-${var.serviceaccount}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Federated = var.oidc_provider_arn }
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

### Caller policy to assume a specific role (Terraform)
```hcl
resource "aws_iam_policy" "assume_role_target" {
  name = "assume-target-role"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["sts:AssumeRole"]
      Resource = ["arn:aws:iam::<TARGET_ACCOUNT_ID>:role/<TARGET_ROLE_NAME>"]
    }]
  })
}
```

---

## Senior DevOps checklist (STS)
- Explain **caller permission vs role trust policy**
- Implement **ExternalId** for vendors
- Always include/export **SessionToken** for STS creds
- Understand how **SCP / boundary / session policy** can block access
- Debug `AccessDenied` using `get-caller-identity` + CloudTrail evidence
- Build **IRSA trust** correctly (`aud`/`sub`)
- Use **regional STS endpoints** and STS **VPC endpoints** in private EKS

---

## Interview scenarios (practice)
1. Why does `AssumeRole` fail even though the trust policy allows me?
2. S3 read is allowed but decrypt fails—what’s happening?
3. My pod is using the node role, not IRSA—how do you prove/fix?
4. How do you secure third-party access to your account?
5. How do SCPs and permission boundaries interact with STS sessions?

---

## Quick reference
- Prefer **roles + STS** everywhere; avoid long-lived access keys
- Require **MFA** for privileged role assumption
- Enforce **least privilege** + conditions
- Use **ExternalId** for vendors
- For EKS private clusters: **regional STS + STS VPC endpoint**