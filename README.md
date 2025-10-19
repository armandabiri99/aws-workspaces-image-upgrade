# 🧠 Amazon WorkSpaces One-Shot Image Upgrade Automation

This project automates **mass image upgrades** for tagged Amazon WorkSpaces fleets.\
It’s a **CloudFormation-driven, one-shot pipeline** that updates a target bundle to a new image, waits for the next automatic snapshot per WorkSpace, optionally notifies users via SSM, performs a controlled rebuild, generates a CSV report in S3, and can self-delete after completion.

---

## 🚀 What It Does

This stack orchestrates a **snapshot-chasing upgrade** for WorkSpaces environments using AWS native services.

### Workflow Summary

1. **Update** the target custom WorkSpaces bundle to a new `ImageId`.
2. **Discover** all WorkSpaces matching specific tags (e.g., `Env=Prod`).
3. **Wait** until the next *rebuild snapshot* appears for each WorkSpace.
4. **Notify users** (optional, via SSM message) before rebuild.
5. **Rebuild** each WorkSpace from the latest snapshot automatically.
6. **Generate report** in S3 (CSV with timestamps and results).
7. **Self-delete** the stack (optional).

---

## 🧩 AWS Services Used

| Service                       | Purpose                                                                                |
| ----------------------------- | -------------------------------------------------------------------------------------- |
| **AWS CloudFormation**        | Deploys and manages all infrastructure as code.                                        |
| **AWS Lambda**                | Handles bundle updates, snapshot polling, rebuilds, user notifications, and reporting. |
| **AWS Step Functions**        | Orchestrates the full upgrade workflow with retries and waits.                         |
| **Amazon SQS (DLQ)**          | Captures failed Lambda events for troubleshooting.                                     |
| **AWS KMS**                   | Encrypts DLQ, CloudWatch Logs, and optional S3 reports.                                |
| **Amazon CloudWatch Logs**    | Stores encrypted logs for every Lambda and workflow run.                               |
| **Amazon S3**                 | Stores the final upgrade CSV report.                                                   |
| **AWS Systems Manager (SSM)** | Sends user notification messages to active WorkSpaces.                                 |

---

## ⚙️ Prerequisites

Before deploying the template, make sure you have:

1. **AWS Account Permissions**

   - `AdministratorAccess` or equivalent custom policies for CloudFormation, Lambda, WorkSpaces, Step Functions, SSM, S3, and KMS.

2. **Networking**

   - Private subnets with **NAT egress** or **VPC endpoints** for: `logs`, `s3`, `sqs`, `ssm`, `ec2messages`, `ssmmessages`, `states`, `workspaces`.
   - A Security Group ID allowing outbound HTTPS traffic.

3. **Target Environment**

   - Existing **custom WorkSpaces Bundle** (`wsb-xxxxxx`) and **Image ID** (`wsi-xxxxxx`).
   - WorkSpaces tagged with a key/value pair (e.g., `Env=Prod`).

4. **Report Storage**

   - Existing **S3 bucket** for the CSV report.
   - (Optional) KMS key ARN for **SSE-KMS encryption**.

5. **Code Signing Profile**

   - AWS Signer profile version ARN for Lambda code signing (Warn mode).

---

## 🏗️ Deployment

### Using AWS Console

1. Download the template file:\
   [`workspaces-upgrade-fixed.yaml`](./workspaces-upgrade-fixed.yaml)
2. In **CloudFormation**, click **Create Stack → With new resources**.
3. Upload the file, fill parameters (`BundleId`, `ImageId`, `TagKey`, `TagValue`, etc.)
4. Confirm and **Create stack**.

### Using AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name WorkSpacesUpgrade \
  --template-body file://workspaces-upgrade-fixed.yaml \
  --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM \
  --parameters \
    ParameterKey=TargetTagKey,ParameterValue=Env \
    ParameterKey=TargetTagValue,ParameterValue=Prod \
    ParameterKey=BundleId,ParameterValue=wsb-xxxxxx \
    ParameterKey=ImageId,ParameterValue=wsi-yyyyyy \
    ParameterKey=ReportBucketName,ParameterValue=my-report-bucket \
    ParameterKey=SubnetIds,ParameterValue=subnet-abc123,subnet-def456 \
    ParameterKey=LambdaSecurityGroupId,ParameterValue=sg-zzz111
