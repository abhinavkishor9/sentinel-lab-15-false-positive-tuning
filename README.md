```markdown
# Sentinel Lab 15 — False Positive Tuning

## Overview

This lab focuses on tuning a Microsoft Sentinel authentication detection to reduce false-positive alert candidates while preserving suspicious authentication activity.

Synthetic authentication telemetry was created with KQL `datatable()` because persistent Sentinel table ingestion was not required for this exercise.

The dataset contains failed authentication events from a known backup service, an administrator workstation, and an external source targeting multiple users.

The initial detection identifies source IPs with three or more failed authentication attempts. Contextual analysis is then used to distinguish the known service pattern from the suspicious multi-user authentication pattern.

## Objectives

- Validate synthetic authentication telemetry with KQL `datatable()`
- Create an initial failed-authentication detection
- Identify alert candidates
- Add contextual authentication fields
- Investigate alert candidates individually
- Identify a false-positive candidate
- Identify suspicious multi-user authentication behavior
- Tune the detection using contextual conditions
- Validate the tuned detection
- Measure alert-candidate reduction

## Environment

- Platform: Microsoft Sentinel
- Query Language: Kusto Query Language (KQL)
- Telemetry: Synthetic authentication events
- Data generation: KQL `datatable()`
- Result code: `50126`

## Simulated Data

| Source IP | User(s) | Failed Attempts | Targeted Users | Application |
|---|---|---:|---:|---|
| `10.10.20.15` | `svc-backup` | 4 | 1 | Backup Service |
| `10.10.30.25` | `admin1` | 2 | 1 | Microsoft Office |
| `185.220.101.10` | `user4`–`user8` | 5 | 5 | Microsoft Office |

## Detection Workflow

1. Generate synthetic authentication data
2. Run the initial detection
3. Identify alert candidates
4. Add contextual analysis
5. Investigate individual sources
6. Identify the false-positive pattern
7. Identify the suspicious pattern
8. Tune the detection
9. Validate the tuning
10. Compare initial and tuned results

## Initial Detection

The initial detection used the condition:

`FailedAttempts >= 3`

The query returned two alert candidates.

| Source IP | Failed Attempts |
|---|---:|
| `185.220.101.10` | 5 |
| `10.10.20.15` | 4 |

The source `10.10.30.25` was not returned because it generated only two failed attempts.

## Contextual Investigation

Additional context was used to understand why the alert candidates were generated.

| Source IP | Failed Attempts | Targeted Users | Application |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Microsoft Office |
| `10.10.20.15` | 4 | 1 | Backup Service |
| `10.10.30.25` | 2 | 1 | Microsoft Office |

The results showed two different patterns.

The backup service generated repeated failures against one service account.

The external source generated failures against five different users.

## False-Positive Candidate

The source `10.10.20.15` generated four failed authentication events involving:

- User: `svc-backup`
- Application: `Backup Service`
- Result code: `50126`
- Targeted users: 1

Within this controlled scenario, the activity was treated as a false-positive candidate representing known service behavior.

## Suspicious Pattern

The source `185.220.101.10` generated five failed authentication attempts against five different users:

- `user4`
- `user5`
- `user6`
- `user7`
- `user8`

The events occurred in consecutive minutes.

Within the lab, this pattern represents simulated password-spray-like authentication behavior.

The synthetic dataset does not establish a confirmed real-world compromise because it does not contain independent evidence such as successful authentication, endpoint execution, persistence, or other compromise indicators.

## Detection Tuning

The tuned detection uses:

`FailedAttempts >= 3`

`TargetedUsers >= 3`

`SourceIP != "10.10.20.15"`

The tuning adds behavioral context instead of simply increasing the failed-attempt threshold.

## Validation

| Source IP | Failed Attempts | Targeted Users | Initial Detection | Tuned Detection |
|---|---:|---:|---|---|
| `185.220.101.10` | 5 | 5 | Alert | Alert |
| `10.10.20.15` | 4 | 1 | Alert | No Alert |
| `10.10.30.25` | 2 | 1 | No Alert | No Alert |

The tuned detection removed the known backup-service candidate while preserving the simulated suspicious multi-user pattern.

## Alert Reduction

| Metric | Result |
|---|---:|
| Initial alert candidates | 2 |
| Tuned alert candidates | 1 |
| Reduced candidates | 1 |

The candidate count was reduced by 50% within this synthetic dataset.

This percentage is specific to the lab dataset and should not be treated as a production detection-performance measurement.

## MITRE ATT&CK Mapping

### T1110.003 — Password Spraying

The external-source activity represents simulated repeated authentication failures against multiple users from one source.

This mapping describes the behavior represented by the lab telemetry and does not establish that a real password-spraying attack occurred.

## Key Lessons

- Failed-authentication volume alone is not sufficient to classify activity.
- Context helps distinguish service behavior from suspicious authentication patterns.
- Distinct targeted users provide useful detection context.
- Known service activity should be validated before being excluded.
- Detection tuning should reduce unnecessary alert candidates without removing the suspicious pattern being investigated.
- Synthetic telemetry demonstrates detection logic but does not reproduce the visibility of a production environment.
- An indicator or threshold is not, by itself, proof of compromise.

## Final Outcome

The initial detection generated two alert candidates.

Investigation identified the backup-service source as a false-positive candidate within the controlled scenario and identified the external source as simulated suspicious multi-user authentication behavior.

After tuning, the backup-service candidate no longer met the detection criteria while the suspicious multi-user pattern remained detectable.
```
