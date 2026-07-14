# Usask_EC2_ASG_cloudformation_template.yaml

Creates an EC2 Auto Scaling Group (launch template + ASG) from a parameterized
VPC, subnets, security groups, and IAM instance profile.

## Deploy from the AWS Console

1. Sign in to the AWS Console for the target account and switch to the
   correct region (top-right region selector).
2. Go to the **CloudFormation** service.
3. Click **Create stack** → **With new resources (standard)**.
4. Under **Prerequisite - Prepare template**, select **Choose an existing
   template**.
5. Under **Specify template**, select **Upload a template file**, click
   **Choose file**, and select `Usask_EC2_ASG_cloudformation_template.yaml`
   from this repo. Click **Next**.
6. On **Specify stack details**:
   - Enter a **Stack name** (e.g. `usask-ec2-asg`).
   - Fill in the parameters:
     | Parameter | Notes |
     |---|---|
     | `AmiId` | Leave default to use the latest Amazon Linux 2023 AMI via SSM, or override |
     | `InstanceType` | Defaults to `t2.micro` |
     | `SubnetIds` | Select one or more subnets from the dropdown |
     | `SecurityGroupIds` | Select one or more existing security groups |
     | `InstanceProfileName` | Name of an existing IAM instance profile (defaults to `DefaultEC2InstanceProfile`) |
     | `MinSize` / `MaxSize` / `DesiredCapacity` | ASG sizing (defaults: 1/1/1) |
     | `KeyPairName` | Optional — leave blank to disable SSH access |
     | `LaunchTemplateName` | Optional name override |
     | `AutoScalingGroupName` | Optional name override |
     | `EC2Name` | Value for the `Name` tag on launched instances |
   - Click **Next**.
7. On **Configure stack options**, leave defaults (add tags/permissions
   boundary only if your account requires them) and click **Next**.
8. On **Review**, scroll down and confirm the stack details, then click
   **Submit**.
9. Wait for the stack **Status** to reach `CREATE_COMPLETE` (refresh via the
   circular arrow icon or enable auto-refresh).

## After deploying

Open the stack in the CloudFormation console and check the **Outputs** tab
for:

- `LaunchTemplateId` — ID of the created launch template.
- `AutoScalingGroupName` — name of the created Auto Scaling Group.

You can inspect running instances under **EC2 → Auto Scaling Groups**, or
view the launch template under **EC2 → Launch Templates**.

## Updating or deleting the stack

- **Update**: select the stack in CloudFormation → **Update** → **Replace
  current template** (re-upload the YAML if it changed) or **Use current
  template** (to change parameters only) → adjust parameters → **Next** →
  **Submit**.
- **Delete**: select the stack → **Delete** → confirm. **This deletes the
  Auto Scaling Group and terminates all of its running instances, and
  removes the launch template** — there is no way to recover them once the
  stack deletion completes.
