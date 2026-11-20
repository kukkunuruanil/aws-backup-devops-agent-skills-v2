# CloudTrail events to check, by branch

Look in the account and Region that own the failed job. Search from the previous successful job (or 7 days back) to the failure. A change is a suspect only if it precedes the first failure.

| Branch | Service (eventSource) | Event names |
|---|---|---|
| A | backup.amazonaws.com | `StopBackupJob`, `DeleteBackupVault`, `DeleteBackupPlan`, `UpdateBackupPlan`, `DeleteBackupSelection`, `StartBackupJob` (overlapping on-demand job) |
| A | the resource's own service | delete or modify events on the resource (for example `DeleteVolume`, `DeleteTable`, `ModifyDBInstance`) |
| B | iam.amazonaws.com | `DetachRolePolicy`, `DeleteRolePolicy`, `PutRolePolicy`, `AttachRolePolicy`, `UpdateAssumeRolePolicy`, `DeleteRole` |
| B | backup.amazonaws.com | `ProcessBackupPlanSelection` (look for a `tag:GetResources` error) |
| B | organizations.amazonaws.com | `AttachPolicy`, `DetachPolicy`, `UpdatePolicy` (SCP changes) |
| C | kms.amazonaws.com | `DisableKey`, `ScheduleKeyDeletion`, `PutKeyPolicy`, `RevokeGrant`, `DeleteImportedKeyMaterial` |
| D | backup.amazonaws.com | `PutBackupVaultAccessPolicy`, `DeleteBackupVaultAccessPolicy`, `UpdateBackupPlan` (copy action changed) |
| D | kms.amazonaws.com | `PutKeyPolicy` on the source key |
| D | organizations.amazonaws.com | `MoveAccount`, `RemoveAccountFromOrganization`, tag policy changes |
| E | backup.amazonaws.com | `PutBackupVaultLockConfiguration`, `DeleteBackupVaultLockConfiguration`, `UpdateBackupPlan` |

## AWS Backup service events in CloudTrail
AWS Backup also writes its own events, including `BackupJobStarted`, `BackupJobCompleted`, `CopyJobStarted`, `CopyJobCompleted`, `RecoveryPointCreated` and `ProcessBackupPlanSelection`. They show when a job started and whether a plan selection ran. They are not made through the public APIs.

## Reading a result
- Compare the event time with the job's `creationDate`.
- Record the identity that made the change (`userIdentity`) so the report can name who and what changed.
- If CloudTrail shows no relevant event, say so. A configuration that was wrong from the start has no change event.