```

Once deployed, the **Step Function** executes automatically.\
Monitor progress in **Step Functions → Executions** and **CloudWatch Logs**.

---

## 🔆 Minimizing Data Loss (RPO)

Unlike plain `MigrateWorkspace`, which rebuilds from the *last available snapshot* (up to 12 hours old), this automation waits for the **next snapshot** before performing the rebuild.\
This ensures that **the most recent user data is safely included** in the rebuild snapshot, drastically reducing the **Recovery Point Objective (RPO)** from hours to minutes.

### Benefits

- **No more post-snapshot data loss:** Users’ latest changes are captured in the fresh snapshot.
- **Perfect for global deployments:** Even if users work 24/7 across time zones, the process ensures near-real-time protection.
- **Fully automated safety:** The workflow handles snapshot waiting, rebuild, and reporting without manual coordination.

---

## 💰 Cost and Lifecycle Management

The stack is designed to be cost-efficient when idle.

### 🔹 When `AutoSelfDeleteStack = true`

- The stack automatically deletes itself after writing the report.
- All related resources (Lambdas, Step Functions, DLQ, roles, etc.) are removed.
- Ideal for one-shot upgrades where you don’t plan to reuse the stack soon.
- **No ongoing cost** after deletion.

### 🔹 When `AutoSelfDeleteStack = false`

- The stack stays deployed for future reuse (simply update parameters for next image upgrades).
- All Lambdas, Step Functions, and SQS remain **idle** and cost **\$0** when not invoked.
- **Only small ongoing cost** (\~\$1 per AWS KMS key per month) applies **if the template created new KMS CMKs** for SQS or CloudWatch Logs.
- If you **provide your own KMS ARNs**, or accept minimal key cost, you can safely keep the stack around.
- Recommended if you plan to re-run upgrades every few months.

### ⚖️ Summary

| Setting                     | Behavior             | Typical Cost per Month  | Recommended When  |
| --------------------------- | -------------------- | ----------------------- | ----------------- |
| `AutoSelfDeleteStack=true`  | Deletes after run    | \$0                     | One-time upgrades |
| `AutoSelfDeleteStack=false` | Keeps stack reusable | \~\$0–\$2 (if new CMKs) | Periodic reuse    |

---

## 📊 Output

After completion:

- CSV report is uploaded to:
  ```
  s3://<ReportBucketName>/<ReportPrefix>/workspaces-upgrade-report-<timestamp>.csv
  ```
- Columns:
  ```
  WorkspaceId, ComputerName, BaselineSnapshotIso, FreshSnapshotIso,
  NotifySeconds, UserNotified, RebuildRequested, RebuildRequestedAt, TimedOutWaiting
  ```

If `AutoSelfDeleteStack=true`, the stack self-deletes after report upload.

---

## 🧰 Example Use Case

Imagine you’re managing **200 Amazon WorkSpaces** across multiple departments (e.g., Accounting, HR, and Engineering).\
You’ve built a **new Windows 11 custom image** that includes:

- Updated corporate software (Office, Chrome, antivirus)
- New domain policies and baseline configurations
- Optimized system settings for performance and security

Now, you need to roll this image out to all production WorkSpaces, but you can’t afford downtime, missed backups, or manual intervention.

With this automation:

1. You **update your custom WorkSpaces Bundle** with the new `ImageId`.
2. Tag all production WorkSpaces with `Env=Prod` (or use any tag-based filter).
3. Deploy this CloudFormation stack with your parameters:
   - `BundleId` → your target custom bundle
   - `ImageId` → your new image version
   - `ReportBucketName` → your audit/report S3 bucket
   - `SubnetIds` → private subnets with egress to AWS APIs
   - `AutoSelfDeleteStack` → `true` (for one-time upgrade)

Once deployed:

- The stack **updates the bundle**, discovers all tagged WorkSpaces, and begins watching each for its **next rebuild snapshot** (often created automatically overnight).
- As soon as a fresh snapshot is available, the automation optionally **notifies the user** through an SSM message (“Your WorkSpace will be rebuilt soon—please save your work”), then **initiates the rebuild**.
- When every WorkSpace has been rebuilt, it **writes a CSV report** to your S3 bucket summarizing:
  - Which WorkSpaces rebuilt successfully
  - Which timed out waiting for a snapshot
  - Whether notifications were delivered

Finally, if you enabled auto-deletion, the stack **removes itself**, leaving only the encrypted CloudWatch logs and the S3 report behind for compliance and auditing.

---

### 🔎 Real-World Impact

This automation replaces what would normally require:

- **Manual rebuilds** for each WorkSpace
- **Tracking snapshot completion times** by hand
- **Manual communication** with each user
- **Multiple AWS console sessions or CLI loops**

With a single CloudFormation deployment, you now get:

- Consistent, auditable, and **fully automated rebuilds**
- **No human error** or skipped instances
- **Nightly snapshot safety** before upgrade
- A ready-to-share **CSV report** for your management or compliance team

This process can be reused for future image upgrades by simply redeploying the same stack with a new `ImageId`.

---

## 🧑‍💻 Contributing

Contributions are **welcome and encouraged!**

1. Fork this repository.
2. Create a feature branch:
   ```bash
   git checkout -b feature/my-improvement
   ```
3. Make changes and validate:
   ```bash
   cfn-lint workspaces-upgrade-fixed.yaml
   ```
4. Submit a Pull Request.

Please follow AWS best practices for naming and IAM least-privilege.

---

## 📜 License

Distributed under the **MIT License** — feel free to use, modify, and share it.

---

## 🌟 Author

**Arman Dabiri**\
IT Director & Cloud Engineer @ Network Experts Inc.\
📧 [support@itproeng.com](mailto\:support@itproeng.com)

---

> *“Automate what you can, validate what you must — and always log everything.”*

