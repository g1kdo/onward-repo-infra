# delivery/ – Order service on ECS Fargate with blue/green releases

Infrastructure for the containerized order service. The application code, Dockerfile, release workflow, task definition and AppSpec live in [onward-order-service](https://github.com/g1kdo/onward-order-service).

Each template is its own CloudFormation stack, deployed by GitSync from its `*-deployment.yaml` file. Stacks share values through exports (`onward-dev-delivery-*`), so they are created in this order and deleted in reverse.

| # | Template | Stack | What it creates |
|---|---|---|---|
| 1 | `network.yaml` | `onward-dev-delivery-network-stack` | VPC, public subnets (load balancer only), private task subnets with **no internet route**, S3 gateway endpoint, interface endpoints for `ecr.api`, `ecr.dkr`, `logs` |
| 2 | `foundation.yaml` | `onward-dev-delivery-foundation-stack` | ECR repository, artifact bucket, GitHub OIDC provider and release role, EventBridge rule that starts the pipeline on an image push |
| – | *first release* | – | Push the app repository once so ECR holds an image (the ECS service needs one to start) |
| 3 | `loadbalancer.yaml` | `onward-dev-delivery-loadbalancer-stack` | Security groups, internet-facing ALB, blue and green target groups, listener |
| 4 | `service.yaml` | `onward-dev-delivery-service-stack` | ECS cluster, task execution role, log group, initial task definition, Fargate service (CodeDeploy controller), CPU auto scaling 2–4 tasks |
| 5 | `pipeline.yaml` | `onward-dev-delivery-pipeline-stack` | CodeDeploy application and blue/green deployment group (canary 10%, auto rollback), CodePipeline V2 |

## Release flow

Merge to `main` in the app repository → GitHub Actions (OIDC, no stored keys) pushes the image and deploy bundle → EventBridge → CodePipeline → CodeDeploy shifts ALB traffic from blue to green.

## Notes

- CloudFormation creates the service once; after that, **CodeDeploy owns task definition changes**. Change the container settings in `taskdef.json` in the app repository, not here.
- The stack role (`onward-dev-delivery-stack-role`) is created manually from `bootstrap/delivery-*.json`.
- Set `CreateOidcProvider: 'false'` in `foundation-deployment.yaml` if the account already has the GitHub OIDC provider.
