# Vault Lock standard (example)

| Vault role | Required lock | Retention range |
|---|---|---|
| Archive vault (`archive-vault`, in the archive account) | Compliance mode, grace time passed | Minimum 35 days, maximum 365 days |
| Primary vaults holding tier-1 | Governance mode | Minimum 35 days, maximum 365 days |
| Primary vaults holding tier-2 or tier-3 | Not required | n/a |

- A compliance-mode lock whose grace time has not passed is reported as "not yet final".
- Plan retention must fall inside the vault's lock range. A plan outside it is reported as a gap, because its jobs will fail.
- Never recommend removing a compliance-mode lock. It cannot be removed after the grace time.
