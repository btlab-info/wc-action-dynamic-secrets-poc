# Dynamic secrets in the WC GitHub Action: proof it works

**Short version:** dynamic AWS credentials over GitHub OIDC already work with the Workload Credentials API today. The Action could support them with one extra API call and no per-integration code.

## What I proved
A GitHub workflow with **no stored secrets** got 15-minute AWS credentials from Workload Credentials, deployed to S3, and was blocked from anything outside its role.

- Workflow: [`.github/workflows/dynamic-aws.yml`](.github/workflows/dynamic-aws.yml), about 15 lines of shell
- To run it: register a Pathfinder Workload Identity for your repo, then fill in the `env:` values at the top of the workflow.
- Ran live on 2026-09-29 in a private demo repo: role session `assumed-role/<role>/bt-<name>-…`, HTTP 201 from WC, creds issued with a 15-minute expiry

## How it works
1. **Log in:** GitHub gives the job an OIDC token. The Action already does this for static secrets.
2. **Generate:** `POST /site/<site>/wlc/dynamic/<name>/generate?folder=<folder>`, with the same headers the Action already sends. WC answers `201` with the creds under `.secret`, the same wrapper as static secrets:
   ```
   secret.accessKeyId  secret.secretAccessKey  secret.sessionToken
   secret.expiration   secret.leaseId  secret.type  secret.credentialType
   ```
3. **Hand off:** mask the values and set `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`. Every later step (AWS CLI, SDKs, Terraform) picks them up automatically.

## About "each integration is set up differently"
True, but that setup is a one-time admin job inside WC and the cloud account, done before any pipeline runs. The Action never touches it. It only has to call generate and pass back whatever fields come back, which works the same for AWS, Azure or anything added later.

## What the Action would need
1. A `dynamic-secrets` input: same entry format as `static-secrets`, but calls generate instead of get.
2. An optional `export-aws-env: true` to set the three `AWS_*` variables.
3. Leave `type`, `credentialType` and `expiration` unmasked. Otherwise masking `aws` redacts that word from every later log line.
4. Later, optionally: revoke the lease when the job ends (Azure only, since AWS STS creds can't be revoked early).

## Setup used
- WC dynamic secret `<folder>/<dynamic-secret-name>` (integration `<aws-integration>`, TTL 900s)
- AWS role `<deployer-role>`: write access to one S3 bucket only
- Pathfinder Workload Identity: service name `<service-name>`, bound to one repo and branch
