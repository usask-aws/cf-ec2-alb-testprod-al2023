# Usask_EC2_Loadbalancer_Test_Prod_cloudformation_template.yaml

Creates a test/prod EC2 instance pair behind a single Application Load
Balancer. The default (port 80, and port 443 if a certificate is provided)
listener forwards to the **prod** target group; requests matching the
`/test/*` path pattern are routed to the **test** target group instead.

## Before uploading: update the UserData script

The template's `TestEC2Instance` and `ProdEC2Instance` resources each embed a
`UserData` bootstrap script (installs `httpd`, writes a placeholder
`index.php` and `/health` page). This is a **placeholder** — edit the
`UserData` block for each instance in the YAML to install and configure your
actual application before uploading the template, otherwise the instances
will only ever serve the sample "Test Works" / "Prod Works" pages.

Each instance also gets an additional EBS data volume (sized via
`DataVolumeSizeTest`/`DataVolumeSizeProd`) that the same `UserData` script
formats (XFS) and mounts at `/data` automatically on first boot, persisting
the mount across reboots via `/etc/fstab`. If your application needs a
different mount point, update the `MOUNT_POINT` value in the `UserData`
block accordingly.

`httpd`'s `DocumentRoot` is repointed from `/var/www/html` to
`$MOUNT_POINT/www/html` (i.e. `/data/www/html` by default) before the
service first starts, so all site content lives on the mounted EBS data
volume rather than the instance's root volume. If you change
`MOUNT_POINT`, the document root moves with it automatically.

## Deploy from the AWS Console

1. Sign in to the AWS Console for the target account and switch to the
   correct region (top-right region selector).
2. Go to the **CloudFormation** service.
3. Click **Create stack** → **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing
   template**.
5. Under **Specify template**, select **Upload a template file**, click
   **Choose file**, and select your edited
   `Usask_EC2_Loadbalancer_Test_Prod_cloudformation_template.yaml`. Click
   **Next**.
6. On **Specify stack details**:
   - Enter a **Stack name** (e.g. `usask-ec2-alb-test-prod`).
   - Fill in the parameters:
     | Parameter | Notes |
     |---|---|
     | `AmiIdTest` / `AmiIdProd` | Leave default to use the latest Amazon Linux 2023 AMI via SSM, or override |
     | `InstanceTypeTest` / `InstanceTypeProd` | Default `t2.micro` |
     | `TestSubnetId` | Subnet for the test instance |
     | `ProdSubnetId` | Subnet for the prod instance |
     | `LoadBalancerSubnetIds` | Must select exactly **two** subnets, in different AZs, and both must be **public** subnets (have a route to an internet gateway) — the ALB is internet-facing and won't provision correctly in private subnets |
     | `InstanceProfileName` | Name of an existing IAM instance profile (defaults to `DefaultEC2InstanceProfile`) |
     | `KeyPairNameTest` / `KeyPairNameProd` | Optional, independent per instance — leave either blank to disable SSH access for that instance. Use different key pairs for test and prod if you want separate SSH credentials per environment |
     | `TestEC2Name` / `ProdEC2Name` | Sets the instance's `Name` tag — use a descriptive instance name (e.g. `myapp-test`, `myapp-prod`) rather than the default placeholder |
     | `VpcId` | VPC containing the subnets above |
     | `LoadBalancerName` | Name of the ALB |
     | `TestTargetGroupName` / `ProdTargetGroupName` | Target group names |
     | `AppPort` | Port the app listens on (default `80`) |
     | `HealthCheckPath` | Target group health check path (default `/health`) |
     | `AllowedHttpCidr` | CIDR allowed to reach the ALB (default `0.0.0.0/0`) |
     | `CertificateArn` | Optional — leave blank to deploy HTTP-only. See **HTTPS/ACM** note below |
     | `DataVolumeTypeTest` / `DataVolumeSizeTest` | EBS volume type (default `gp3`) and size in GiB (default `20`) for the additional data volume attached to the test instance |
     | `DataVolumeTypeProd` / `DataVolumeSizeProd` | EBS volume type (default `gp3`) and size in GiB (default `20`) for the additional data volume attached to the prod instance |
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

## Verify the deployment in the console

1. **CloudFormation** → open the stack → **Outputs** tab → note
   `LoadBalancerDNSName`, `TestEC2InstanceId`, `ProdEC2InstanceId`,
   `TestTargetGroupArn`, `ProdTargetGroupArn`.
2. **EC2 → Instances** → confirm `TestEC2InstanceId` and `ProdEC2InstanceId`
   both show **Status check: 2/2 checks passed**.
3. **EC2 → Target Groups** → open the test and prod target groups → check
   the **Targets** tab → confirm each registered instance shows
   **Health status: healthy**. If a target is `unhealthy`, verify the
   instance's UserData actually started the app and serves the
   `HealthCheckPath`.
4. **EC2 → Load Balancers** → open the ALB → confirm **State: Active**, and
   check the **Listeners and rules** tab to confirm the `/test/*` rule
   exists on the correct listener (HTTPS listener if a certificate was
   supplied, otherwise HTTP).

## Load-test the prod and test sites via the ALB DNS name

Using the `LoadBalancerDNSName` output value (e.g.
`App-ALB-123456789.us-east-1.elb.amazonaws.com`):

- **Prod site**: open `http://<LoadBalancerDNSName>/` (or `https://` if a
  certificate is configured) in a browser, or run:
  ```bash
  curl -i http://<LoadBalancerDNSName>/
  ```
  This should return the prod instance's response (default listener action
  forwards here).
- **Test site**: open `http://<LoadBalancerDNSName>/test/` in a browser, or
  run:
  ```bash
  curl -i http://<LoadBalancerDNSName>/test/
  ```
  This matches the `/test/*` path-pattern rule and should return the test
  instance's response instead.
- If either request fails or times out, recheck the target group health
  (step 3 above) and confirm `AllowedHttpCidr` includes your client's IP/CIDR.

## Updating or deleting the stack

- **Update**: select the stack in CloudFormation → **Update** → **Replace
  current template** (re-upload the YAML if it changed) or **Use current
  template** (to change parameters only, e.g. adding `CertificateArn`) →
  adjust parameters → **Next** → **Submit**.
- **Delete**: select the stack → **Delete** → confirm. **This deletes the
  Application Load Balancer, both target groups, both EC2 instances, and
  both security groups** — there is no way to recover them once the stack
  deletion completes.
