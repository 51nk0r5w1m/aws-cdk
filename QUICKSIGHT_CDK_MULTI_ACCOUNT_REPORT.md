# CDK Fit for Multi-Tenant QuickSight SDLC (Multi-Account + Bitbucket)

## 1) One-line position
Use **Terraform for shared account foundations** and **CDK for the QuickSight portal service/application layer** where code, assets, identity wiring, queues, and API behavior change frequently.

## 2) Indexed evidence map (specific files + lines)

## 2.1 Current platform is already app-centric and CDK-driven
- `README.md` shows the portal is a QuickSight asset platform and includes export/import, lineage, audit, and schedules (core multi-tenant app concerns):  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/README.md#L3-L15
- Architecture explicitly states frontend + Lambda backend + CDK + Cognito + SQS:  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/README.md#L19-L25

## 2.2 CDK is wired into day-2 dev workflow (build, deploy, diff)
- Root scripts include synth/diff/bootstrap/deploy paths for regular SDLC use:  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/package.json#L31-L42
- Infra package is CDK-native with bootstrap/synth/deploy/destroy commands:  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/infrastructure/cdk/package.json#L7-L15

## 2.3 Multi-account readiness is already in code
- Stack env resolves account/region from deployment context variables:  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/infrastructure/cdk/bin/app.ts#L8-L15
- Resource names and ARNs are account-aware (`this.account`, `this.region`) across S3, SQS, IAM, QuickSight scope, env vars:  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/infrastructure/cdk/lib/quicksight-portal-stack.ts#L60-L213

## 2.4 Why CDK is high leverage here (typed orchestration + assets)
- Tight service composition in one stack (S3, Cognito, SQS, Lambda, API, WAF, CloudFront):  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/infrastructure/cdk/lib/quicksight-portal-stack.ts#L58-L474
- Frontend asset publish + runtime config injection in deploy flow (`BucketDeployment` + `Source.asset` + `Source.data`):  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/infrastructure/cdk/lib/quicksight-portal-stack.ts#L444-L463
- API behavior and auth-caching edge rules are encoded as typed infra logic (important for tenant security and reliability):  
  https://github.com/51nk0r5w1m/quicksight-portal/blob/913f13654e12cd093992aac5509cca57458d6e56/infrastructure/cdk/lib/quicksight-portal-stack.ts#L223-L414

## 2.5 Diff-first guardrails are first-class in CDK CLI usage
- This repo’s CDK README documents deploy/synth/diff as standard lifecycle commands:  
  /tmp/workspace/51nk0r5w1m/aws-cdk/README.md#L49-L52  
  /tmp/workspace/51nk0r5w1m/aws-cdk/README.md#L123-L126

## 3) Solid use case (multi-tenant QuickSight, multiple AWS accounts, Bitbucket)
**Use case:**  
You run one QuickSight portal per environment/account (`dev`, `staging`, `prod`) and optionally per tenant segment. Teams frequently change:
- asset export/import logic,
- auth rules and callback domains,
- API route behavior,
- queue/worker settings,
- frontend runtime config.

**Why CDK fits this exactly:**  
You can change app code + infra composition in one typed codebase, then gate promotion with a deterministic diff step before deployment.

## 4) Recommended Bitbucket pipeline workflow
1. **PR validation**
   - Build app artifacts
   - Run tests/lint
   - Run `npm run cdk:synth`
   - Run `npm run cdk:diff`
   - Post diff summary to PR and block merge on unsafe changes
2. **Promotion by account**
   - Deploy same commit to `dev` account
   - Promote to `staging` account
   - Promote to `prod` account
3. **“What changed / what broke” automation**
   - Save synth/diff artifacts per run
   - Compare diff deltas between last good run and current run
   - Alert when IAM scope, auth callbacks, edge rules, or queue semantics changed unexpectedly

This directly addresses “see which deploy got screwed up”: you get a reproducible, account-scoped change record tied to each commit and pipeline run.

## 5) Why Terraform is recklessly hard for this exact model
Terraform can provision the same services, but for this app-centric SDLC it becomes high-friction because:
- **Typed service composition** in one language is weaker (large HCL graphs + module glue for this many cross-service dependencies).
- **Asset-centric deploy behavior** (frontend artifact + generated runtime config + coupled API/auth changes) usually requires extra scripting layers outside Terraform.
- **PR-grade change visibility for app+infra together** is less natural; teams often depend on ephemeral plan outputs and custom wrappers.
- **Fast iteration on tightly coupled API/auth/edge behavior** is slower when logic is split across HCL + external build/deploy scripts.

In practice: not impossible to provision, but **materially harder to operate well** for this specific QuickSight portal SDLC unless you add substantial custom tooling around Terraform.

## 6) Practical split (best operating model)
- Keep **Terraform** for long-lived shared foundations (org networking, baseline IAM guardrails, shared DNS/certs, account bootstrap).
- Use **CDK** for the QuickSight portal product layer (API/Lambda/SQS/Cognito/edge rules/frontend assets) where release velocity and diff-driven safety matter most.
