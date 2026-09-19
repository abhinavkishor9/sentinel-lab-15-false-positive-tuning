```markdown
# Investigation Notes — Sentinel Lab 15

## Investigation Focus

The investigation focused on a failed-authentication detection that generated multiple alert candidates.

The goal was to determine whether the candidates represented similar activity or different behavioral patterns and then tune the detection using the observed context.

## Synthetic Dataset

The lab used KQL `datatable()` to create authentication telemetry.

The final synthetic dataset contained 11 failed authentication events.

| Source IP | User(s) | Events | Targeted Users | Application | Result Code |
|---|---|---:|---:|---|---|
| `10.10.20.15` | `svc-backup` | 4 | 1 | Backup Service | `50126` |
| `10.10.30.25` | `admin1` | 2 | 1 | Microsoft Office | `50126` |
| `185.220.101.10` | `user4`–`user8` | 5 | 5 | Microsoft Office | `50126` |

## Initial Detection

The initial detection used:

`FailedAttempts >= 3`

The result was:

| Source IP | Failed Attempts |
|---|---:|
| `185.220.101.10` | 5 |
| `10.10.20.15` | 4 |

Two alert candidates were generated.

The source `10.10.30.25` was not returned because it generated only two failures.

## Context Query

The next step was to aggregate the authentication activity by source.

The analysis included:

- Failed-attempt count
- Distinct targeted-user count
- Targeted usernames
- Application context

The result was:

| Source IP | Failed Attempts | Targeted Users | Application |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Microsoft Office |
| `10.10.20.15` | 4 | 1 | Backup Service |
| `10.10.30.25` | 2 | 1 | Microsoft Office |

This showed that the two alert candidates had different behavioral characteristics.

## Source Investigation — Backup Service

The source `10.10.20.15` generated four events.

| Time | User | Source IP | Result Code | Application |
|---|---|---|---|---|
| 10:00 | `svc-backup` | `10.10.20.15` | `50126` | Backup Service |
| 10:01 | `svc-backup` | `10.10.20.15` | `50126` | Backup Service |
| 10:02 | `svc-backup` | `10.10.20.15` | `50126` | Backup Service |
| 10:03 | `svc-backup` | `10.10.20.15` | `50126` | Backup Service |

The four events followed the same pattern:

- Same source
- Same service account
- Same application
- Same result code
- One targeted user

Within the controlled lab scenario, this was treated as a false-positive candidate representing expected backup-service behavior.

## Source Investigation — Administrator

The source `10.10.30.25` generated two events.

| Time | User | Source IP | Result Code | Application |
|---|---|---|---|---|
| 11:00 | `admin1` | `10.10.30.25` | `50126` | Microsoft Office |
| 11:01 | `admin1` | `10.10.30.25` | `50126` | Microsoft Office |

The source targeted one user and generated two failures.

It did not meet the initial threshold and therefore was not an alert candidate.

## Source Investigation — External Source

The source `185.220.101.10` generated five events.

| Time | User | Source IP | Result Code | Application |
|---|---|---|---|---|
| 12:00 | `user4` | `185.220.101.10` | `50126` | Microsoft Office |
| 12:01 | `user5` | `185.220.101.10` | `50126` | Microsoft Office |
| 12:02 | `user6` | `185.220.101.10` | `50126` | Microsoft Office |
| 12:03 | `user7` | `185.220.101.10` | `50126` | Microsoft Office |
| 12:04 | `user8` | `185.220.101.10` | `50126` | Microsoft Office |

The important characteristics were:

- Five failed authentication attempts
- Five distinct targeted users
- One source IP
- Consecutive-minute activity
- Same application
- Same result code

Within the lab, this pattern was treated as simulated suspicious multi-user authentication behavior.

The evidence is not sufficient to classify it as a confirmed compromise because the dataset is synthetic and contains no independent compromise telemetry.

## Tuning Decision

The initial threshold was considered too dependent on failure volume.

The tuning introduced a second behavioral condition:

`TargetedUsers >= 3`

The known backup-service source was also excluded:

`SourceIP != "10.10.20.15"`

The resulting logic was:

`FailedAttempts >= 3`

`AND TargetedUsers >= 3`

`AND SourceIP != "10.10.20.15"`

## Tuned Detection Result

The tuned detection returned:

| Source IP | Failed Attempts | Targeted Users | Result |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Alert |

The backup-service source was no longer returned.

## False-Positive Validation

The backup-service source was checked against the tuned conditions.

| Source IP | Failed Attempts | Targeted Users | Meets Tuned Criteria |
|---|---:|---:|---|
| `10.10.20.15` | 4 | 1 | No |

The source was successfully removed from the tuned alert candidate set.

## Suspicious-Pattern Validation

The external source was checked against the same conditions.

| Source IP | Failed Attempts | Targeted Users | Meets Tuned Criteria |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Yes |

The suspicious multi-user authentication pattern remained detectable.

## Initial vs Tuned Results

| Source IP | Failed Attempts | Targeted Users | Initial Detection | Tuned Detection |
|---|---:|---:|---|---|
| `185.220.101.10` | 5 | 5 | Alert | Alert |
| `10.10.20.15` | 4 | 1 | Alert | No Alert |
| `10.10.30.25` | 2 | 1 | No Alert | No Alert |

## Candidate Reduction

| Metric | Result |
|---|---:|
| Initial candidates | 2 |
| Tuned candidates | 1 |
| Reduced candidates | 1 |

The tuned detection reduced the synthetic candidate count by 50%.

This result is limited to the lab dataset.

## Evidence Assessment

### Confirmed

- `10.10.20.15` generated four failed authentication events.
- `185.220.101.10` generated five failed authentication events.
- `185.220.101.10` targeted five distinct users.
- The initial detection generated two candidates.
- The tuned detection generated one candidate.
- The backup-service candidate no longer met the tuned criteria.
- The external multi-user pattern continued to meet the tuned criteria.

### Simulated

- The backup-service activity represents known internal service behavior within the lab.
- The external source represents suspicious password-spray-like behavior.

### Unknown

- Whether any credentials were valid
- Whether any authentication succeeded
- Whether any account was compromised
- Whether the source IP represents an actual malicious infrastructure
- Whether similar patterns would occur at the same frequency in production

## Investigation Conclusion

The investigation showed that the two initial candidates were not behaviorally equivalent.

The backup service produced repeated failures against one known service account, while the external source produced failures against multiple distinct users.

The tuned detection incorporated targeted-user count and a known-service exclusion. This removed the identified false-positive candidate while preserving the suspicious multi-user pattern in the synthetic dataset.
```
