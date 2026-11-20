---
name: aws-backup-failure-investigation
description: Investigation procedures for AWS Backup backup jobs and copy
  jobs in FAILED, ABORTED, or EXPIRED state, including jobs that expire
  before completion, IAM backup role permission errors, AWS KMS key disabled
  or access denied errors, cross-account and cross-Region copy failures,
  destination vault access policy errors, and Vault Lock retention or
  lifecycle violations. Use when a Backup Job State Change or Copy Job State
  Change event reports a failure, or when asked why an AWS Backup job failed.
---

<!-- last_verified: NOT YET TESTED against a live account. Update after the fault-injection runs. -->

# AWS Backup failure investigation

Use this skill when an AWS Backup backup job or copy job reports FAILED,
ABORTED, or EXPIRED. Classify the failure from its signals, follow one
branch, confirm with CloudTrail or configuration, then report.

A failed backup job cannot be retried. A scheduled plan creates a new job at
the next run, and an on-demand job needs a new StartBackupJob request.
Remediation therefore means fixing the cause so the next job succeeds.

## Step 1: Collect the signals

From the incident payload read: state, statusMessage, resourceArn,
resourceType, iamRoleArn, backupVaultArn (backup job), sourceBackupVaultArn
and destinationBackupVaultArn (copy job), the Region, and the account that
owns the failed job (the event's top-level `account` field).

If a field is missing, call backup:DescribeBackupJob or
backup:DescribeCopyJob with the job ID. Investigate in the account and Region
that owns the job, not in the delegated administrator account.

## Step 2: Classify

Match statusMessage and state against the table. Use
references/failure-signatures.md for the full list of known messages.

| Primary signal | Likely branch | Go to |
|---|---|---|
| State EXPIRED, "running job for the same resource", "did not complete within completion window", "expired before completion"; State ABORTED; vault or source resource not found | Job could not start or finish | Branch A |
| "not authorized to perform this operation", "insufficient privileges", "Access denied exception while trying to list tags" | Backup role (IAM) | Branch B |
| "KMS key is either disabled or pending deletion", "KMS key access denied", "KMS validation error", "Given key ID is not accessible" | Encryption key (KMS) | Branch C |
| Copy job with "same organization", "not supported for provided resource type", "default Backup service managed key", "Access Denied trying to call AWS Backup service", tag policy | Cross-account or cross-Region copy | Branch D |
| "lifecycle is outside the valid range for backup vault" | Vault Lock retention | Branch E |
| None of the above | Unclassified | Step 4 |

**Priority rule.** If more than one signal is present, check A, then B, then
C, then D, then E.
- A is the cheapest check and can end the investigation: a job stopped by a
  user is not a fault.
- B comes before C because the backup role is what calls AWS KMS. A role
  problem produces KMS-looking errors.
- D and E have distinctive messages and rarely occur together. Take them last.

## Branch A: Job could not start or finish

Investigation procedure:
1. If state is ABORTED and the message says the job was stopped by a user, this
   is not a fault. Find the StopBackupJob event in CloudTrail, report who
   stopped it and when, and end the investigation.
2. If state is EXPIRED, read percentDone, bytesTransferred and startBy, and
   compare with the rule's completion window (CompletionWindowMinutes). List
   jobs for the same resource around the same time (backup:ListBackupJobs by
   resource ARN) to find overlapping jobs.
3. If the message is about a missing vault, call backup:DescribeBackupVault
   and look for DeleteBackupVault in CloudTrail.
4. If the resource is missing or not backable, describe it with the owning
   service's API and look for delete or modify events before the job.
5. Check scheduling conflicts in references/backup-limits-and-thresholds.md:
   database maintenance and backup windows, DynamoDB table creation, EFS
   queued backups, EBS snapshot quota.

Interpreting the findings:
- If a second job for the same resource overlapped, two rules or an on-demand
  job collided. Stagger the schedules.
- If percentDone was still rising when the window closed, the window is too
  short for the data size. Lengthen CompletionWindowMinutes.
- If the job started inside a database maintenance or backup window, move the
  schedule outside it, or let AWS Backup manage snapshots and continuous
  backups together.
- If the vault or source resource was deleted, the plan still points at
  something that no longer exists. Fix the plan or selection.
- If none of these fit, go to Branch B.

## Branch B: Backup role (IAM)

Investigation procedure:
1. Read the role from iamRoleArn. Call iam:GetRole and
   iam:ListAttachedRolePolicies.
2. Confirm the trust policy lists backup.amazonaws.com.
3. Confirm the role carries the AWS Backup service role policies, or
   equivalent permissions for the resource type.
4. Confirm KMS permissions: at least Decrypt, CreateGrant and GenerateDataKey
   on the keys involved.
5. Confirm backup:TagResource on recovery point ARNs if the source has tags.
6. Check whether the job used AWSBackupDefaultServiceRole or a custom role.
   AWSServiceRoleForBackup is a different role with a different purpose.
7. In CloudTrail, look for DetachRolePolicy, DeleteRolePolicy,
   PutRolePolicy, UpdateAssumeRolePolicy and DeleteRole before the failure.
8. For tag-based selections, check ProcessBackupPlanSelection events for a
   tag:GetResources error, and check organization service control policies
   (SCPs) for a deny.

Interpreting the findings:
- If a role change event precedes the first failure, that change is the cause.
- If the trust policy lacks backup.amazonaws.com, AWS Backup cannot assume the
  role.
- If an SCP denies tag:GetResources, tag-based selections fail even though the
  role allows the action.
- If the role is unchanged and complete, go to Branch C.

## Branch C: Encryption key (KMS)

Investigation procedure:
1. Find the key. For fully managed resource types it is the destination
   vault's key (backup:DescribeBackupVault, EncryptionKeyArn). For other
   types it is the source resource's own key.
2. Call kms:DescribeKey and kms:GetKeyPolicy. For a copy job check both the
   source key and the destination key.
3. In CloudTrail, look for DisableKey, ScheduleKeyDeletion, PutKeyPolicy,
   RevokeGrant and DeleteImportedKeyMaterial near the failure.
4. For RDS, compare outcomes: on-demand backups can succeed while scheduled
   backups fail, because the on-demand path uses CopyDBSnapshot, which does
   not require KMS access from the backup role.

Interpreting the findings:
- KeyState Disabled or PendingDeletion is the cause. Re-enable, or cancel the
  deletion.
- A key policy that no longer names the account or role is the cause.
- If on-demand RDS backups succeed and scheduled ones fail, add the backup
  role to the key.
- If the key is healthy and the policy is correct, go to Branch D when the
  job is a copy, otherwise Step 4.

## Branch D: Cross-account or cross-Region copy

Investigation procedure:
1. Confirm source and destination accounts are in the same organization.
2. Confirm the resource type supports the copy combination. RDS, Aurora,
   DocumentDB and Neptune cannot do a cross-account and cross-Region copy in
   one action.
3. Confirm the destination vault uses a customer managed key, not the default
   AWS Backup managed key.
4. Confirm the source key policy allows the destination account (use
   references/copy-prerequisites.md).
5. Confirm the destination vault access policy allows the source account to
   call backup:CopyIntoBackupVault (backup:GetBackupVaultAccessPolicy in the
   destination account).
6. If the message mentions tags, check the organization's tag policy.

Interpreting the findings:
- "same organization" means the accounts are in different organizations.
- "not supported for provided resource type" means split the copy into two
  hops, one cross-account and one cross-Region.
- A default-key destination vault cannot receive copies. Use a vault with a
  customer managed key.
- "Access Denied trying to call AWS Backup service" points to the destination
  vault access policy.
- If all of these are correct, go to Branch E.

## Branch E: Vault Lock retention

Investigation procedure:
1. Call backup:DescribeBackupVault on the target vault. Read Locked,
   MinRetentionDays and MaxRetentionDays.
2. Read the rule's lifecycle (DeleteAfterDays and MoveToColdStorageAfterDays)
   from backup:GetBackupPlan. For a copy, read the copy action's lifecycle. If
   the copy action does not set one, it follows the source backup's lifecycle.
3. In CloudTrail, look for PutBackupVaultLockConfiguration and
   UpdateBackupPlan near the first failure.

Interpreting the findings:
- If DeleteAfterDays is below MinRetentionDays, or above MaxRetentionDays, or
  unset (retained indefinitely) while MaxRetentionDays is set, the job fails by
  design.
- If the lock was added after the plan, the plan is now out of range.
- In compliance mode after the grace time, the lock cannot be changed. Fix the
  plan, not the vault.
- Cold storage needs DeleteAfterDays of at least MoveToColdStorageAfterDays
  plus 90.

## Step 4: Report

Provide, in this order:
1. Failure state and matched branch, or "unclassified".
2. Root cause, with the CloudTrail event or configuration value that proves it.
3. Remediation in three tiers: immediate (restore the next job's chance of
   success), short-term (fix the configuration), long-term (prevent
   recurrence, for example alarms or policy-as-code).
4. For each fix that is a single AWS API call, name the exact operation and
   the target (for example kms:EnableKey on one key ARN) so an operator can
   ask you to apply it and approve it. Do not apply anything yourself. Mark
   fixes you cannot apply as operator steps, such as deleting resources or
   starting a job that passes a role.
5. What you ruled out, so a person can continue if the cause is unconfirmed.

If no branch confirmed a cause, say so. Do not guess. List what was checked
and recommend escalation to the backup administrator.
