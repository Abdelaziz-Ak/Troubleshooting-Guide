# Terraform Troubleshooting & Error Handling Guide

This document captures common errors, edge cases, and blind spots you might encounter while managing the DevOps infrastructure (EKS, Jenkins, RDS, Networking, IAM, etc.) with Terraform. This guide focuses on methodical troubleshooting—eliminating low-probability causes—and serves as a checklist when an apply, plan, or destroy fails. It covers both your specific project components and general Terraform principles.

---

## 1. State and Backend Management Errors

The [terraform-backend](./Terraform-Infra/terraform-backend/) manages the S3 bucket and DynamoDB lock table. Errors here typically block all operations.

### Error: `Error acquiring the state lock`
*   **Cause:** A previous `terraform apply` or `plan` was interrupted unexpectedly (network drop, killed terminal, CI/CD pipeline canceled), leaving the lock in DynamoDB.
*   **Methodical Troubleshooting:**
    1.  Confirm no one else on the team (or an automated pipeline) is actively running Terraform.
    2.  If safe, manually remove the lock.
*   **Fix:** Run `terraform force-unlock <LOCK_ID>`, referencing the lock ID provided in the error message.

### Error: `AccessDenied: Access Denied` (S3 or DynamoDB)
*   **Cause:** Your local AWS credentials have expired, or you are assuming the wrong IAM profile.
*   **Fix:** Re-authenticate your AWS CLI (`aws sts get-caller-identity`), check `AWS_PROFILE` env vars, and ensure your user has read/write to the S3 bucket and DynamoDB table.

### Error: `Error: Inconsistent dependency lock file` (Provider version mismatches)
*   **Cause:** Someone updated a module to require AWS provider `~> 5.0`, but your local `.terraform.lock.hcl` still points to `4.x`, or you are running Terraform across different OS platforms without proper lock file tracking.
*   **Fix:** Run `terraform init -upgrade` to download the latest matching providers and update the lock file.

---

## 2. Infrastructure-Specific Errors

### A. AWS EKS (Elastic Kubernetes Service)

Working with `eks infra` usually involves complex interactions between IAM and Networking.

#### Error: Cannot delete VPC or Subnets during `terraform destroy`
*   **Cause:** Kubernetes created AWS resources (like Application Load Balancers for Ingress, or EBS Volumes for Persistent Volumes) that Terraform doesn't know about. These resources keep Elastic Network Interfaces (ENIs) attached inside your subnets.
*   **Fix:** Use `kubectl` to delete Services (type LoadBalancer), Ingresses, and PVCs inside the cluster *before* running `terraform destroy`. If the cluster is already gone, manually locate the ENIs in the AWS EC2 console, detach them, delete them, and then re-run `terraform destroy`.

#### Error: Nodes fail to join the cluster (Stuck in `NotReady` or missing)
*   **Cause:** AWS EKS managed nodes look healthy in the AWS console but don't show up in `kubectl get nodes`.
*   **Fix:** 
    1.  **IAM Role:** Ensure the EKS Node Group IAM role has `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryReadOnly`, and `AmazonEKS_CNI_Policy`.
    2.  **Networking:** Private subnets *must* have a NAT Gateway to reach the EKS control plane and internet.
    3.  **Tags:** Subnets must be tagged `kubernetes.io/cluster/<cluster-name> = shared` or `owned`.

#### Error: `Unauthorized` when running `kubectl` commands
*   **Cause:** EKS cluster authentication is tied to the AWS IAM identity that *created* the cluster. If you created it via a Jenkins pipeline role but attempt to access it with your local AWS CLI user, you are blocked.
*   **Fix:** Ensure the provisioning script automatically updates the `aws-auth` ConfigMap in the `kube-system` namespace to map admin ARNs, or use the newer AWS EKS Access Entries APIs.

### B. Networking and Security Groups

#### Error: `DependencyViolation` or Cyclic Dependencies
*   **Cause:** Security Group A needs an ingress rule allowing traffic from Security Group B. But Security Group B needs an ingress rule allowing traffic from Security Group A. Terraform throws a cyclic dependency error or AWS API rejects it.
*   **Fix:** Create the base Security Groups (`aws_security_group`) first *without* inline ingress/egress blocks. Attach the rules via standalone `aws_security_group_rule` resources (this is why having a separate `security-group-rule` module is best practice).

### C. IAM Roles and Policies

#### Error: `InvalidParameterValue: Invalid IAM Instance Profile name`
*   **Cause:** IAM resources in AWS are "Eventually Consistent." Terraform creates the IAM Role/Profile, receives a success signal from the IAM API, and immediately tries to assign it to an EC2 instance/EKS node via the EC2 API. The EC2 API hasn't seen the role yet and fails.
*   **Fix:** Often, just wait 30 seconds and re-run `terraform apply`. For automation, insert `time_sleep` resources or `depends_on` blocks.

#### Error: `MalformedPolicyDocument: Syntax errors in policy`
*   **Cause:** Writing JSON directly as a multi-line string in Terraform is error-prone (missing commas or quotes).
*   **Fix:** Always use the `aws_iam_policy_document` data source or `jsonencode()` function to generate JSON policies dynamically safely.

