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

GitHub Actions ──(OIDC)──▶ render taskdef (inject role ARNs) ──▶ S3 config-source.zip (if changed)
                       └──▶ build + push :ange_buhendwa_<sha> + :latest to ECR
ECR :latest push ──▶ EventBridge ──▶ CodePipeline (S3 config + ECR image) ──▶ CodeDeploy (blue/green) ──▶ ECS
```

- **No NAT Gateway**: tasks reach all required AWS services exclusively through
  VPC interface endpoints (`ecr.api`, `ecr.dkr`, `logs`, `sts`) and an S3
  gateway endpoint. This eliminates the per-hour/per-GB NAT cost.
- **Least-privilege security groups**: ALB allows `:80` from the internet; tasks
  accept the container port only from the ALB; endpoints accept `:443` from the
  full VPC CIDR (not just from the task SG — prevents lock-out when the task SG
  changes).
- **Mutable `latest` tag**: GitHub Actions pushes two tags per build — an
  immutable `ange_buhendwa_<sha>` for traceability and a moving `latest`. An
  EventBridge rule (filtered to `image-tag=latest`) starts the pipeline on the
  `latest` push; CodeDeploy gets the exact digest from `imageDetail.json`.
- **Deploy files via S3**: GitHub Actions renders `taskdef.json` (injecting the
  role ARNs + region from repo variables), bundles it with `appspec.yaml`, and
  uploads `config-source.zip` to S3 — but only when the rendered content
  changes. CodePipeline's S3 source consumes it; CodeDeployToECS substitutes
  only `<IMAGE1_NAME>`.
- **Auto scaling**: 1 desired / 1 min / 4 max, target tracking on average CPU.
- **OIDC**: GitHub Actions assumes an IAM role to push to ECR — no static keys.

## Files

| File | Purpose |
|------|---------|
| `template.yaml` | The full CloudFormation stack. |
| `deployment-config.json` | GitSync deployment file (template path + parameters + tags). |

> The CodeDeploy deploy descriptors (`taskdef.json`, `appspec.yaml`) live in
> the app repo under `deploy/`. The workflow renders `taskdef.json` (role ARNs
> injected) and uploads the bundle to S3, where the pipeline's S3 source reads
> it. See the app repo's `NOTES-question.md` for why ARNs are injected in CI
> rather than resolved from SSM.

## GitHub Actions variables (app repo)

Set these as **repository variables** (not secrets) in
`amalitech-lab-aws-ecs`:

| Variable | Example value | From stack output |
|----------|--------------|-------------------|
| `AWS_ROLE_ARN` | `arn:aws:iam::<account-id>:role/ecs-cicd-github-oidc-role` | `GitHubActionsRoleArn` |
| `AWS_REGION` | `eu-north-1` | — |
| `ECR_REPOSITORY` | `ecs-cicd-app` | — |
| `ECS_EXECUTION_ROLE_ARN` | `arn:aws:iam::<account-id>:role/ecs-cicd-task-exec-role` | `TaskExecutionRoleArn` |
| `ECS_TASK_ROLE_ARN` | `arn:aws:iam::<account-id>:role/ecs-cicd-task-role` | `TaskRoleArn` |

The IAM roles have deterministic names (`RoleName` set in the template), so the
ARNs are stable across re-deploys.

## Deploy — two phases

Because there is no bootstrap image (no NAT, no public egress), the stack must
be deployed in two passes:

**Pass 1** — `CreateService: "false"` (default in `deployment-config.json`):
1. Deploy the stack via GitSync. This creates VPC, endpoints, ECR, ALB, and
   IAM roles.
2. Read the stack outputs and set the five GitHub repository variables above.
3. Push to the app repo (`main`) to render+upload the deploy bundle and build +
   push the first real image. The `latest` tag lands in ECR.

**Pass 2** — change `CreateService` to `"true"` in `deployment-config.json`
and push to this repo:
4. GitSync re-deploys and creates the ECS service, CodeDeploy application,
   CodePipeline, and the EventBridge trigger. The pipeline immediately runs
   (ECR source sees `latest`) and performs the first blue/green deployment.

Every subsequent push to the app repo's `main` triggers the full pipeline
automatically via the EventBridge rule on the `latest` ECR push.

## Outputs

`AlbDnsName`, `EcrRepositoryUri`, `GitHubActionsRoleArn`, `TaskExecutionRoleArn`,
`TaskRoleArn`, `ArtifactBucketName`, `ClusterName`, `ServiceName` (only when
`CreateService=true`).
