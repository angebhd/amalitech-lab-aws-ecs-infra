# AWS ECS CI/CD — Infrastructure

CloudFormation (deployed via **GitSync**) for a highly available, containerized
Java web app on **Amazon ECS (Fargate)** with **blue/green** deployments.

> This is a standalone infrastructure repository. The application code
> (Spring Boot app + `Dockerfile` + GitHub Actions workflow) lives in
> [`amalitech-lab-aws-ecs`](https://github.com/angebhd/amalitech-lab-aws-ecs).

## Architecture

```
Internet ──▶ ALB (public subnets, 2 AZ)
                │  prod listener :80  ┐
                │  test listener :8080│─▶ blue / green target groups
                ▼
        ECS Fargate service (private subnets, 2 AZ)
                │   reaches AWS services via VPC endpoints only — no NAT
                │   ecr.api · ecr.dkr · logs · sts (interface) + s3 (gateway)
                ▼
              Amazon ECR (mutable tags, scan on push)

GitHub Actions ──(OIDC)──▶ build + push :ange_buhendwa_<sha> + :latest to ECR
ECR :latest push ──▶ CodePipeline (ECR + GitHub sources) ──▶ CodeDeploy (blue/green) ──▶ ECS
```

- **No NAT Gateway**: tasks reach all required AWS services exclusively through
  VPC interface endpoints (`ecr.api`, `ecr.dkr`, `logs`, `sts`) and an S3
  gateway endpoint. This eliminates the per-hour/per-GB NAT cost.
- **Least-privilege security groups**: ALB allows `:80` from the internet; tasks
  accept the container port only from the ALB; endpoints accept `:443` from the
  full VPC CIDR (not just from the task SG — prevents lock-out when the task SG
  changes).
- **Mutable `latest` tag**: the ECR source action in CodePipeline watches the
  `latest` tag. GitHub Actions pushes two tags per build — an immutable
  `ange_buhendwa_<sha>` for traceability and a moving `latest` that triggers
  the pipeline. CodeDeploy gets the exact digest from `imageDetail.json`.
- **Static deploy files**: `taskdef.json` and `appspec.yaml` in the app repo
  contain no dynamic values. CodeDeployToECS substitutes only `<IMAGE1_NAME>`.
- **Auto scaling**: 1 desired / 1 min / 4 max, target tracking on average CPU.
- **OIDC**: GitHub Actions assumes an IAM role to push to ECR — no static keys.

## Files

| File | Purpose |
|------|---------|
| `template.yaml` | The full CloudFormation stack. |
| `deployment-config.json` | GitSync deployment file (template path + parameters + tags). |

> The CodeDeploy deploy descriptors (`taskdef.json`, `appspec.yaml`) live in
> the app repo under `deploy/`. CodePipeline reads them directly from GitHub
> via the CodeConnections source action — no S3 upload step required.

## GitHub Actions variables (app repo)

Set these as **repository variables** (not secrets) in
`amalitech-lab-aws-ecs`:

| Variable | Example value |
|----------|--------------|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/ecs-cicd-github-oidc-role` |
| `AWS_REGION` | `eu-north-1` |
| `ECR_REPOSITORY` | `ecs-cicd-app` |

## Static ARNs in `deploy/taskdef.json`

The IAM roles have deterministic names (`RoleName` set in the template):

| Role | ARN pattern |
|------|-------------|
| Task execution | `arn:aws:iam::<account-id>:role/ecs-cicd-task-exec-role` |
| Task | `arn:aws:iam::<account-id>:role/ecs-cicd-task-role` |

After deploying the stack, update `deploy/taskdef.json` in the app repo with
your account ID and region (replacing the `YOUR_ACCOUNT_ID` / `YOUR_AWS_REGION`
placeholders), then commit and push.

## Deploy — two phases

Because there is no bootstrap image (no NAT, no public egress), the stack must
be deployed in two passes:

**Pass 1** — `CreateService: "false"` (default in `deployment-config.json`):
1. Deploy the stack via GitSync. This creates VPC, endpoints, ECR, ALB, IAM
   roles, and the CodeConnections connection.
2. Complete the GitHub handshake: AWS Console → CodePipeline → Settings →
   Connections → activate the pending connection.
3. Push to the app repo (`main`) to build and push the first real image via
   GitHub Actions. The `latest` tag lands in ECR.

**Pass 2** — change `CreateService` to `"true"` in `deployment-config.json`
and push to this repo:
4. GitSync re-deploys and creates the ECS service, CodeDeploy application,
   and CodePipeline. The pipeline immediately runs (ECR source sees `latest`)
   and performs the first blue/green deployment.

Every subsequent push to the app repo's `main` triggers the full pipeline
automatically via the ECR source action.

## Outputs

`AlbDnsName`, `EcrRepositoryUri`, `GitHubActionsRoleArn`, `GitHubConnectionArn`,
`ArtifactBucketName`, `ClusterName`, `ServiceName` (only when `CreateService=true`).
