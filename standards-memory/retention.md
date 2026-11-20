# Retention standard (example)

| Tier | Minimum frequency | Minimum retention | Maximum retention |
|---|---|---|---|
| tier-1 | Daily | 35 days | 365 days |
| tier-2 | Daily | 14 days | 90 days |
| tier-3 | Weekly | 7 days | 35 days |

- Retention below the minimum is a posture gap.
- Retention above the maximum is a cost observation. It is not a posture gap.
- Continuous backups (point-in-time recovery) are limited to 35 days by the service. Treat 35 days as met for tier-1.
- Warm storage before a cold storage transition: at least 8 days. Cold storage is allowed only for tier-1 and tier-2 and only when total retention is 90 days or more beyond the transition.
