# delivery/ – Order service on ECS Fargate with blue/green releases

Infrastructure for the containerized order service. The application code, Dockerfile, release workflow, task definition and AppSpec live in [onward-order-service](https://github.com/g1kdo/onward-order-service).

## One root stack, five nested stacks

`root.yaml` is the only stack created by hand (`onward-dev-delivery-root-stack`, deployed by GitSync from `root-deployment.yaml`). It owns five nested stacks and passes values between them as parameters (`!GetAtt <Stack>.Outputs.<Name>`), so there are no cross-stack exports.

| Nested template | What it creates | Gets from |
|---|---|---|
| `network.yaml` | VPC, public subnets (load balancer only), private task subnets with **no internet route**, S3 gateway endpoint, interface endpoints for `ecr.api`, `ecr.dkr`, `logs` | – |
| `foundation.yaml` | ECR repository, artifact bucket, GitHub OIDC release role, EventBridge rule that starts the pipeline on an image push | – |
| `loadbalancer.yaml` | Security groups, internet-facing ALB, blue and green target groups, listener | network |
| `service.yaml` | ECS cluster, task execution role, log group, initial task definition, Fargate service (CodeDeploy controller), CPU auto scaling 2–4 tasks | network, foundation, loadbalancer |
| `pipeline.yaml` | CodeDeploy blue/green deployment group (canary 10%, auto rollback), CodePipeline V2 | foundation, loadbalancer, service |

`service.yaml` and `pipeline.yaml` are only created when `DeployApplication` is `'true'` in `root-deployment.yaml`, because the ECS service needs an image in ECR before it can start.

## How templates reach CloudFormation

Nested stacks read their templates from S3, not from Git:

1. A push that changes `delivery/*.yaml` runs [publish-templates](../.github/workflows/publish-templates.yml) (OIDC role, no stored keys).
2. It validates the templates and uploads the nested ones to `s3://onward-dev-cfn-templates-<account>/delivery/<commit-sha>/`.
3. It commits that SHA as `TemplatesVersion` in `root-deployment.yaml`.
4. GitSync sees the deployment file change and updates the root stack, which reads the nested templates from that exact S3 folder.

Every deployment is tied to one commit's templates, so going back to an earlier version means setting `TemplatesVersion` to that commit's SHA.

## Release flow (application)

Merge to `main` in the app repository → GitHub Actions (OIDC) pushes the image and deploy bundle → EventBridge → CodePipeline → CodeDeploy shifts ALB traffic from blue to green.

## Notes

- After the service is created, **CodeDeploy owns task definition changes**. Change container settings in `taskdef.json` in the app repository, not here.
- Created manually from `bootstrap/` (once): the stack role `onward-dev-delivery-stack-role`, the templates bucket `onward-dev-cfn-templates-<account>`, and the infra repository's OIDC role `onward-dev-infra-github-actions-role`.
- GitHub issues immutable OIDC subjects (`repo:<owner>@<id>/<repo>@<id>`); read them with `gh api repos/<owner>/<repo>/actions/oidc/customization/sub`.
