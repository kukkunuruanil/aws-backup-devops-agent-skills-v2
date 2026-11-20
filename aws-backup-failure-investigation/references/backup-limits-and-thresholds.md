# AWS Backup limits and thresholds used during investigation

Values come from the AWS Backup Developer Guide and re:Post Knowledge Center. Verify against current documentation before relying on them. Account-specific values (your windows, your quotas) belong in a memory store, not here.

## Scheduling and windows
| Condition | Threshold | Effect | Branch |
|---|---|---|---|
| AWS database services, snapshot backup near maintenance or automatic backup window | Cannot start 1 hour before or during the window | Snapshot backup fails. Aurora is exempt. | A |
| Amazon FSx, same condition | Cannot start 3 hours before or during the window | Backup fails. | A |
| RDS daily 30-minute backup window, when AWS Backup does not manage both snapshots and continuous backups | Backup started during the window | Backup fails. | A |
| RDS or Aurora backup started from the RDS console while an AWS Backup job runs | Conflict | `Backup job expired before completion.` | A |
| DynamoDB table being created | Typically a couple of minutes | Backup fails during creation. | A |
| Amazon EFS, very large file systems | Up to 7 days | Window can expire. Only one backup can be queued at a time. | A |
| Amazon EBS backups per Region per account | Soft quota 100,000 | Further backups fail. | A |

## Completion window
| Parameter | Meaning |
|---|---|
| `StartWindowMinutes` | How long a job can wait to start before it is cancelled. |
| `CompletionWindowMinutes` | Time allowed for the backup to complete after it starts. If it does not finish, AWS Backup stops the job and it shows EXPIRED. |

Read `percentDone` and `bytesTransferred` from the job. A job that was still progressing when the window closed needs a longer window. A job at 0% needs a different explanation.

## Vault Lock
| Setting | Value |
|---|---|
| `MinRetentionDays` | 1 day or more, optional |
| `MaxRetentionDays` | Up to 36,500 days, optional |
| Compliance mode `ChangeableForDays` (grace time) | 3 to 36,500 days |
| Governance mode | Lock can be removed by users with sufficient IAM permissions |
| Compliance mode after grace time | Lock and vault are immutable while recovery points exist |
| Enforcement | New backup and copy jobs with lifecycle outside min or max fail. Existing recovery points are not affected. |

## Lifecycle
| Rule | Value |
|---|---|
| Snapshot retention (`DeleteAfterDays`) | 1 day to 100 years, or indefinite if unset |
| Continuous backup retention | 1 to 35 days |
| Cold storage minimum | 90 days in cold, so `DeleteAfterDays` is at least `MoveToColdStorageAfterDays` + 90 |
| Warm storage before cold | AWS Backup recommends at least 8 days, so incremental backups are not forced back to full |
| Copy action with no lifecycle set | Follows the lifecycle of the source backup |

## Copy support
| Condition | Result |
|---|---|
| Source and destination accounts in different organizations | Copy fails |
| RDS, Aurora, DocumentDB, Neptune | No single cross-account plus cross-Region copy. Do one, then the other. |
| DynamoDB cross-account | Needs DynamoDB with AWS Backup advanced features |
| Destination vault encrypted with default AWS Backup managed key | Copy fails. Use a customer managed key. |
| First copy to a new Region or account | Full copy. Later copies are incremental if the service supports it. |

## Events
| Item | Value |
|---|---|
| EventBridge delivery | Best effort, about every 5 minutes |
| Backup job event fields | `backupJobId`, `backupVaultArn`, `backupVaultName`, `iamRoleArn`, `resourceArn`, `resourceType`, `state`, `statusMessage`, `creationDate`, `startBy`, `percentDone`, `bytesTransferred`, `retryCount` |
| Copy job event fields | `copyJobId`, `iamRoleArn`, `resourceArn`, `resourceType`, `sourceBackupVaultArn`, `destinationBackupVaultArn`, `state`, `statusMessage`, `creationDate`, `completionDate` |
| Account of the job | Top-level `account` field of the event, not in `detail` |
