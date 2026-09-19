# sentinel-lab-15-false-positive-tuning
## Overview
False positive tuning is the process of improving a security detection so that it produces fewer alerts for legitimate activity while still identifying genuinely suspicious behavior.

In a SOC, an analytic rule that alerts on every occurrence of a behavior can become noisy. Analysts may repeatedly investigate the same legitimate activity, such as:

A service account repeatedly failing authentication because of an expired password
An administrator mistyping a password
An internal application repeatedly attempting authentication
A scheduled task using outdated credentials

However, the same basic behavior can also indicate an attack.

For example:

Multiple failed authentications from one external IP against many different users

may represent password spraying, while:

Multiple failed authentications from a known internal backup server against one service account

may be expected operational behavior.

The objective of this lab is therefore not simply to suppress alerts. The objective is to determine why alerts are occurring and introduce evidence-based tuning conditions.

Detection lifecycle
Raw Authentication Events
        ↓
Initial Detection
        ↓
Multiple Alerts
        ↓
Investigate Alert Patterns
        ↓
Identify Benign Activity
        ↓
Identify Suspicious Activity
        ↓
Define Tuning Conditions
        ↓
Tuned Detection
        ↓
Compare Before vs After
        ↓
Validate Detection Coverage

This lab focuses on tuning a Microsoft Sentinel authentication detection to reduce false-positive alert candidates while preserving suspicious authentication activity.

Synthetic authentication telemetry was created with KQL `datatable()` because persistent Sentinel table ingestion was not required for this exercise.

The dataset contains failed authentication events from a known backup service, an administrator workstation, and an external source targeting multiple users.

The initial detection identifies source IPs with three or more failed authentication attempts. Contextual analysis is then used to distinguish the known service pattern from the suspicious multi-user authentication pattern.

## Objectives

- Assess whether a threshold-based authentication detection is generating unnecessary alert candidates.
- Examine authentication events at the source and user level to identify differences in behavior.
- Use distinct targeted-user counts as additional detection context.
- Compare repeated failures against a single service account with failures distributed across multiple users.
- Determine which alert candidates require further investigation before classification.
- Test whether contextual conditions can reduce known benign activity without suppressing the suspicious pattern.
- Validate the tuned detection against previously identified alert candidates and non-alerting activity.
- Measure the change in alert-candidate volume after applying the tuning logic.
- Document the evidence supporting the tuning decision and the limitations of the available telemetry.
- Practice iterative detection refinement based on observed authentication behavior rather than assumptions.

## Scenario

A Microsoft Sentinel detection is generating alerts for repeated failed authentication attempts. During review, the SOC analyst notices that the alerts come from sources with different authentication patterns, making it unclear whether the same detection logic should apply to all of them.

The lab uses synthetic authentication telemetry to represent three different situations:

- A backup service generating repeated failures against the same service account.
- An administrator account generating a small number of failed attempts.
- A single external source generating failures against multiple user accounts within a short period.

The initial detection is based mainly on failed-attempt volume. This causes the backup service and the external source to become alert candidates even though their behavior is different. The analyst must investigate the underlying events and compare source, user, application, result code, timing, and the number of distinct users targeted.

The investigation shows that the backup-service activity follows a consistent single-account pattern, while the external source produces failures across multiple users. The analyst then tests whether adding behavioral context to the detection can reduce the unwanted alert candidate without removing the activity that requires further investigation.

The tuned detection is validated against the same dataset, and the results are compared with the original detection. The exercise focuses on practical false-positive tuning, evidence-based detection refinement, and validating that a reduction in alert volume does not simply come from suppressing useful security activity.

All activity in the scenario is synthetic and is intended to demonstrate Sentinel detection-tuning techniques rather than confirm a real-world compromise.


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

