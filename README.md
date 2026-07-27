# cf-ec2-alb-testprod-al2023.yaml

Creates a test and prod EC2 instance pair behind a single Application Load
Balancer(ALB). The default (port 80, and port 443 if a certificate is provided)
listener forwards to the **prod** target group; requests matching the
`/test/*` path pattern are routed to the **test** target group instead.

## Before uploading: update the UserData script

The template's `TestEC2Instance` and `ProdEC2Instance` resources each embed a
`UserData` bootstrap script (installs `httpd`, writes a placeholder
`index.php` and `/health` page). This is a **placeholder** , edit the
`UserData` block for each instance in the YAML to install and configure your
actual application before uploading the template, otherwise the instances
will only ever serve the sample "Test Works" / "Prod Works" pages.

Each instance also gets an additional EBS data volume (sized via
`DataVolumeSizeTest`/`DataVolumeSizeProd`), created and attached as
**standalone `AWS::EC2::Volume` / `AWS::EC2::VolumeAttachment` resources**
(`TestDataVolume`/`ProdDataVolume` and their `*Attachment` resources) rather
than an inline instance block device. This is deliberate: the volume's
lifecycle is fully decoupled from the instance's, and both volumes have
`DeletionPolicy: Retain`. That means:

- If the instance is replaced (e.g. a stack update changes `AmiIdTest` /
  `InstanceTypeTest` and forces a new instance), CloudFormation re-attaches
  the **same** data volume to the new instance, no data loss.
- If the whole stack is **deleted**, the two data volumes are **not**
  deleted along with it, they're left behind (in `available` state once
  detached) so app content survives even full stack teardown. See
  **Updating or deleting the stack** below for cleanup.

Because the volume is attached as a separate step (not guaranteed present
the instant the instance boots), the `UserData` script polls for the device
(up to ~2.5 minutes) before formatting/mounting it. Once found, it's
formatted (XFS, first boot only) and mounted directly at **`/var/www`**,
persisting across reboots via `/etc/fstab` — deliberately the same path
`httpd` already uses by default, rather than an out-of-the-way path like
`/data`. If your application needs a different mount point, update the
`MOUNT_POINT` value in the `UserData` block accordingly.

Because the mount happens *before* `dnf install -y httpd` runs, the `httpd`
package populates `/var/www/html` (and `/var/www/cgi-bin`) directly onto
that freshly-mounted, empty volume when it installs — so `DocumentRoot`
stays at its unmodified default (`/var/www/html`) and **`httpd.conf` is
never edited**. All site content still ends up on the persistent EBS data
volume, it just gets there by mounting the volume at the path `httpd`
already expects, instead of moving `httpd`'s configuration to a different
path. If you change `MOUNT_POINT` away from `/var/www`, you'd need to
either repoint `DocumentRoot` yourself or accept that content will live
under the new path's `/html` subdirectory instead of the conventional one.

`httpd`'s access/error logs are also persisted on the data volume, using
the same "keep the default config, relocate what it points at" approach as
`DocumentRoot` above — but via a symlink rather than a second mount.
`httpd.conf`'s `ErrorLog "logs/error_log"` (and the equivalent
`CustomLog`) directive is a path relative to `ServerRoot` (`/etc/httpd`),
resolved through `/etc/httpd/logs` — which the `httpd` package ships as a
symlink to `/var/log/httpd` by default. The `UserData` script creates
`$MOUNT_POINT/logs` (i.e. `/var/www/logs`) on the persistent data volume,
then replaces that symlink so `/etc/httpd/logs` points at
`/var/www/logs` instead. No `httpd.conf` edit, and — since `/var/www` is
already the persistently-mounted data volume — no second mount/fstab
entry either; a symlink is enough, because the directory it points into is
already on persistent storage. This means logs now survive instance
replacement and full stack deletion the same way site content does. Log
rotation (`logrotate`) or shipping logs elsewhere can still be layered on
separately if you need retention beyond what's on the data volume.

## Deploy from the AWS Console

1. Sign in to the AWS Console for the target account and switch to the
   correct region (top-right region selector).
2. Go to the **CloudFormation** service.
3. Click **Create stack** → **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing
   template**.
