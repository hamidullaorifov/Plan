# Domain 2 – Security (26%)

What this domain tests:
- Implementing authentication and authorization for applications and AWS services (IAM, Cognito, federation, tokens, RBAC)
- Implementing encryption with AWS services (at rest and in transit, KMS, key rotation, certificates)
- Managing sensitive data in application code (Secrets Manager, Parameter Store, environment variables, data masking)

---

## 1. IAM (Identity and Access Management)

### 1.1 Identities
- **Root user:** do not use day to day. Enable MFA and don't create access keys for it.
- **IAM users:** long-term credentials (password, access keys). Prefer **IAM Identity Center (SSO)** for humans.
- **IAM groups:** collections of users that receive the same permissions. Groups **cannot** be nested and are **not** principals in policies.
- **IAM roles:** assumed identities that issue **temporary credentials** via STS. Used by AWS services (EC2, Lambda, ECS), cross-account access, and federated users.
  - **Trust policy:** **who** can assume the role (the principal).
  - **Permissions policy:** **what** the role can do.
- **Instance profile:** the container that attaches a role to EC2. The SDK on EC2 picks up credentials automatically from IMDS.

### 1.2 Policy types
| Type | Attached to | Notes |
|---|---|---|
| **Identity-based** (AWS managed, customer managed, inline) | User, group, role | Most common |
| **Resource-based** | Resource (S3 bucket policy, SQS queue policy, KMS key policy, Lambda resource policy, SNS topic policy, API Gateway resource policy, ECR repo policy) | Has a `Principal` element. Enables cross-account access **without assuming a role**. |
| **Permissions boundary** | User or role | **Maximum** permissions an identity can have. It doesn't grant anything itself. Used to safely delegate role creation to developers. |
| **SCP (Service Control Policy)** | AWS Organizations OU or account | Maximum permissions for accounts. Doesn't apply to the management account. |
| **Session policy** | Passed during AssumeRole/federation | Further restricts the session |
| **ACLs** | S3 (legacy) | Avoid; disabled by default ("bucket owner enforced") |

