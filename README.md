# Usask_EC2_Loadbalancer_Test_Prod_cloudformation_template.yaml

Creates a test/prod EC2 instance pair behind a single Application Load
Balancer. The default (port 80, and port 443 if a certificate is provided)
listener forwards to the **prod** target group; requests matching the
`/test/*` path pattern are routed to the **test** target group instead.

## Deploy from the AWS Console

1. Sign in to the AWS Console for the target account and switch to the
   correct region (top-right region selector).
2. Go to the **CloudFormation** service.
3. Click **Create stack** → **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing
   template**.
5. Under **Specify template**, select **Upload a template file**, click
   **Choose file**, and select
   `Usask_EC2_Loadbalancer_Test_Prod_cloudformation_template.yaml` from this
   repo. Click **Next**.
6. On **Specify stack details**:
   - Enter a **Stack name** (e.g. `usask-ec2-alb-test-prod`).
   - Fill in the parameters:
     | Parameter | Notes |
     |---|---|
     | `AmiIdTest` / `AmiIdProd` | Leave default to use the latest Amazon Linux 2023 AMI via SSM, or override |
     | `InstanceTypeTest` / `InstanceTypeProd` | Default `t2.micro` |
     | `TestSubnetId` | Subnet for the test instance |
     | `ProdSubnetId` | Subnet for the prod instance |
     | `LoadBalancerSubnetIds` | Select **two** subnets (different AZs) for the ALB |
     | `InstanceProfileName` | Name of an existing IAM instance profile (defaults to `DefaultEC2InstanceProfile`) |
     | `KeyPairName` | Optional — leave blank to disable SSH access |
     | `TestEC2Name` / `ProdEC2Name` | Name tags for each instance |
     | `VpcId` | VPC containing the subnets above |
     | `LoadBalancerName` | Name of the ALB |
     | `TestTargetGroupName` / `ProdTargetGroupName` | Target group names |
     | `AppPort` | Port the app listens on (default `80`) |
     | `HealthCheckPath` | Target group health check path (default `/health`) |
     | `AllowedHttpCidr` | CIDR allowed to reach the ALB (default `0.0.0.0/0`) |
     | `CertificateArn` | Optional — leave blank to deploy HTTP-only. See **HTTPS/ACM** note below |
   - Click **Next**.
7. On **Configure stack options**, leave defaults (add tags/permissions
   boundary only if your account requires them) and click **Next**.
8. On **Review**, scroll down and confirm the stack details, then click
   **Submit**.
9. Wait for the stack **Status** to reach `CREATE_COMPLETE`.

## HTTPS / ACM note

If you don't yet have an approved ACM certificate, deploy the stack with
`CertificateArn` left blank — the ALB will be created with an HTTP-only
listener on port 80. Once your ACM certificate request is validated and
shows status **Issued** in the ACM console, come back and **update** the
stack with the certificate's ARN in `CertificateArn`; this adds the HTTPS
listener on port 443 without needing to recreate the stack.

## After deploying

Open the stack in the CloudFormation console and check the **Outputs** tab
for:

- `LoadBalancerDNSName` — the ALB's public DNS name.
  - `http://<LoadBalancerDNSName>/` → routes to the **prod** instance.
  - `http://<LoadBalancerDNSName>/test/` → routes to the **test** instance.
- `TestEC2InstanceId` / `ProdEC2InstanceId` — instance IDs.
- `TestTargetGroupArn` / `ProdTargetGroupArn` — target group ARNs.
- `EC2SecurityGroupId` / `LoadBalancerSecurityGroupId` — security group IDs.

## Updating or deleting the stack

- **Update**: select the stack in CloudFormation → **Update** → **Replace
  current template** (re-upload the YAML if it changed) or **Use current
  template** (to change parameters only, e.g. adding `CertificateArn`) →
  adjust parameters → **Next** → **Submit**.
- **Delete**: select the stack → **Delete** → confirm. **This deletes the
  Application Load Balancer, both target groups, both EC2 instances, and
  both security groups** — there is no way to recover them once the stack
  deletion completes.
