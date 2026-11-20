---
name: aws-backup-posture-evaluation
description: Proactive AWS Backup posture review against the organization's
  backup standards. Use on a schedule or on request to evaluate whether
  in-scope resources are covered by a backup plan, whether plan frequency and
  retention meet the standard for their tier, whether cross-Region and
  cross-account copies exist where required, whether vaults use Vault Lock in
  the required mode and retention range, and whether recovery points and vaults
  are encrypted with the required customer managed key. Produces a scored
  report with ranked gaps and evidence.
---

<!-- last_verified: NOT YET TESTED against a live account. -->

# AWS Backup posture evaluation

Review backup posture for every account connected to this Agent Space. This
is a read-only review. Do not change any resource.

## Step 0: Load the standards
Read the standards from the attached `backup-standards` memory store before
anything else: tier definitions and how to find a resource's tier, required
frequency and retention per tier, required copies, required Vault Lock mode
and retention range, and encryption requirements. If the store is empty or
unreadable, report that and stop. Never invent a standard.

Work account by account. For each connected account, list the Regions where
AWS Backup is in use (backup:ListBackupVaults per Region with vaults) and
review each one.

## Step 1: Coverage
Standard: every in-scope resource is selected by an active backup plan.
1. List in-scope resources by tier, using the tags or types defined in the
   standards.
2. List backup selections (backup:ListBackupPlans, backup:ListBackupSelections)
   and the resources they cover (backup:ListProtectedResources).
3. A resource with no selection or no protected-resource entry is a coverage
   gap. Rank by tier.
If a tag-based selection returns nothing, check whether an SCP blocks
tag:GetResources before reporting the resources as unprotected.

## Step 2: Retention
Standard: frequency and retention at or above the tier's minimum.
1. For each plan rule read the schedule and DeleteAfterDays
   (backup:GetBackupPlan).
2. Retention or frequency below the tier minimum is a gap.
3. Retention above the tier maximum is not a posture gap. Record it as an
   observation for the cost review and do not score it.
4. Check the newest recovery point date for each in-scope resource
   (backup:ListRecoveryPointsByResource). A plan that exists but has produced
   no recent recovery point is a gap.

## Step 3: Cross-Region and cross-account copies
Standard: tiers that require a copy have a copy action to the required
Region or account.
1. Read the plan rules' copy actions (CopyActions in backup:GetBackupPlan).
2. Compare the destination Region and account with the standard.
3. A required copy with no copy action is a gap. A copy action whose recent
   copy jobs failed is a gap. Check with backup:ListCopyJobs.
4. Note resource types that cannot do a cross-account and cross-Region copy
   in one action (RDS, Aurora, DocumentDB, Neptune). Two copy hops are
   expected for those.

## Step 4: Vault Lock
Standard: vaults that hold the required tiers are locked in the required mode
with the required retention range.
1. Call backup:DescribeBackupVault for each vault. Read Locked, MinRetentionDays,
   MaxRetentionDays and LockDate.
2. A vault that must be locked and is not is a gap. A lock whose range does
   not match the standard is a gap.
3. For a compliance-mode lock, check whether the grace time has passed. After
   it passes, the lock cannot be changed. Report this as information so a
   person knows the setting is final.
4. Check that no plan rule's retention is outside a vault's lock range. Jobs
   to that vault would fail.

## Step 5: Encryption
Standard: vaults use a customer managed key where required, and recovery
points are encrypted.
1. Read each vault's EncryptionKeyArn. A vault on the default AWS Backup
   managed key cannot receive cross-account copies. Flag it where the
   standard requires copies.
2. Read key state and rotation (kms:DescribeKey, kms:GetKeyRotationStatus).
   A key that is not Enabled is a gap.
3. For cross-account copy destinations, check that the key policy names the
   source account.
4. Check that recovery points in required tiers are encrypted
   (backup:ListRecoveryPointsByBackupVault, EncryptionKeyArn present).

## Step 6: Score and report
Produce a score for each domain (coverage, retention, copies, Vault Lock,
encryption) as the share of in-scope resources or vaults that meet the
standard. Then list gaps, highest risk first. Rank by tier first, then by
how much of the standard is missing (for example no backup at all outranks a
retention shortfall). For each gap give the account, Region, resource or vault,
the standard it violates, the evidence (the API result), and the first
corrective step.

Output as: one artifact with the scorecard and ranked gap table, one
recommendation for each gap in the top ten, and one short notification with
the scores and the three highest-risk gaps. Say plainly which accounts or
Regions could not be reviewed and why.
