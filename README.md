# 🧠 Amazon WorkSpaces One-Shot Image Upgrade Automation

This project automates **mass image upgrades** for tagged Amazon WorkSpaces fleets.  
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

| Service | Purpose |
|----------|----------|
| **AWS CloudFormation** | Deploys and manages all infrastructure as code. |
| **AWS Lambda** | Handles bundle updates, snapshot polling, rebuilds, user notifications, and reporting. |
| **AWS Step Functions** | Orchestrates the full upgrade workflow with retries and waits. |
| **Amazon SQS (DLQ)** | Captures failed Lambda events for troubleshooting. |
| **AWS KMS** | Encrypts DLQ, CloudWatch Logs, and optional S3 reports. |
| **Amazon CloudWatch Logs** | Stores encrypted logs for every Lambda and workflow run. |
| **Amazon S3** | Stores the final upgrade CSV report. |
| **AWS Systems Manager (SSM)** | Sends user notification messages to active WorkSpaces. |

---

## ⚙️ Prerequisites

Before deploying the template, make sure you have:

1. **AWS Account Permissions**
   - `AdministratorAccess` or equivalent custom policies for CloudFormation, Lambda, WorkSpaces, Step Functions, SSM, S3, and KMS.

2. **Networking**
   - Private subnets with **NAT egress** or **VPC endpoints** for:
     `logs`, `s3`, `sqs`, `ssm`, `ec2messages`, `ssmmessages`, `states`, `workspaces`.
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
1. Download the template file:  
   [`workspaces-upgrade-fixed.yaml`](./workspaces-upgrade-fixed.yaml)
2. In **CloudFormation**, click **Create Stack → With new resources**.
3. Upload the file, fill parameters (`BundleId`, `ImageId`, `TagKey`, `TagValue`, etc.)
4. Confirm and **Create stack**.

### Using AWS CLI
```bash
aws cloudformation create-stack   --stack-name WorkSpacesUpgrade   --template-body file://workspaces-upgrade-fixed.yaml   --capabilities CAPABILITY_IAM CAPABILITY_NAMED_IAM   --parameters     ParameterKey=TargetTagKey,ParameterValue=Env     ParameterKey=TargetTagValue,ParameterValue=Prod     ParameterKey=BundleId,ParameterValue=wsb-xxxxxx     ParameterKey=ImageId,ParameterValue=wsi-yyyyyy     ParameterKey=ReportBucketName,ParameterValue=my-report-bucket     ParameterKey=SubnetIds,ParameterValue=subnet-abc123,subnet-def456     ParameterKey=LambdaSecurityGroupId,ParameterValue=sg-zzz111
```

Once deployed, the **Step Function** executes automatically.  
Monitor progress in **Step Functions → Executions** and **CloudWatch Logs**.

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

## 💡 Tips & Cost Optimization

- Increase `PollSeconds` (e.g., 300) to reduce Step Function/Lambda cost.
- Set `AutoSelfDeleteStack=false` if you want to reuse the stack.
- Keep NAT or VPC endpoints active only during the upgrade.
- Use **Reserved Concurrency** to control Lambda cost.

---

## 🧰 Example Use Case

> You’ve created a new Windows 11 image and need to roll it out to 200 WorkSpaces.  
> Tag your production WorkSpaces (`Env=Prod`), update the custom bundle, and deploy this stack.  
> It waits for snapshots, notifies users, rebuilds, and gives you a CSV audit in S3.

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

**Arman Dabiri**  
IT Director & Cloud Engineer @ Network Experts Inc.  
📧 [support@itproeng.com](mailto:support@itproeng.com)

---

> *“Automate what you can, validate what you must — and always log everything.”*
