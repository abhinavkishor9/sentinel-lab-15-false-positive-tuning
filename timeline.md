# Timeline

## Authentication Activity

| Time | Source IP | User | Activity | Investigation Context |
|---|---|---|---|---|
| 10:00 | `10.10.20.15` | `svc-backup` | Failed authentication | Backup-service activity |
| 10:01 | `10.10.20.15` | `svc-backup` | Failed authentication | Same source and account |
| 10:02 | `10.10.20.15` | `svc-backup` | Failed authentication | Repeated service pattern |
| 10:03 | `10.10.20.15` | `svc-backup` | Failed authentication | Four total failures |
| 11:00 | `10.10.30.25` | `admin1` | Failed authentication | Administrator activity |
| 11:01 | `10.10.30.25` | `admin1` | Failed authentication | Two total failures |
| 12:00 | `185.220.101.10` | `user4` | Failed authentication | Multi-user pattern begins |
| 12:01 | `185.220.101.10` | `user5` | Failed authentication | Second targeted user |
| 12:02 | `185.220.101.10` | `user6` | Failed authentication | Third targeted user |
| 12:03 | `185.220.101.10` | `user7` | Failed authentication | Fourth targeted user |
| 12:04 | `185.220.101.10` | `user8` | Failed authentication | Fifth targeted user |

## Initial Detection

The first detection used:

    FailedAttempts >= 3

The result produced two candidates:

| Source IP | Failed Attempts | Initial Result |
|---|---:|---|
| `185.220.101.10` | 5 | Alert |
| `10.10.20.15` | 4 | Alert |

The source `10.10.30.25` remained below the threshold with two failed attempts.

---

## Contextual Investigation

The alert candidates were reviewed using additional authentication context.

The analysis showed:

| Source IP | Failed Attempts | Targeted Users | Application | Pattern |
|---|---:|---:|---|---|
| `185.220.101.10` | 5 | 5 | Microsoft Office | Multiple users |
| `10.10.20.15` | 4 | 1 | Backup Service | Single service account |
| `10.10.30.25` | 2 | 1 | Microsoft Office | Low volume |

This established that the two initial candidates represented different authentication behaviors.

---

## Backup-Service Investigation

Between 10:00 and 10:03, the source `10.10.20.15` generated four failed authentication attempts.

All events involved:

    User: svc-backup
    Source: 10.10.20.15
    Application: Backup Service
    Result Code: 50126

Within the controlled scenario, the activity was treated as a false-positive candidate.

---

## Administrator Investigation

Between 11:00 and 11:01, the source `10.10.30.25` generated two failed authentication attempts.

The events involved:

    User: admin1
    Source: 10.10.30.25
    Application: Microsoft Office
    Result Code: 50126

The activity did not meet the initial three-failure threshold.

---

## External Source Investigation

Between 12:00 and 12:04, the source `185.220.101.10` generated five failed authentication attempts.

The targeted users were:

    user4
    user5
    user6
    user7
    user8

The source therefore produced five failures against five distinct users within a short period.

Within the lab, this represented simulated suspicious multi-user authentication behavior.

---

## Detection Tuning

The detection was refined with two contextual conditions:

    FailedAttempts >= 3
    TargetedUsers >= 3

The known backup-service source was also excluded:

    SourceIP != "10.10.20.15"

The final logic was:

    FailedAttempts >= 3
    AND TargetedUsers >= 3
    AND SourceIP != "10.10.20.15"

---

## Tuned Detection

After tuning, the remaining alert candidate was:

| Source IP | Failed Attempts | Targeted Users | Result |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Alert |

The backup-service source no longer satisfied the tuned criteria.

---

## Validation

The tuned logic was validated against the previously identified sources.

| Source IP | Failed Attempts | Targeted Users | Initial Detection | Tuned Detection |
|---|---:|---:|---|---|
| `185.220.101.10` | 5 | 5 | Alert | Alert |
| `10.10.20.15` | 4 | 1 | Alert | No Alert |
| `10.10.30.25` | 2 | 1 | No Alert | No Alert |

The external multi-user pattern remained detectable while the backup-service candidate was removed.

---

## Candidate Reduction

The final candidate counts were:

| Metric | Result |
|---|---:|
| Initial Candidates | 2 |
| Tuned Candidates | 1 |
| Reduced Candidates | 1 |

The synthetic dataset therefore showed a 50% reduction in alert candidates.

This percentage applies only to the controlled lab dataset.

---

## Investigation Flow

    Authentication Events
            ↓
    Initial Detection
            ↓
    2 Alert Candidates
            ↓
    Contextual Investigation
            ↓
    Backup Service + Multi-User Pattern
            ↓
    Detection Tuning
            ↓
    1 Alert Candidate
            ↓
    Suspicious Pattern Preserved

---

## Evidence Notes

The timeline is based on synthetic authentication telemetry created specifically for the lab.

The backup-service activity was treated as a known false-positive candidate within the controlled scenario.

The external source represents simulated suspicious password-spray-like behavior.

The available synthetic telemetry does not establish a confirmed real-world account compromise.

The alert reduction reflects only the candidate count within this dataset and should not be interpreted as a production false-positive rate.