5. Under **Specify template**, select **Upload a template file**, click
   **Choose file**, and select your edited
   `Usask_EC2_Loadbalancer_Test_Prod_Web_cloudformation_template.yaml`. Click
   **Next**.
6. On **Specify stack details**:
   - Enter a **Stack name** (e.g. `ec2-alb-test-prod-stack`).
   - Fill in the parameters:
     Parameters are listed here in alphabetical order, matching how the
     CloudFormation console prompts for them (it lists parameters
     alphabetically by name when no parameter groups are defined):

     | Parameter | Notes |
     |---|---|
     | `AllowedHttpCidr` | CIDR allowed to reach the ALB (default `0.0.0.0/0`) |
     | `AmiIdProd` | Leave default to use the latest Amazon Linux 2023 AMI via SSM, or override (prod instance) |
     | `AmiIdTest` | Leave default to use the latest Amazon Linux 2023 AMI via SSM, or override (test instance) |
     | `AppPort` | Port the app listens on (default `80`) |
     | `CertificateArn` | Optional, leave blank to deploy HTTP-only. See **HTTPS/ACM** note below |
     | `DataVolumeSizeProd` | Size in GiB (default `20`) for the additional data volume attached to the prod instance |
     | `DataVolumeSizeTest` | Size in GiB (default `20`) for the additional data volume attached to the test instance |
     | `DataVolumeTypeProd` | EBS volume type (default `gp3`) for the additional data volume attached to the prod instance |
     | `DataVolumeTypeTest` | EBS volume type (default `gp3`) for the additional data volume attached to the test instance |
     | `EC2NameProd` | Sets the prod instance's `Name` tag, use a descriptive name (e.g. `myapp-prod`) rather than the default placeholder |
     | `EC2NameTest` | Sets the test instance's `Name` tag, use a descriptive name (e.g. `myapp-test`) rather than the default placeholder |
     | `HealthCheckPath` | Target group health check path (default `/health`) |
     | `InstanceProfileName` | Name of an existing IAM instance profile (defaults to `DefaultEC2InstanceProfile`) |
     | `InstanceTypeProd` | Instance type for the prod instance (default `t2.micro`) |
     | `InstanceTypeTest` | Instance type for the test instance (default `t2.micro`) |
     | `KeyPairNameProd` | Optional, leave blank to disable SSH access to the prod instance |
     | `KeyPairNameTest` | Optional, leave blank to disable SSH access to the test instance. Use a different key pair than `KeyPairNameProd` if you want separate SSH credentials per environment |
     | `LoadBalancerName` | Name of the ALB |
     | `LoadBalancerSubnetIds` | Must select exactly **two** subnets, in different AZs, and both must be **PUBLIC** subnets |
     | `SubnetIdProd` | Subnet for the prod instance (MUST BE **PRIVATE**) |
     | `SubnetIdTest` | Subnet for the test instance (MUST BE **PRIVATE**) |
     | `TargetGroupNameProd` | Prod target group name |
     | `TargetGroupNameTest` | Test target group name |
     | `VpcId` | VPC containing the subnets above |
   - Click **Next**.
7. On **Configure stack options**, leave defaults (add tags/permissions
   boundary only if your account requires them) and click **Next**.
8. On **Review**, scroll down and confirm the stack details, then click
   **Submit**.
9. Wait for the stack **Status** to reach `CREATE_COMPLETE`.

## HTTPS / ACM note

If you don't yet have an approved ACM certificate, deploy the stack with
`CertificateArn` left blank, the ALB will be created with an HTTP-only
listener on port 80. Once your ACM certificate request is validated and
shows status **Issued** in the ACM console, come back and **update** the
stack with the certificate's ARN in `CertificateArn`; this adds the HTTPS
listener on port 443 without needing to recreate the stack.

## Best practice (optional, not enforced by this template): enable ALB access logging

This template does not enable Application Load Balancer access logging, and
does not require you to. Whether to turn it on is a cost/visibility
trade-off for you to make (S3 storage + request costs for every log file,
scaling with traffic volume) — it isn't mandatory the way the WAF
association is for internet-facing ALBs. That said, it's worth turning on
for any ALB serving real traffic: without it, there is no durable record of
requests that reached the load balancer, which matters if you ever need to
investigate suspicious traffic or reconstruct what a specific client did.

