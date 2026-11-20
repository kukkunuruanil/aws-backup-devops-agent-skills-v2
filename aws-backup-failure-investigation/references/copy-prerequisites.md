# Cross-account copy prerequisites

Replace `SourceAccountID` and `DestinationAccountID` with real account IDs. These are examples for comparison, not policies to apply blindly. Scope them to your own key and vault.

## 1. Source key policy must allow the destination account
Compare the source key's policy with this shape. A missing statement is the usual cause of "Given key ID is not accessible" and "source snapshot KMS key does not exist, is not enabled or you do not have permissions".

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::SourceAccountID:root" },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow use of the key",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::SourceAccountID:root",
          "arn:aws:iam::DestinationAccountID:root"
        ]
      },
      "Action": [
        "kms:DescribeKey",
        "kms:Encrypt",
        "kms:Decrypt",
        "kms:ReEncrypt*",
        "kms:GenerateDataKey",
        "kms:GenerateDataKeyWithoutPlaintext"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Allow attachment of persistent resources",
      "Effect": "Allow",
      "Principal": {
        "AWS": [
          "arn:aws:iam::SourceAccountID:root",
          "arn:aws:iam::DestinationAccountID:root"
        ]
      },
      "Action": ["kms:CreateGrant", "kms:ListGrants", "kms:RevokeGrant"],
      "Resource": "*",
      "Condition": { "Bool": { "kms:GrantIsForAWSResource": "true" } }
    }
  ]
}
```

## 2. Destination vault access policy must allow the source account
Read it with `backup:GetBackupVaultAccessPolicy` in the destination account.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::SourceAccountID:root" },
      "Action": "backup:CopyIntoBackupVault",
      "Resource": "*"
    }
  ]
}
```

AWS Backup rejects any other cross-account action on a vault regardless of the vault access policy, so `backup:CopyIntoBackupVault` is the only cross-account action to look for.

## 3. Encryption rules
- The copy is encrypted with the customer managed key of the destination vault.
- A destination vault using the default AWS Backup managed key cannot receive copies.
- Fully managed resource types use the backup vault's key. Others use the source resource's key.
- Snapshots encrypted with an AWS managed key cannot be shared. Re-encrypt with a customer managed key first.

## 4. Other checks
- Both accounts are in the same organization.
- The resource type supports the copy combination (see `backup-limits-and-thresholds.md`).
- Organization tag policy accepts the tags AWS Backup copies to the recovery point.
