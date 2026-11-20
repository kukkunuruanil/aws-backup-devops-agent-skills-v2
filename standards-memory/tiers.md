# Backup tiers (example)

Every resource belongs to one tier. Find the tier from the resource tag `backup-tier`. A resource with no tag is treated as **tier 3** and reported as "untiered".

| Tier | Meaning | Examples |
|---|---|---|
| tier-1 | Business critical. Loss is a reportable incident. | Production databases, customer data stores |
| tier-2 | Important. Loss causes a significant outage. | Internal platforms, shared file systems |
| tier-3 | Recoverable. Can be rebuilt. | Development and test, build caches |

In scope: Amazon RDS, Aurora, DynamoDB, Amazon EBS, Amazon EFS, Amazon S3 and Amazon FSx resources in all connected accounts and in the Regions listed in `standards/copies`. Out of scope: resources tagged `backup-exempt=true`.