### 1.3 Policy structure
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowReadOwnPrefix",
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::my-bucket/${aws:username}/*",
    "Condition": { "Bool": { "aws:SecureTransport": "true" } }
  }]
}
```
- Elements: `Version`, `Statement`, `Sid`, `Effect` (Allow/Deny), `Principal` (resource policies only), `Action`/`NotAction`, `Resource`/`NotResource`, `Condition`.
- **Policy variables:** `${aws:username}`, `${aws:PrincipalTag/team}`, `${cognito-identity.amazonaws.com:sub}`.
- Useful condition keys: `aws:SourceIp`, `aws:SecureTransport`, `aws:RequestedRegion`, `aws:MultiFactorAuthPresent`, `aws:PrincipalOrgID`, `aws:SourceArn`, `aws:SourceAccount`, `s3:x-amz-server-side-encryption`, `dynamodb:LeadingKeys`, `kms:ViaService`, `iam:PassedToService`.
- **ABAC** (attribute-based access control): grants permissions based on tags (`aws:PrincipalTag` matches `aws:ResourceTag`). It scales better than creating a role per project.

### 1.4 Policy evaluation logic (memorize this)
1. By default everything is **implicitly denied**.
2. An **explicit Deny** anywhere **always wins**.
3. Otherwise an **explicit Allow** grants access, provided the SCP, permissions boundary and session policy also allow it.
4. **Cross-account access requires both sides:** an identity policy in account A **and** a resource policy (or role trust) in account B.

### 1.5 `iam:PassRole`
- To give a role to a service (e.g., launch EC2 with a role, create a Lambda function with an execution role), the developer needs **`iam:PassRole`** on that role. This is a very common exam answer for "user can't create a Lambda function or launch EC2 with a role".

### 1.6 Tools
- **IAM Policy Simulator:** test policies.
- **IAM Access Analyzer:** find resources shared externally, validate policies, generate least-privilege policies from CloudTrail activity.
- **Credential report** (account-level) and **Access Advisor** (last-accessed services) support least privilege.
- `aws sts decode-authorization-message` decodes encoded "UnauthorizedOperation" messages.

### 1.7 Best practices (exam favorites)
- **Least privilege.** Use roles instead of long-term keys. Rotate keys. Require MFA. Use IAM Identity Center.
- **Never** embed access keys in code, AMIs, Git, or Lambda env vars.
- Application on EC2 → instance profile role. Lambda → execution role. ECS → **task role**.
- Apps running **outside AWS** (on-premises) → **IAM Roles Anywhere** (X.509 certs) or IAM user access keys stored securely.

---

## 2. AWS STS (Security Token Service)

| API | Use |
|---|---|
| **`AssumeRole`** | Assume a role in the same or another account. Returns temporary credentials (15 min – 12 h; default 1 h). |
| **`AssumeRoleWithWebIdentity`** | Assume a role with a token from an OIDC provider (Google, Facebook, Amazon). **AWS recommends Cognito identity pools instead** for mobile/web apps. Also used by EKS IRSA and GitHub Actions OIDC. |
| **`AssumeRoleWithSAML`** | Enterprise SAML 2.0 federation (AD FS) |
| **`GetSessionToken`** | Temporary credentials for an IAM user, typically **with MFA** (for APIs protected by an `aws:MultiFactorAuthPresent` condition) |
| `GetFederationToken` | Temporary credentials for a federated user (proxy app) |
| `GetCallerIdentity` | Returns account, ARN and user ID |
| `DecodeAuthorizationMessage` | Decodes the encoded error message |

- Cross-account pattern: account B creates a role trusting account A. A user in account A needs `sts:AssumeRole` on that role ARN. They call AssumeRole and use the returned `AccessKeyId`, `SecretAccessKey` and `SessionToken`.
- **External ID** in the trust policy prevents the **confused deputy** problem with third parties.

---

## 3. Amazon Cognito

### 3.1 User pools vs identity pools (most tested Cognito concept)

| | **User pools** | **Identity pools** (Federated Identities) |
|---|---|---|
| Purpose | **Authentication**: user directory, sign-up/sign-in | **Authorization to AWS resources**: exchange identities for **temporary AWS credentials** |
| Output | **JWT tokens**: ID token, access token, refresh token | **AWS credentials** (via STS) mapped to IAM roles |
| Use | Login for web/mobile apps; protect API Gateway (Cognito authorizer) or ALB | Let app users directly access S3, DynamoDB, etc. |
| Identity sources | Native users, social (Google, Facebook, Apple, Amazon), SAML, OIDC | **User pools**, social providers, SAML, OIDC, developer-authenticated identities, **guest (unauthenticated) access** |

- **Typical flow:** the user signs in to the **user pool** and gets a JWT, then exchanges it with the **identity pool** for AWS credentials, then calls S3/DynamoDB directly.
- Identity pools have an **authenticated role** and an **unauthenticated (guest) role**, plus rules-based or token-based role mapping.
- Fine-grained access: IAM policy with `${cognito-identity.amazonaws.com:sub}` (e.g., S3 prefix per user, or DynamoDB `LeadingKeys`).

### 3.2 User pool features
- **Hosted UI / managed login** with OAuth 2.0 flows (authorization code grant, implicit, client credentials). Custom domain needs an **ACM certificate in us-east-1**.
- **MFA** (SMS, TOTP, email), password policies, account recovery, email/phone verification.
- **Advanced security / threat protection:** adaptive authentication (blocks or requires MFA for risky sign-ins) and compromised credential detection.
- **Lambda triggers:** Pre sign-up (auto-confirm/validate), Post confirmation (welcome email, write to DB), Pre authentication, Post authentication, **Pre token generation** (add or modify claims), Custom message, User migration (migrate users from an old directory on first login), Define/Create/Verify auth challenge (custom auth flows, e.g., passwordless).
- **Groups** in user pools map to IAM roles (`cognito:groups` claim) for RBAC.
- Integrations: **API Gateway Cognito authorizer**, **ALB authentication** (ALB can authenticate with Cognito or OIDC before forwarding).
- Tokens: **ID token** (user identity claims; use for authentication), **access token** (scopes; use for authorization against APIs), **refresh token** (gets new tokens; default 30 days). JWTs are verified with the pool's JWKS public keys.

---

## 4. AWS KMS (Key Management Service)

### 4.1 Key types
| Type | Who manages | Examples | Rotation |
|---|---|---|---|
| **AWS owned keys** | AWS (not visible in your account) | SSE-S3 default, DynamoDB default | AWS-managed |
| **AWS managed keys** | AWS in your account (`aws/s3`, `aws/ebs`, `aws/lambda`) | Free; can't change the key policy | Automatic every year |
| **Customer managed keys (CMK)** | You (key policy, grants, rotation, enable/disable, deletion) | $1/month + API calls | **Optional** automatic rotation (yearly by default; configurable 90–2,560 days) or on-demand |
| **Imported key material** | You | Bring your own key | Manual only |

- **Symmetric** (AES-256, default; you never see the key; used by AWS services) vs **asymmetric** (RSA/ECC; download the public key to encrypt or verify outside AWS; sign/verify).
- **HMAC keys:** generate and verify MACs.
- KMS keys are **Regional**. To move encrypted data (e.g., an EBS snapshot) across Regions, **re-encrypt with a key in the destination Region**. **Multi-Region keys** share the same key material across Regions.
- Deleting a key has a mandatory waiting period of **7–30 days** (default 30). Prefer **disable** instead.

### 4.2 Key policies
- **Every KMS key must have a key policy.** Without a statement allowing the account root (`arn:aws:iam::<acct>:root`), IAM policies alone **cannot** grant access.
- The **default key policy** gives the account root full access, which **enables IAM policies** to control access.
- **Grants:** temporary, programmatic delegation (used by services like EBS). `CreateGrant`, `RetireGrant`.
- Cross-account use: the key policy allows the other account, **and** the other account's IAM policy allows the user.
- `kms:ViaService` condition: allow use of the key only through a specific service (e.g., S3).

### 4.3 Envelope encryption (very high yield)
- The **KMS `Encrypt` API is limited to 4 KB** of data. For anything larger, use **envelope encryption**:
  1. Call **`GenerateDataKey`**, which returns a **plaintext data key** and an **encrypted (ciphertext) data key**.
  2. Encrypt your data **locally** with the plaintext data key, then **discard the plaintext key from memory**.
  3. Store the **encrypted data key alongside** the encrypted data (the envelope).
  4. To decrypt: call **`Decrypt`** on the encrypted data key to get the plaintext data key, then decrypt the data locally.
- **`GenerateDataKeyWithoutPlaintext`** returns only the encrypted data key, for use later (e.g., a system that encrypts at a later time).
- The **AWS Encryption SDK** implements envelope encryption for you, with **data key caching** to reduce KMS calls and cost or throttling.
- Other APIs: `Encrypt`, `Decrypt`, `ReEncrypt` (change key without exposing plaintext), `GenerateRandom`, `Sign`, `Verify`, `CreateKey`, `EnableKeyRotation`, `DescribeKey`.
- **Encryption context:** additional authenticated data (key/value pairs) that must match on decrypt. It appears in CloudTrail for auditing.

### 4.4 KMS quotas and throttling
- Cryptographic operations share a per-Region request quota (5,500, 10,000, 50,000 or more requests/second depending on Region and key type). Exceeding it returns **`ThrottlingException`**.
- Solutions:
  - **Exponential backoff** and retries
  - **Data key caching** (Encryption SDK)
  - **S3 Bucket Keys** for SSE-KMS (reduce KMS calls by up to 99%)
  - Request a quota increase (Service Quotas)
- SSE-KMS on S3 counts against KMS quotas: uploads call `GenerateDataKey` and downloads call `Decrypt`. The user needs **`kms:GenerateDataKey`** (upload) and **`kms:Decrypt`** (download and multipart upload).

### 4.5 CloudHSM (to recognize)
- A dedicated, single-tenant **hardware security module** (FIPS 140-2/140-3 Level 3) where **you manage the keys**; AWS has no access. Used for compliance and SSL offloading. It can back KMS as a custom key store.

---

## 5. Encryption in S3

| Method | Key management | Header / notes |
|---|---|---|
| **SSE-S3** | S3-managed AES-256 keys | `x-amz-server-side-encryption: AES256`. **Default for all new objects since Jan 2023.** |
| **SSE-KMS** | KMS key (AWS managed `aws/s3` or customer managed) | `x-amz-server-side-encryption: aws:kms` (+ `x-amz-server-side-encryption-aws-kms-key-id`). Audit in CloudTrail. Subject to KMS quotas; use **Bucket Keys**. |
| **DSSE-KMS** | Dual-layer encryption with KMS | `aws:kms:dsse`. For compliance requiring two layers. |
| **SSE-C** | **You provide the key on every request**; S3 doesn't store it | **HTTPS required.** Headers `x-amz-server-side-encryption-customer-algorithm/-key/-key-MD5`. Not supported in the console. |
| **Client-side encryption** | You encrypt before upload (e.g., Amazon S3 Encryption Client, AWS Encryption SDK) | S3 only stores ciphertext |

- **Enforce encryption:** a bucket policy that **denies `PutObject`** when the `s3:x-amz-server-side-encryption` header is not `aws:kms` (or missing), or set default bucket encryption. Bucket policies are evaluated before default encryption.
- **Enforce encryption in transit:** a bucket policy that denies when `aws:SecureTransport` is `false`.

---

## 6. Encryption in transit and certificates
- **TLS/HTTPS** everywhere. AWS service endpoints support TLS.
- **AWS Certificate Manager (ACM):** free public certificates for **ELB, CloudFront, API Gateway** with auto renewal. You **cannot** export ACM public certificates to install on EC2 (exportable public certificates exist in newer ACM, but the classic exam answer is: use ACM with ALB/CloudFront). **CloudFront requires certificates in us-east-1.**
- ACM Private CA for internal certificates.
- **ALB SSL termination:** HTTPS listener with an ACM certificate; can re-encrypt to targets. **NLB** supports TLS termination or TCP pass-through (end-to-end encryption to EC2).
- SNI (Server Name Indication) lets an ALB or CloudFront serve multiple certificates on one listener.

---

## 7. Managing secrets and sensitive configuration

### 7.1 AWS Secrets Manager vs SSM Parameter Store

| | **Secrets Manager** | **SSM Parameter Store** |
|---|---|---|
| Purpose | Secrets (DB passwords, API keys) | Configuration and secrets |
| **Automatic rotation** | **Yes, built in** (Lambda rotation function; native for RDS, Aurora, Redshift, DocumentDB) | **No native rotation** (build your own with EventBridge + Lambda) |
| Encryption | Always encrypted with KMS | Optional (`SecureString` with KMS); `String`/`StringList` are plaintext |
| Cost | ~$0.40/secret/month + API calls | **Standard tier free** (up to 10,000 params, 4 KB); Advanced tier: 8 KB, parameter policies (expiration notifications), charged |
| Size | Up to 64 KB | 4 KB standard / 8 KB advanced |
| Cross-account | Yes (resource policy) | Advanced tier via sharing |
| Extra | Multi-Region replication of secrets; random password generation | **Hierarchies** (`/myapp/prod/db-url`), `GetParametersByPath`, versioning, public parameters (e.g., latest AMI IDs), CloudFormation dynamic references |

**Rule of thumb for the exam:**
- "**Rotate** database credentials automatically" → **Secrets Manager**.
- "Store configuration values hierarchically, **cheapest**" → **Parameter Store**.
- Parameter Store can **reference** Secrets Manager secrets: `/aws/reference/secretsmanager/<secret-name>`.

### 7.2 Using secrets in services
- **Lambda:** fetch at init (outside the handler) and cache, or use the **AWS Parameters and Secrets Lambda Extension**. Lambda env vars are encrypted at rest; for sensitive values use a customer managed KMS key and encryption helpers, or better, Secrets Manager.
- **ECS:** task definition `secrets` with `valueFrom` (a Secrets Manager or Parameter Store ARN), injected as env vars. The **task execution role** needs permission to read them.
- **CloudFormation dynamic references:** `{{resolve:ssm:/path/param:1}}`, `{{resolve:ssm-secure:/path:1}}`, `{{resolve:secretsmanager:MySecret:SecretString:password}}`. Use `NoEcho: true` for sensitive parameters.
- **CodeBuild:** `env: parameter-store:` and `secrets-manager:` sections in `buildspec.yml`.
- **Elastic Beanstalk:** environment properties (can reference secrets).
- **RDS:** "Manage master credentials in Secrets Manager" option.

### 7.3 Sensitive data handling
- Never log secrets or PII. Use **Amazon Macie** to discover PII in S3.
- **CloudWatch Logs data protection** policies mask sensitive data (e.g., credit card numbers) in logs.
- Data masking and sanitizing in application code; tokenization.
- **Git secrets scanning** (git-secrets) and **Amazon CodeGuru Reviewer / Security** detect hard-coded secrets.

---

## 8. Other security services to recognize
- **AWS WAF:** layer-7 protection (SQL injection, XSS, rate-based rules, geo blocking) for CloudFront, ALB, API Gateway, AppSync, Cognito.
- **AWS Shield:** DDoS protection (Standard is free; Advanced is paid with a response team).
- **Amazon GuardDuty:** threat detection from CloudTrail, VPC Flow Logs, and DNS logs.
- **Amazon Inspector:** vulnerability scanning for EC2, ECR images, and Lambda.
- **AWS Security Hub:** aggregated security posture.
- **AWS Config:** resource configuration history and compliance rules.
- **VPC security:** security groups (stateful, allow-only) vs NACLs (stateless, allow and deny). **VPC endpoints:** gateway endpoints (S3, DynamoDB; free) and interface endpoints (PrivateLink) for private access to AWS services without the internet.

---

## 9. Security exam traps
- "Application on EC2 needs to access S3": **IAM role / instance profile**, **not** access keys in code or config.
- "Mobile app users need to upload to their own S3 folder": **Cognito identity pool** + IAM policy with `${cognito-identity.amazonaws.com:sub}`.
- "Encrypt data larger than 4 KB with KMS": **envelope encryption with `GenerateDataKey`**.
- "KMS ThrottlingException": **data key caching / backoff / quota increase / S3 Bucket Keys**.
- "Rotate RDS password automatically": **Secrets Manager**.
- "Developer gets AccessDenied creating a Lambda function with a role": missing **`iam:PassRole`**.
- "Decode an EC2 authorization error message": **`sts decode-authorization-message`**.
- "Check permissions without launching": **`--dry-run`**.
- "Sign in with Google and access DynamoDB": **Cognito identity pools** (web identity federation).
- "Need user directory + sign-up/sign-in + JWT": **Cognito user pools**.
- "Third party needs access to your account": **cross-account role with external ID**.
- "Restrict DynamoDB items per user": **`dynamodb:LeadingKeys` condition**.
- "S3 SSE-C": **HTTPS is mandatory**; the key isn't stored by AWS.
- "Audit who used a KMS key": **CloudTrail**.
