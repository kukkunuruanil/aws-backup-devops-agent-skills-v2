# Cross-Region and cross-account copy standard (example)

| Tier | Cross-Region copy | Cross-account copy |
|---|---|---|
| tier-1 | Required, to a second Region from the list below | Required, to the backup archive account |
| tier-2 | Required | Not required |
| tier-3 | Not required | Not required |

- Approved secondary Regions: `us-west-2` for `us-east-1` sources, `us-east-1` for `us-west-2` sources.
- Backup archive account: `<ARCHIVE_ACCOUNT_ID>`. Copies go to the vault `archive-vault` in the same Region as the source.
- For RDS, Aurora, Amazon DocumentDB and Amazon Neptune, a cross-account copy and a cross-Region copy are two separate copy actions. Both are expected for tier-1.
- A copy action whose last three copy jobs failed counts as not met.
