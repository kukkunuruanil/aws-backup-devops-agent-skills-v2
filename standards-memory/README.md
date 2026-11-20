# Backup standards memory store

These files are the content of the custom memory store `backup-standards`. A memory store holds facts the agent reads when they apply. The posture and cost Skills read it so the standard can change without editing a Skill.

Create the store, then one memory per file. Memory names are `/`-separated lowercase kebab-case paths, so the folders below are the name prefixes. Write each description so the agent can decide from it alone whether to open the memory.

## Create with the AWS CLI

```bash
# 1. The store (agent_types GENERIC makes it visible to every agent type)
aws devops-agent create-asset \
  --agent-space-id <AGENT_SPACE_ID> \
  --asset-type memory_store \
  --metadata '{"name":"backup-standards","description":"Backup standards for this organization: tier definitions, required backup frequency and retention, cross-Region and cross-account copy requirements, Vault Lock mode and retention range, and encryption requirements. Read before any backup posture or cost review.","agent_types":["GENERIC"]}' \
  --content '{"file":{"path":"README.md","body":{"text":"Backup standards. One memory per domain."}}}' \
  --region <REGION>

# 2. One memory per file (repeat for each file below)
aws devops-agent create-asset \
  --agent-space-id <AGENT_SPACE_ID> \
  --asset-type memory \
  --metadata '{"name":"standards/retention","description":"Required backup frequency and minimum and maximum retention for each tier.","memory_store_id":"<MEMORY_STORE_ASSET_ID>"}' \
  --content '{"file":{"path":"retention.md","body":{"text":"<CONTENTS OF retention.md, JSON-escaped>"}}}' \
  --region <REGION>
```

You can also create the store and memories in the console, or ask Chat: "In the backup-standards store, remember that tier 1 resources need daily backups kept for at least 35 days."

## Files
| Memory name | File |
|---|---|
| `standards/tiers` | `tiers.md` |
| `standards/coverage` | `coverage.md` |
| `standards/retention` | `retention.md` |
| `standards/copies` | `copies.md` |
| `standards/vault-lock` | `vault-lock.md` |
| `standards/encryption` | `encryption.md` |

The values are examples. Replace them with your own standards before use.
