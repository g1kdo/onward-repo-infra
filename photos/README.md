# photos/ – Storing and serving delivery photos

Infrastructure for the delivery photos service. The foundation (templates bucket, ECR repository, GitHub OIDC roles, stack role, replica bucket, signing key) comes from [onward-bootstrap](https://github.com/g1kdo/onward-bootstrap). The application lives in [onward-photos-app](https://github.com/g1kdo/onward-photos-app).

## One root stack, eight nested stacks

`root.yaml` is the only stack created by hand (`onward-dev-photos-root-stack`, deployed by GitSync from `root-deployment.yaml` with the role `service-role/onward-dev-photos-stack-role`). Values pass between nested stacks as parameters, so there are no exports.

| Nested template | Responsibility | Gets from |
|---|---|---|
| `network.yaml` | VPC, public subnets (ALB), app subnets (tasks), data subnets (database), no NAT | – |
| `security.yaml` | All security groups: internet → ALB → tasks → database / endpoints | network |
| `endpoints.yaml` | S3 gateway endpoint, interface endpoints for ECR, Logs, Secrets Manager | network, security |
| `storage.yaml` | Private, versioned photo bucket, lifecycle, replication to eu-central-1 | – |
| `database.yaml` | PostgreSQL Multi-AZ, encrypted, private, password managed by Secrets Manager, 7-day PITR | network, security |
| `loadbalancer.yaml` | Internet-facing ALB on HTTP, also the CloudFront origin for app pages | network, security |
| `cdn.yaml` | CloudFront: app pages from the ALB, `/photos/*` from S3 via OAC with signed URLs; photo bucket policy | storage, loadbalancer |
| `service.yaml` | ECS cluster, roles, task definition, Fargate service with circuit-breaker rollback | all of the above |

## How the requirements are met

| Requirement | How |
|---|---|
| Photos only viewable through Onward | Bucket is private (Block Public Access); only this CloudFront distribution may read it (OAC + `AWS:SourceArn`); every photo link is signed by the app and expires after 5 minutes |
| No direct storage access | Tasks may only `PutObject`; viewers never talk to S3 |
| Database unreachable from the internet | Data subnets have no internet route, `PubliclyAccessible: false`, security group allows tasks only |
| No credentials in code | RDS-managed secret read at runtime by the task role; signing key in Secrets Manager; GitHub uses OIDC |
| Recovery after a deletion | Bucket versioning (30-day recovery window) and RDS point-in-time recovery (7 days) plus a final snapshot |
| Survives loss of a Region | Every photo version replicated to `onward-dev-photos-replica-<account>` in eu-central-1; deletions are not replicated |
| Fast, low-cost global viewing | CloudFront `PriceClass_200`, photos cached at the edge |

## How templates reach CloudFormation

1. A push that changes `photos/*.yaml` runs [publish-photos-templates](../.github/workflows/publish-photos-templates.yml) (OIDC role from the `PHOTOS_TEMPLATES_ROLE_ARN` secret).
2. It validates the templates and uploads the nested ones to `s3://onward-dev-bootstrap-templates-<account>/photos/<commit-sha>/`.
3. It commits that SHA as `TemplatesVersion` in `root-deployment.yaml`; GitSync then updates the root stack.

## Notes

- The photo bucket is **retained** when the stack is deleted (business records). Empty and delete it by hand after the review if it is no longer needed.
- The database keeps a final snapshot on stack deletion; delete it by hand when no longer needed.
- CloudFront talks to the ALB over HTTP because there is no custom domain or certificate in this lab. Viewers always use HTTPS to CloudFront.
