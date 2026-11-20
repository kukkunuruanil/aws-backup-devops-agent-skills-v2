# AWS Backup failure signatures

Maps known status messages to investigation branches in `SKILL.md`. Messages are matched by fragment. Wording can vary by resource type and over time. Treat a mismatch as "unclassified", not as "no failure".

Source column: **EB** = AWS Backup EventBridge documentation, **TS** = AWS Backup troubleshooting documentation, **KC** = AWS re:Post Knowledge Center article.

| Status message (fragment) | State | Branch | Meaning and first check | Source |
|---|---|---|---|---|
| "Backup job was stopped by user." | ABORTED | A | Not a fault. Find `StopBackupJob` in CloudTrail. | EB |
| "Backup job failed because there was a running job for the same resource." | EXPIRED | A | Overlapping job. List jobs for the resource. | EB |
| "Backup Job did not complete within completion window" | EXPIRED | A | Window shorter than the transfer. Compare `percentDone` and `CompletionWindowMinutes`. | KC |
| "An AWS Backup job failed to complete in time" | EXPIRED | A | Same as above. | KC |
| "Backup job expired before completion." | EXPIRED | A | RDS or Aurora job started from the RDS console conflicted with the AWS Backup job. Lengthen the window. | TS |
| "Backup job failed because backup vault ... does not exist." | FAILED | A | Plan points at a deleted vault. `DescribeBackupVault`, then `DeleteBackupVault` in CloudTrail. | EB |
| "You are not authorized to perform this operation" | FAILED | B | Role permissions. Check role policies and trust. | KC |
| "Backup job failed because of insufficient privileges" | FAILED | B | Same. | KC |
| "Access denied exception while trying to list tags" | FAILED | B | Role lacks tag permissions on the resource. | EB |
| `ProcessBackupPlanSelection` event with `tag:GetResources` error | n/a | B | SCP denies `tag:GetResources` for tag-based selections. | TS |
| "FAILED - KMS key is either disabled or pending deletion or access to KMS key is denied" | FAILED | C | `DescribeKey` for state, then `GetKeyPolicy`. | KC |
| "KMS key access denied error" | FAILED | C | Key policy or role. | KC |
| "KMS validation error" | FAILED | C | Key state. | KC |
| "The source snapshot KMS key does not exist, is not enabled or you do not have permissions to access it" | FAILED | C | RDS scheduled backup fails while on-demand succeeds: add the backup role to the key. In a cross-account copy, check the source key policy for the destination account (Branch D). | TS, KC |
| "Given key ID is not accessible. You must have DescribeKey permissions on the default CMK" | FAILED | C | Key policy missing `kms:DescribeKey` for the principal. In a cross-account copy, see Branch D. | KC |
| "Copy job failed. Both source and destination account must be a member of the same organization" | FAILED | D | Accounts are in different organizations. | KC |
| "Copy job from ... cannot be initiated for RDS resources. Feature is not supported for provided resource type." | FAILED | D | Split into a cross-account copy and a cross-Region copy. | KC |
| "Copy job failed because the destination Backup vault is encrypted with the default Backup service managed key. The contents of this vault cannot be copied. Only the contents of a Backup vault encrypted by a customer master key (CMK) may be copied" | FAILED | D | Destination vault key. | KC |
| "Snapshots encrypted with the AWS Managed CMK can't be shared." | FAILED | D | Source snapshot uses an AWS managed key. Re-encrypt with a customer managed key. | KC |
| "Access Denied trying to call AWS Backup service" | FAILED | D | Destination vault access policy lacks `backup:CopyIntoBackupVault` for the source account. | KC |
| "We are unable to copy resource tags to your backup because of the Internal Failure" | FAILED | D | Tag policy. | KC |
| "The tag policy does not allow the specified value for the following tag key: ..." | FAILED | D | Tag policy. | KC |
| "Backup job failed because the lifecycle is outside the valid range for backup vault" | FAILED | E | Vault Lock min or max retention. | KC |

## Notes
- Copy job states documented for EventBridge include CREATED, RUNNING, COMPLETED, FAILED. Backup job states include FAILED, ABORTED, EXPIRED. The Part 1 rule matches all three failure states for both job types, which is harmless.
- AWS Backup emits events on a best-effort basis about every 5 minutes. A failure can reach the agent several minutes after it happens. Always compare CloudTrail timestamps with the job's `creationDate`, not with the event arrival time.
- Add messages from your own environment at the bottom of this table. Do not edit the rows above without checking the source.
