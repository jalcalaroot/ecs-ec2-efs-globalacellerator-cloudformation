# ECS on EC2 with EFS, ALB, Global Accelerator and an Nginx Service (CloudFormation)

CloudFormation template that builds an ECS cluster on an EC2 Auto Scaling group with shared EFS storage, a public ALB behind AWS Global Accelerator, an ECR repository, a sample Nginx ECS service and a Route 53 alias record. (The repo name `globalacellerator` and the file name `ecs-globlal.yml` contain typos.)

> **Status:** legacy reference template (2026), not actively maintained. Review and update before deploying.

## Architecture

Template: `ecs-globlal.yml`

```
Global Accelerator (TCP 80/443) --> ALB --443--> target group (ip) --> ECS service "nginx" (awsvpc, EC2 launch type)
Route 53 A alias (DNSName) -------> ALB
ECS cluster <-- Auto Scaling group (ECS-optimized AL2) --NFS--> EFS mounted at /mnt/efs
```

- ECS cluster, launch template (encrypted gp3 root, no public IP) and Auto Scaling group with CPU step scaling (up >75%, down <70%)
- User data registers the instance, ships Docker logs to the `dockerlogs` log group and mounts EFS at `/mnt/efs`
- EFS (encrypted, elastic throughput, backups enabled) with three mount targets; NFS allowed only from the container instances
- Public ALB: HTTP 301 to HTTPS; HTTPS listener (ACM certificate, default SSL policy) forwards to an `ip` target group on port 80
- Global Accelerator with TCP listener on 80/443 pointing to the ALB
- ECR repository (`ECRName`) - not used by the task, which pulls `nginx:latest` from Docker Hub
- Task definition `nginx` (256 CPU / 256 MiB, awsvpc) with execution and task roles; ECS service with `DesiredCount = MinSize`
- Route 53 A alias record `DNSName` pointing to the ALB (the Global Accelerator alias is commented out)
- Instance IAM role/profile; outputs: cluster, ALB DNS, instance SG, EFS ID, accelerator DNS

## Parameters

| Name | Purpose | Default |
|------|---------|---------|
| `ProyectID` | Project name for the cluster and resource names | (required) |
| `KeyPair` | Existing EC2 key pair | `<KEY_PAIR_NAME>` |
| `VpcId`, `LoadBalancerSubnets`, `ECSSubnets` | VPC, ALB subnets, instance/task subnets | (required) |
| `IntraSubnet1/2/3` | EFS mount target subnets | (required) |
| `InstanceType`, `ISize` | Instance type, root volume GiB | `t3a.medium`, `100` |
| `DesiredCapacity` / `MinSize` / `MaxSize` | ASG sizing (MinSize also = service task count) | `2` / `2` / `4` |
| `healthCheckType` | ASG health check type | `ELB` |
| `LaunchTemplateVersionNumber` | Launch template version | `1` |
| `AmiID` | SSM parameter for the ECS-optimized AMI | `/aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id` |
| `DNSName` | Record name, e.g. `app.<DOMAIN>` | `<SUBDOMAIN>` |
| `ACMARN` | ACM certificate ARN | `<CERTIFICATE_ARN>` |
| `HostedZoneId` | Route 53 hosted zone | `<HOSTED_ZONE_ID>` |
| `ECRName` | ECR repository name | `nginx` |

## Deploy

```bash
aws cloudformation deploy \
  --stack-name <STACK_NAME> \
  --template-file ecs-globlal.yml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    ProyectID=<PROJECT_ID> KeyPair=<KEY_PAIR_NAME> VpcId=<VPC_ID> \
    LoadBalancerSubnets=<PUBLIC_SUBNET_1>,<PUBLIC_SUBNET_2> ECSSubnets=<PRIVATE_SUBNET_1>,<PRIVATE_SUBNET_2> \
    IntraSubnet1=<SUBNET_A> IntraSubnet2=<SUBNET_B> IntraSubnet3=<SUBNET_C> \
    DNSName=app.<DOMAIN> ACMARN=<CERTIFICATE_ARN> HostedZoneId=<HOSTED_ZONE_ID>
```

## Security notes

- Instance role is very broad: S3, SSM, CloudWatch, EFS and ECR full access, CodeDeploy policies and an inline `sts:*` on `*`.
- Container instance/task security group allows 80/443 from `0.0.0.0/0` and SSH from `10.0.0.0/8`.
- The task uses the mutable `nginx:latest` image from Docker Hub; the HTTPS listener uses the default (older) SSL policy.
- User data uses IMDSv1 and legacy packages (`python27-pip`, `awslogs`); IMDSv2 is not enforced.
- Template quirks: `Encrypted` is declared twice on the EFS file system, `LaunchTemplate: !` has a stray YAML tag, and the task log config sets `awslogs-create-group` while the execution role lacks `logs:CreateLogGroup`.

## Cleanup

```bash
aws ecr delete-repository --repository-name <ECR_NAME> --force   # only if images were pushed
aws cloudformation delete-stack --stack-name <STACK_NAME>
```

The ECR repository must be empty for stack deletion to succeed. Delete the `dockerlogs` log group manually; EFS backup recovery points stay in AWS Backup.

## License

MIT — see [LICENSE](LICENSE).