### D. Relational Database Service (RDS)

#### Error: `FinalDBSnapshotIdentifier is required`
*   **Cause:** By default, Terraform tries to prevent accidental data loss. If you destroy an RDS instance and `skip_final_snapshot` is set to `false`, the destroy fails.
*   **Fix:** In dev/test, set `skip_final_snapshot = true`. In production, provide a valid `final_snapshot_identifier`.

#### Error: `Cannot upgrade from 13.1 to 14.2 without apply_immediately`
*   **Cause:** You changed the engine version for RDS in your `.tf` files. AWS requires explicit confirmation to apply disruptive database upgrades right now instead of waiting for the weekly maintenance window.
*   **Fix:** Add `apply_immediately = true` to the `aws_db_instance` block.

### E. Elastic Container Registry (ECR)

#### Error: `RepositoryNotEmptyException`
*   **Cause:** Trying to destroy an ECR repository that still contains Docker images.
*   **Fix:** Add `force_destroy = true` inside the `aws_ecr_repository` block if you don't care about the images.

### F. EC2 & Jenkins Servers

#### Error: "Terraform applied successfully," but Jenkins isn't running!
*   **Cause:** Your `user_data` script (Bash script passed to EC2 to install Jenkins, Docker, Java) failed halfway through. Terraform assumes success as soon as the EC2 instance transitions to "Running" state; it *does not* track the internal success of bash scripts.
*   **Fix:** 
    1.  SSH into the EC2 instance and run `cat /var/log/cloud-init-output.log` to see exactly which bash command failed.
    2.  Add `set -e` or `set -ex` at the top of your `user_data` scripts to fail fast and trace execution.

### G. AWS Secrets Manager

#### Error: `InvalidRequestException: You can't create this secret because a secret with this name is already scheduled for deletion.`
*   **Cause:** AWS soft-deletes secrets (7-30 days by default). If you `terraform destroy` a secret and then immediately run `terraform apply` using the same secret name, AWS blocks it.
*   **Fix:** In development, set `recovery_window_in_days = 0` on `aws_secretsmanager_secret` resources to force immediate deletion.

---

## 3. General Terraform Lifecycle & Syntax Errors

### Error: The "Count Shift" (Mass Recreation of Resources)
*   **Cause:** You used `count = length(var.subnets)` to create subnets or EC2 instances based on a list. You then decided to insert a new subnet into the *middle* of the list. Terraform shifts the index of all subsequent resources (index 2 becomes 3, etc.), interpreted as a directive to destroy and recreate all of them!
*   **Fix:** Always use `for_each` with a map or set of strings for iterating over resources. `for_each` assigns static string keys, so adding an element only modifies that specific key without shifting others.

### Error: `Cycle: module.A, module.B`
*   **Cause:** You created a logical loop. For example, the Jenkins Subnet needs the VPC ID, but the VPC module references an output from the Jenkins Subnet module. Wait... that's impossible to provision!
*   **Fix:** Break the dependencies. Usually requires lifting common data to a parent configuration directory, or hardcoding stable inputs.

### Error: `Data source returning exactly zero or multiple results`
*   **Cause:** A `data "aws_vpc"` or `data "aws_subnet"` block executes during the Plan phase. If the resource it's looking for hasn't been created yet (or your tag filters are wrong), it crashes.
*   **Fix:** 
    1.  Ensure tag filters are unique.
    2.  If the data block depends on a resource created in the same Terraform run, use resource outputs (`aws_vpc.main.id`) instead of data blocks where possible. Use `depends_on` only as a last resort.

### Error: `Provider configuration not present` during `terraform destroy`
*   **Cause:** You removed an `aws` provider block with a specific alias (e.g., region=eu-west-1) from your `.tf` file, but your Terraform state file still has resources deployed in that region. When you destroy, Terraform doesn't know how to authenticate to that region anymore.
*   **Fix:** Temporarily add the provider block back, run `terraform destroy`, and then delete the provider block from both the code and the state.

---

## 4. Developing a "Troubleshooting Mindset"

When a Terraform error occurs, avoid blindly copy-pasting to StackOverflow. Use this methodical approach:

1.  **Read the Red Text carefully:** Terraform errors are notoriously long. Scroll to the very middle or bottom where it provides the exact AWS API rejection reason (e.g., "Account limits reached").
2.  **Verify Authentication:** 20% of problems are expired tokens or using the wrong `AWS_PROFILE`.
3.  **Check CloudTrail / AWS Console:** If Terraform is opaque, logging into the AWS Console and trying to perform the exact same action manually will instantly reveal the AWS-sided constraint preventing the action.
4.  **Isolate the Issue:** If a massive run fails, isolate the blast radius using `terraform apply -target=module.specific_component`.
5.  **Enable Debug Logs:** If Terraform crashes without a clear AWS error, run it with trace enabled:
    ```bash
    TF_LOG=DEBUG terraform apply
    ```
6.  **Understand "Out of Band" resources:** Did manually created infrastructure sneak in? Terraform only knows what *it* provisions. Use AWS context to bridge the gap between what TF expects and reality.