To enable it after this stack is deployed:

1. Have (or create) an S3 bucket to receive the logs — e.g. via
   `Usask_S3_cloudformation_template.yaml`, or a dedicated logs bucket.
2. Add a bucket policy statement allowing the Elastic Load Balancing service
   to write to it. For most regions (the current, service-principal-based
   method):
   ```json
   {
     "Effect": "Allow",
     "Principal": { "Service": "logdelivery.elasticloadbalancing.amazonaws.com" },
     "Action": "s3:PutObject",
     "Resource": "arn:aws:s3:::<your-logs-bucket>/*"
   }
   ```
   Older regions (e.g. GovCloud, China) instead require granting a
   region-specific ELB AWS account ID as the `Principal` rather than the
   service principal above — check the current AWS documentation for
   Application Load Balancer access logs for the account ID that applies to
   your region before assuming the statement above is sufficient.
   If you're adding this to the bucket created by
   `Usask_S3_cloudformation_template.yaml`, add the statement to that
   template's `S3BucketPolicy` resource (a bucket can only have one bucket
   policy document, so this can't be a second, separate `AWS::S3::BucketPolicy`
   resource pointed at the same bucket from this LB stack).
3. **EC2 → Load Balancers** → select the ALB → **Attributes** tab → **Edit**
   → enable **Monitoring** → **Access logs** → toggle on → enter the S3 URI
   (bucket and optional prefix) → **Save changes**.
4. Send a few requests through the ALB and confirm log files start
   appearing under the configured S3 prefix within a few minutes.

## Verify the deployment in the console

1. **CloudFormation** → open the stack → **Outputs** tab → note
   `LoadBalancerDNSName`, `TestEC2InstanceId`, `ProdEC2InstanceId`,
   `TestTargetGroupArn`, `ProdTargetGroupArn`, `TestDataVolumeId`,
   `ProdDataVolumeId`.
2. **EC2 → Instances** → confirm `TestEC2InstanceId` and `ProdEC2InstanceId`
   both show **Status check: 2/2 checks passed**.
3. Connect to each instance via **SSM Session Manager** (no SSH/bastion
   needed — the environment's SSM VPC endpoints are already deployed, and
   the instances have no public IP by design):
   - **EC2 → Instances** → select `TestEC2InstanceId` (or
     `ProdEC2InstanceId`) → **Connect** → **Session Manager** tab →
     **Connect**. This requires `InstanceProfileName` to include the
     `AmazonSSMManagedInstanceCore` policy (or equivalent).
   - Once connected, verify the mount and app locally:
     ```bash
     df -h /var/www
     systemctl status httpd
     curl -i http://localhost/health
     ```
     Confirm `/var/www` is mounted (from the `TestDataVolume`/`ProdDataVolume`
     EBS volume), `httpd` is `active (running)`, and the health check
     returns `200 OK`.
4. **EC2 → Target Groups** → open the test and prod target groups → check
   the **Targets** tab → confirm each registered instance shows
   **Health status: healthy**. If a target is `unhealthy`, verify the
   instance's UserData actually started the app and serves the
   `HealthCheckPath`.
5. **EC2 → Load Balancers** → open the ALB → confirm **State: Active**, and
   check the **Listeners and rules** tab to confirm the `/test/*` rule
   exists on the correct listener (HTTPS listener if a certificate was
   supplied, otherwise HTTP).

## Testing the prod and test sites via the ALB DNS name

Using the `LoadBalancerDNSName` output value (e.g.
`App-ALB-123456789.ca-central-1.elb.amazonaws.com`):

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
  both security groups**, there is **NO WAY** to **RECOVER** them once the stack
  deletion completes. **The two data volumes (`TestDataVolume` /
  `ProdDataVolume`) are intentionally NOT deleted** (`DeletionPolicy:
  Retain`), they'll remain in the account in `available` state, still
  billing, holding your app content and logs. If you're tearing the
  environment down for good, manually delete them afterwards via
  **EC2 → Volumes** once you've confirmed you no longer need the data (or
  snapshot them first if you might).

## Deploying real application code: pair this with the CodeBuild template

