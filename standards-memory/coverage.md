# Coverage standard (example)

- Every in-scope resource in tier-1 and tier-2 must be selected by an active backup plan.
- Tier-3 resources should be covered. An uncovered tier-3 resource is reported as low risk.
- A resource counts as covered only if it also has a recovery point newer than the tier's backup frequency plus one day.
- Plans that select by tag must select on `backup-tier`. A plan that selects by resource type alone is reported as an observation.
