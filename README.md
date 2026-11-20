# aws-backup-devops-agent-skills

Version-controlled AWS Backup expertise for AWS DevOps Agent. Push this repository to GitHub, associate the GitHub account with your Agent Space, and import each Skill directory with **Knowledge > Skills > Add skill > Import from repository**. After you change a Skill in Git, open it in the Agent Space and choose **Sync**.

This repository is Part 2 of the AWS Backup and AWS DevOps Agent series. Part 1 ([Automate root cause analysis for AWS Backup failures with AWS DevOps Agent](https://aws.amazon.com/blogs/storage/automate-root-cause-analysis-for-aws-backup-failures-with-aws-devops-agent/)) builds the event pipeline that sends failed jobs to the agent.

## Contents

| Path | What it is | Where it goes in the Agent Space |
|---|---|---|
| `aws-backup-failure-investigation/` | Investigation playbook for failed backup and copy jobs. Five branches plus reference files. | Skill. Agent types Incident RCA and On-demand. |
| `aws-backup-posture-evaluation/` | Scheduled review of coverage, retention, cross-Region copies, Vault Lock and encryption against your standards. | Skill. Evaluation agent type. Attached to the custom agent. |
| `aws-backup-cost-optimization/` | Scheduled review of spend you can remove. | Skill. Evaluation agent type. Attached to the custom agent. |
| `standards-memory/` | Example content for the `backup-standards` memory store. | Custom memory store, attached to the custom agent. |
| `custom-agent/backup-posture-reviewer.md` | Configuration and system prompt for the scheduled agent. | Custom agent with a schedule trigger. |
| `directed-actions/elevated-role.yaml` | Optional CloudFormation template for the elevated role that lets the agent carry out an approved fix. | One role per investigated account. |

## Skill inventory

Keep this table current. Two Skills that cover the same alarm or symptom can conflict or duplicate work.

| Skill | Agent type | Covers | Does not cover |
|---|---|---|---|
| `aws-backup-failure-investigation` | Incident RCA, On-demand | Backup and copy jobs in FAILED, ABORTED, EXPIRED. Branches: A job could not start or finish, B backup role, C KMS key, D cross-account or cross-Region copy, E Vault Lock retention | Restore job failures, malware scan failures, recovery point indexing failures |
| `aws-backup-posture-evaluation` | Evaluation | Coverage, retention below standard, copies, Vault Lock, encryption | Over-retention (cost Skill) |
| `aws-backup-cost-optimization` | Evaluation | Retention above standard, cold storage, redundant copies, expired and orphaned recovery points, unused vaults | Under-retention (posture Skill) |

## Division of knowledge

| Knowledge | Where it lives | Why |
|---|---|---|
| Procedures: how to investigate, how to review | Skills (this repository) | Durable, version controlled, loaded when relevant |
| Standards: tiers, retention, copies, lock, encryption | `backup-standards` memory store | Changes with policy, without a Skill edit |
| Corrections to a wrong investigation | `reflections` memory store (managed) | Written from your feedback, read by later investigations of the same alarm |
| Rules for every session | Agent instructions (AGENTS.md) | Always loaded. Keep it short. |

## Review process
- Open a pull request for every change to a Skill. The diff is the review.
- Review each Skill quarterly. Update the `last_verified` comment after a test against a test account.
- Re-run `docs/fault-injection-plan.md` after any change to a Skill.
