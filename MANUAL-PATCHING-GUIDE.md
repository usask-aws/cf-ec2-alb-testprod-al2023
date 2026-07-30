# Manual Patching Guide (Console) — Test/Prod EC2 Instances

For clients who aren't comfortable deploying the automated `cf-ec2-ssm-patch-manager.yaml`
stack, this guide covers patching the Test/Prod instances created by
`cf-ec2-alb-testprod-al2023.yaml` **manually, on demand, from the AWS Console**.

This uses **Patch Manager's "Patch now"** feature — the same underlying AWS-managed
SSM document (`AWS-RunPatchBaseline`) that the automated stack's Maintenance Windows
use. The behavior is identical either way; the only difference is that you trigger it
yourself whenever you choose, instead of it running on a recurring schedule. Nothing in
this guide requires deploying `cf-ec2-ssm-patch-manager.yaml` — it works against any
instance with the right SSM permissions, regardless of which stacks are deployed.

## Prerequisites

- The instance must be **running**, with an instance profile that grants SSM
  permissions (e.g. `AmazonSSMManagedInstanceCore`) — the same requirement the main
  README's Session Manager and automated-patching instructions already assume.
- Like the automated stack, this relies on the **account/region's default Patch
  Baseline** for the instance's OS (Amazon Linux 2023's default is
  `AWS-AmazonLinux2023DefaultPatchBaseline`, unless you've registered a custom one as
  the default). If the instance is associated with a **patch group**, that group's
  assigned baseline is used instead — most setups following this repo's conventions
  won't have patch groups configured, so the account/region default applies.
- This only patches instances in whichever **AWS Region** you're currently viewing in
  the console — same region-scoping caveat as the automated stack.

## Running "Patch now"

1. Sign in to the AWS Console for the target account and switch to the correct region
   (top-right region selector) — the same region as the Test/Prod instances.
2. Go to the **Systems Manager** service.
3. In the left navigation pane, choose **Patch Manager**.
4. Choose **Patch now**.
5. For **Patching operation**, choose:
   - **Scan** — finds missing patches without installing anything. **Recommended for
     your first run**, so you can review what would change before actually installing
     it (see **Verify before installing**, below).
   - **Scan and install** — finds and installs missing patches in the same step. Use
     this once you're ready to actually apply patches.
6. *(Only if you chose "Scan and install")* For **Reboot option**, choose one of:
   - **Reboot if needed** — reboots the instance only if a patch requires it. This
     matches the automated stack's own behavior (`RebootIfNeeded`) — recommended for
     consistency if you also run the automated stack elsewhere.
   - **Don't reboot my instances** — installs patches but never reboots automatically;
     you reboot manually later if needed. Some patches won't take effect until a
     reboot happens.
   - **Schedule a reboot time** — installs patches now, reboots at a specific
     date/time/timezone you choose separately.
7. For **Instances to patch**, choose **Patch only the target instances I specify**
   (not **Patch all instances** — that would include every managed instance in the
   account/region, not just the Test/Prod instances from this repo's templates).
8. In the **Target selection** section that appears, choose **Specify instance tags**
   and enter:
   - Tag key: `Environment` (matches `InstanceTagKey`'s default in both this repo's
     EC2/ALB and automated-patching templates)
   - Tag value: `test` to patch the Test instance, or `prod` to patch the Prod
     instance — run this procedure separately for each environment, one at a time,
     the same way the automated stack patches them on separate schedules (Tuesday for
     Test, Thursday for Prod) rather than together.
9. Leave **Patching log storage** and **lifecycle hooks** unconfigured unless you
   specifically need them (both optional; the automated stack doesn't use either).
10. Choose **Patch now**. The **Association execution summary** page opens — this is
    where you monitor progress (Patch now runs as a State Manager association, not a
    Maintenance Window Task, so it looks different from the automated stack's own
    verification steps, but reports the same underlying patch results).

## Verify before installing

Because **Scan** never installs anything, it's a safe way to preview what a
**Scan and install** run would do:

1. Run the steps above with **Patching operation: Scan**.
2. **Systems Manager → Patch Manager → Compliance** → find the instance → review
   which patches are reported missing.
3. Once you're satisfied, repeat the same steps with **Patching operation: Scan and
   install** to actually apply them.

## Verify after installing

1. **Systems Manager → Patch Manager → Compliance** → confirm the instance shows an
   updated compliance summary (the "last scan/install time" should reflect this run).
2. **Systems Manager → State Manager → Associations** → find the association created
   by this run → confirm **Status: Success**. If you chose **Schedule a reboot time**,
   a second association named `AWS-PatchRebootAssociation` also appears — see the
   warning below.
3. If the Prod instance is behind the ALB (via `cf-ec2-alb-testprod-al2023.yaml`):
   **EC2 → Target Groups** → open the Prod target group → confirm the instance briefly
   shows `unhealthy` then `healthy` again if the patch run triggered a reboot.

## Important: cleaning up a scheduled reboot

If you chose **Schedule a reboot time** in step 6 and later need to cancel the whole
patching operation after it's already started, AWS does **not** automatically cancel
the reboot — you must manually delete the `AWS-PatchRebootAssociation` association
yourself (**Systems Manager → State Manager → Associations**), or the instance will
still reboot unexpectedly at the scheduled time even though you cancelled the patch
install itself.

## How this differs from the automated stack

|  | Automated (`cf-ec2-ssm-patch-manager.yaml`) | Manual (this guide) |
|---|---|---|
| Trigger | Recurring schedule (Tuesdays/Thursdays 17:30, configurable) | You, whenever you choose |
| Underlying document | `AWS-RunPatchBaseline` | `AWS-RunPatchBaseline` (same) |
| Patch baseline used | Account/region default (or patch group's, if configured) | Same |
| Reboot behavior | Always `RebootIfNeeded` | Your choice each run |
| Notification | SNS message after every scheduled run (doesn't confirm success — see main README) | None automatic; you check Compliance/State Manager yourself |
| IAM | References a centrally-deployed role; deploying the stack creates none of its own | None — the EC2 instance's own instance profile (SSM permissions) is all that's needed |

Both approaches use the exact same patching mechanism under the hood — this guide
exists for clients who'd rather stay in full manual control than deploy the
CloudFormation stack that automates the schedule.
