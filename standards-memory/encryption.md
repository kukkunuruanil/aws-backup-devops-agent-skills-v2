# Encryption standard (example)

- Every vault that holds tier-1 or tier-2 data uses a customer managed AWS KMS key. The default AWS Backup managed key is not allowed for these vaults.
- Customer managed keys used for backup have automatic rotation turned on and are in state Enabled.
- A key used as the destination of a cross-account copy has a key policy that names the source account and allows `kms:DescribeKey`, `kms:Encrypt`, `kms:Decrypt`, `kms:ReEncrypt*`, `kms:GenerateDataKey`, and `kms:CreateGrant` with `kms:GrantIsForAWSResource`.
- Recovery points for tier-1 and tier-2 resources must be encrypted. Snapshots encrypted with an AWS managed key cannot be shared across accounts, so they are reported as a gap where a cross-account copy is required.