This template only gets you a placeholder app (see **Before uploading**
above) — it has no way to build or deploy your actual application code by
itself. For that, deploy `Usask_CodeBuild_EC2_Deploy_cloudformation_template.yaml`
as a **separate, follow-up stack** once these `test`/`prod` instances
exist.

That template provisions a CodeBuild project that pulls your app's source
from GitHub, builds it (via the app repo's own `buildspec.yml`), uploads the
build artifact to an S3 bucket, and deploys it onto these already-running
instances via SSM Run Command — targeting them by the `Environment` tag
this template already sets (`test`/`prod`) on `TestEC2Instance`/
`ProdEC2Instance`. In practice that means: deploy this template once, then
deploy the CodeBuild template **twice** — once with `Environment: test`,
once with `Environment: prod` — pointing both at the same GitHub repo, to
get an automatic build-and-deploy pipeline for each instance created here.

See `Usask_CodeBuild_EC2_Deploy_cloudformation_template.README.md` for full
setup steps, prerequisites (a GitHub PAT in Secrets Manager, an existing S3
bucket), and the account/region-level constraints that apply when deploying
it more than once (e.g. `CreateGitHubSourceCredential`).




# cf-ec2-alb-testprod-al2023-waf.yaml

Creates a regional AWS WAF Web ACL (with AWS-managed rule groups) and
associates it with an existing Application Load Balancer.

## Optional stack : Mandatory for public web services

This template is deployed as a **separate, standalone stack** from the ALB
itself, CloudFormation does not require it, and the ALB will function
without it. However, as per secuirty policy, **attaching this WAF Web ACL is
mandatory for any internet-facing web service** (any ALB with
`Scheme: internet-facing`, such as the one created by
`Usask_EC2_Loadbalancer_Test_Prod_cloudformation_template.yaml`). Do not
skip this step for a production or public-facing deployment, only
internal/private-only load balancers may go without it.

## Prerequisites

- An existing **regional** Application Load Balancer already deployed (e.g.
  via `Usask_EC2_Loadbalancer_cloudformation_template.yaml` or
  `Usask_EC2_Loadbalancer_Test_Prod_cloudformation_template.yaml`) and its
  ARN. Get the load balancer ARN from the console.
  **EC2 → Load Balancers → select the ALB → Description tab → ARN** (or copy
  it from the **Load Balancer ARN** field).

## Deploy from the AWS Console

1. Sign in to the AWS Console for the target account and switch to the
   correct region (top-right region selector), WAF must be created in the
   **same region** as the ALB (this template uses `Scope: REGIONAL`, not
   `CLOUDFRONT`).
2. Go to the **CloudFormation** service.
3. Click **Create stack** → **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing
   template**.
5. Under **Specify template**, select **Upload a template file**, click
   **Choose file**, and select `Usask_WAF_cloudformation_template.yaml`.
   Click **Next**.
6. On **Specify stack details**:
   - Enter a **Stack name** (e.g. `waf-app-alb-stack`).
   - Fill in the parameters:
     | Parameter | Notes |
     |---|---|
     | `WebACLName` | Name of the Web ACL (default `Standalone-App-WAF`) |
     | `LoadBalancerArn` | ARN of the existing ALB (from Prerequisites above) |
     | `EnableCommonRuleSet` | AWS Managed Common Rule Set — leave `true` unless you have a specific reason to disable it |
     | `EnableKnownBadInputsRuleSet` | AWS Managed Known Bad Inputs Rule Set — leave `true` |
     | `EnableAmazonIpReputationList` | AWS Managed Amazon IP Reputation List — leave `true` |
     | `EnableCloudWatchMetrics` | Emits per-rule CloudWatch metrics — leave `true` for visibility |
     | `EnableSampledRequests` | Stores a rolling sample of matched requests for troubleshooting — leave `true` |
   - Leave the following at their default (`false`) unless a specific
     managed sub-rule is blocking legitimate application traffic (see
     [Sub-rule Count-mode overrides](#sub-rule-count-mode-overrides-optional)
     below before enabling any of these):
     | Parameter | Sub-rule affected | Notes |
     |---|---|---|
     | `CountModeSizeRestrictionsBody` | `SizeRestrictions_BODY` (CommonRuleSet) | Set `true` if legitimately large request bodies (uploads, forms) are being blocked |
     | `CountModeSizeRestrictionsQueryString` | `SizeRestrictions_QUERYSTRING` (CommonRuleSet) | Set `true` if legitimately long query strings are being blocked |
     | `CountModeNoUserAgentHeader` | `NoUserAgent_HEADER` (CommonRuleSet) | Set `true` if internal health checks / service-to-service calls without a `User-Agent` header are being blocked |
     | `CountModeCrossSiteScriptingBody` | `CrossSiteScripting_BODY` (CommonRuleSet) | Set `true` if an app that legitimately accepts HTML/script-like body content (e.g. a CMS or rich-text field) is being blocked |
     | `CountModeCrossSiteScriptingQueryArguments` | `CrossSiteScripting_QUERYARGUMENTS` (CommonRuleSet) | Same as above, via query parameters |
     | `CountModeHostLocalhostHeader` | `Host_localhost_HEADER` (KnownBadInputsRuleSet) | Set `true` if internal traffic or health checks using a `localhost`/loopback `Host` header are being blocked |
     | `CountModePropfindMethod` | `PROPFIND_METHOD` (KnownBadInputsRuleSet) | Set `true` for WebDAV-based applications that legitimately use the `PROPFIND` HTTP method |
   - Click **Next**.
7. On **Configure stack options**, leave defaults (add tags/permissions
   boundary only if your account requires them) and click **Next**.
8. On **Review**, scroll down and confirm the stack details, then click
   **Submit**.
9. Wait for the stack **Status** to reach `CREATE_COMPLETE`.

## Sub-rule Count-mode overrides (optional)

Each `EnableCommonRuleSet`/`EnableKnownBadInputsRuleSet` toggle enables or
disables an entire AWS managed rule group at once. The 7 `CountMode*`
parameters above are more targeted: they switch **one specific sub-rule**
inside `CommonRuleSet` or `KnownBadInputsRuleSet` from `Block` to `Count`
(the request is logged and allowed through, not blocked), while every other
sub-rule in that group keeps blocking as normal. This lets you work around a
false positive against your application's own traffic without disabling an
entire rule group's protection.

Only 7 sub-rules — the ones most likely to false-positive against normal
application traffic — are exposed this way; every other sub-rule in these
managed groups (and all of `AmazonIpReputationList`) is not tunable from
this template. If you need to Count a sub-rule not listed above, edit the
template directly to add a matching parameter, condition, and
`RuleActionOverrides` entry following the same pattern.

All 7 parameters default to `"false"` (normal `Block` behavior) — leaving
every one at its default produces the exact same rendered template as
before these parameters existed. Before setting one to `"true"`, confirm
via the Web ACL's **Sampled requests** tab (see
[Verify the deployment in the console](#verify-the-deployment-in-the-console)
below) that the sub-rule in question is actually the one blocking your
traffic.

## Verify the deployment in the console

1. **CloudFormation** → open the stack → **Outputs** tab → note
   `WebACLArn`, `WebACLId`, `AssociatedLoadBalancerArn`.
2. **WAF & Shield** console → **Web ACLs** → select the Web ACL you created
   → confirm the **Associated AWS resources** tab lists your ALB.
3. Still on the Web ACL page, open the **Rules** tab and confirm the
   managed rule groups you enabled are listed with **Action: Block/Count**
   as expected (managed rule groups use their own internal action per rule
   when `OverrideAction` is `None`).
4. Send a request through the ALB and check the **Requests** /
   **Sampled requests** tab a few minutes later to confirm traffic is being
   evaluated (CloudWatch metrics may take a few minutes to appear under
   **CloudWatch → Metrics → WAFV2**).

## Updating or deleting the stack

- **Update**: select the stack in CloudFormation → **Update** → **Replace
  current template** (re-upload the YAML if it changed) or **Use current
  template** (to change parameters only, e.g. toggling a rule group) →
  adjust parameters → **Next** → **Submit**.
- **Delete**: select the stack → **Delete** → confirm. **This removes the
  Web ACL and its association with the ALB**, leaving the ALB
  unprotected, if the ALB is internet-facing, redeploy WAF (or point it at
  a replacement Web ACL) before leaving it in that state for any length of
  time.

