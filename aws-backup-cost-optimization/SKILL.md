---
name: aws-backup-cost-optimization
description: Proactive AWS Backup cost analysis. Use on a schedule or on
  request to find spend you can remove, including retention longer than
  required, recovery points eligible for cold storage tiering, cross-Region
  or cross-account copies not required by policy, expired recovery points that
  AWS Backup could not delete, orphaned recovery points whose source resource
  was deleted, and empty or unused backup vaults.
---

<!-- last_verified: NOT YET TESTED against a live account. -->

# AWS Backup cost optimization

Use read-only configuration and usage data. Do not change any resource.

Read the standards from the attached `backup-standards` memory store. Never
recommend retention below the required minimum, even if it would save money.
Under-retention is a posture gap and belongs to the posture evaluation.

## Step 1: Retention above requirement
Compare each rule's DeleteAfterDays with the maximum in the standards. For each
rule above it, count recovery points older than the maximum
(backup:ListRecoveryPointsByBackupVault) and sum their sizes
(BackupSizeInBytes). Exclude vaults with a Vault Lock minimum that requires
the retention, because those recovery points cannot be removed.

## Step 2: Cold storage
For resource types that support cold storage, find recovery points kept in
warm storage for long periods. Recommend a transition only if the total
retention can be at least 90 days beyond the transition day, and warm storage
stays at least 8 days. Note the exceptions for EBS (monthly or less frequent
backups only) and for services with full backups in cold storage.

## Step 3: Redundant copies
List copy actions in plans (backup:GetBackupPlan). Compare each destination
with the resilience requirements in the standards. A copy no requirement
needs is a candidate. Estimate the monthly cost from the copied bytes.

## Step 4: Expired recovery points
Find recovery points in EXPIRED status. These are recovery points AWS Backup
could not delete, often because a role policy was changed or deleted. They
keep costing money until deleted. Report them with the likely cause, and note
that fixing the role (see the failure investigation, Branch B) lets deletion
proceed.

## Step 5: Orphaned and unused
Find recovery points whose source resource no longer exists. Find vaults with
no recovery points and no plan referencing them. List only; a person decides.

## Step 6: Report
Rank opportunities by estimated monthly savings. For each, give the resource,
the evidence, the assumption behind the estimate (pricing source and date), and
the risk of acting. State clearly where the agent could not obtain pricing and
the figure is a size estimate only.
