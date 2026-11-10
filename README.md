# example-role-infra

CloudFormation template for the **Git Sync IAM role** that CloudFormation
assumes to deploy the certificate and site infrastructure stacks in this series.

This repo is its own stack. You create the role once by uploading `infra.yaml`
manually, then connect Git Sync to this repo so every later change to the role
deploys automatically. The role is complete on day one: you never come back to
add permissions as the series introduces new resources.

## Repository Layout

| File | Purpose |
| --- | --- |
| `infra.yaml` | The `CloudFormationGitSyncRole` and its three managed policies. |
| `deployment-file.yaml` | Git Sync deployment configuration. No parameters. |
| `README.md` | This document. |

## Prerequisites

1. The connection stack (`example-connection-infra`) is already deployed and
   exports the connection ARN as `Example-GitHubConnectionArn`. This role
   imports that value to grant `PassConnection`.
2. An administrator identity to create the stack on the first run. Git Sync
   cannot bootstrap the role it needs, so the initial create uses a privileged
   deployer.

## What Gets Deployed

`infra.yaml` defines `CloudFormationGitSyncRole` (`AWS::IAM::Role`).

### Trust Policy

Two service principals may assume the role:

- `cloudformation.sync.codeconnections.amazonaws.com`, the Git Sync service.
- `cloudformation.amazonaws.com`, CloudFormation itself.

Both are fenced with an `aws:SourceAccount` condition bound to this account,
which guards against confused deputy access from another account.

### Permissions

Permissions live in three **customer-managed policies** attached to the role via
`ManagedPolicyArns`, not one inline block. A role's aggregate inline policy size
caps at 10,240 characters (whitespace excluded), and splitting into multiple
inline policies does not help because that limit is the sum of all of them.
Managed policies have a separate budget: up to 6,144 characters each, up to 10
attached by default (raising to 20 with a quota increase). Splitting by concern leaves plenty of room to grow, and another role
in the account could attach the same policy by ARN instead of redefining it. All
three policies live under the IAM path `/gitsync/`.

- **`GitSyncPlatformPolicy`**: CloudFormation change set lifecycle; EventBridge
  (`PutRule`/`PutTargets`/`DeleteRule`/`RemoveTargets`, fenced by
  `events:ManagedBy`, plus `DescribeRule`); and self-management, so Git Sync can
  update the role and push permission edits as new policy versions
  (`iam:CreatePolicyVersion` on `policy/gitsync/*`, `AttachRolePolicy` on the
  role's own ARN).
- **`CertificateStackPolicy`**: ACM (request, describe, delete, tag, list) and
  Route 53 (change and read record sets for DNS validation and the alias
  records).
- **`SiteStackPolicy`**: S3 (bucket lifecycle and policy), CloudFront
  (distribution, origin access control, response headers policy, function),
  CodePipeline, CodeBuild, Lambda (`*EmptyBucket`), the site's IAM service roles
  and `iam:PassRole` (scoped to the `/site/` path, see below), `PassConnection`
  on the shared connection (both the `codestar-connections` and `codeconnections`
  prefixes), and CloudWatch Logs.

### How the role scopes site service roles without a stack name

The site stack never names its service roles explicitly. Instead it creates them
under the IAM **path** `/site/`. This role scopes `iam:CreateRole`, the related
management actions, and `iam:PassRole` to
`arn:aws:iam::<account>:role/site/*`.

That path convention is what lets the policy target the site stack's roles
without knowing the stack's name. There is no `SiteStackName` parameter and no
`your-site-stack-*` placeholder to hand-edit. A custom path (unlike a custom
role name) does not require `CAPABILITY_NAMED_IAM`, so the site stack still
deploys under `CAPABILITY_IAM`.

## Deployment

### First run (manual, creates the role)

1. Open **CloudFormation > Stacks > Create stack > With new resources**.
2. Choose **Upload a template file** and upload `infra.yaml`.
3. Name the stack, acknowledge the IAM capability (this role has an explicit `RoleName`, so the console asks for `CAPABILITY_NAMED_IAM`), and create it.

### After that (Git Sync keeps it updated)

1. Push this repo to GitHub.
2. On the stack, open the **Git sync** tab and **Connect**.
3. Select the CodeConnection, the repo and branch, and
   `deployment-file.yaml` as the deployment file.
4. Select `CloudFormationGitSyncRole` itself as the role Git Sync uses.

From then on, a push that edits `infra.yaml` deploys through Git Sync. This works
because `GitSyncPlatformPolicy` lets the role version its own managed policies
(`iam:CreatePolicyVersion` on `policy/gitsync/*`) and attach them to itself.

> **Caution:** the role now manages the stack that defines it. Anyone who can
> push to this repo can grant the role, and themselves, new permissions. Protect
> `main` and keep push access as tight as IAM access in the console.
