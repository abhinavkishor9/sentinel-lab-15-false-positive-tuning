# Troubleshooting Notes

## `datatable()` Parser Error

The initial synthetic authentication query produced a parser error around the `datatable()` definition.

To isolate the issue, the dataset was reduced to a minimal test:

    datatable(User:string, SourceIP:string)
    [
        "user1", "10.10.10.10",
        "user2", "10.10.10.11"
    ]

The test returned successfully, confirming that `datatable()` could be used for the lab.

The full dataset was then rebuilt with a compact schema:

    datatable(TimeGenerated:datetime, User:string, SourceIP:string, ResultCode:string, App:string)

This version worked and was used for the remaining queries.

### Lesson

Start with the smallest possible `datatable()` query and confirm that it works before adding the complete dataset.

---

## Initial Detection Query

The first detection used a simple threshold:

    FailedAttempts >= 3

The query returned two candidates:

| Source IP | Failed Attempts |
|---|---:|
| `185.220.101.10` | 5 |
| `10.10.20.15` | 4 |

The administrator source `10.10.30.25` generated only two failures and was therefore not returned.

The result confirmed that the threshold was functioning, but also showed that failure count alone was not enough to distinguish the activity.

---

## Contextual Aggregation

Additional context was added to understand the behavior behind each source.

The query summarized:

- Failed authentication attempts
- Distinct targeted users
- Targeted usernames
- Application
- Source IP

The results were:

| Source IP | Failed Attempts | Targeted Users | Application |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Microsoft Office |
| `10.10.20.15` | 4 | 1 | Backup Service |
| `10.10.30.25` | 2 | 1 | Microsoft Office |

This showed that the two initial alert candidates were not following the same pattern.

---

## `union` Query Issue

An initial attempt was made to compare the initial and tuned detections using separate tabular expressions combined with `union`.

The query produced problems because the two result sets did not contain the same columns.

The initial result focused mainly on the failed-attempt count, while the tuned result also included contextual fields such as `TargetedUsers`.

### Resolution

The comparison was rewritten as a single query pipeline.

The authentication data was summarized once, and both the initial and tuned conditions were evaluated from that same result.

This produced a consistent comparison across all three sources.

### Lesson

When combining KQL result sets, make sure the branches expose compatible columns. For a small lab dataset, evaluating both conditions in one pipeline can be simpler.

---

## `No tabular expression statement found`

A later query used nested `let` statements together with `toscalar()` and `print`.

The query returned:

    No tabular expression statement found

The query was simplified instead of continuing with the nested structure.

The working structure became:

    datatable()
    | where
    | summarize
    | extend
    | summarize

The final query calculated:

| Metric | Result |
|---|---:|
| Initial Alert Candidates | 2 |
| Tuned Alert Candidates | 1 |
| Reduced Candidates | 1 |

### Lesson

When a KQL comparison becomes unnecessarily complex, return to a simple tabular pipeline and calculate the results from the same dataset.

---

## Known-Service Exclusion

The backup-service source was identified as:

    10.10.20.15

A dynamic-list approach was considered for excluding the known source, but the lab contained only one known service IP.

The condition was therefore simplified to:

    | where SourceIP != "10.10.20.15"

This returned the external multi-user source as the remaining tuned candidate.

### Lesson

For a single controlled value, direct comparison is easier to test and explain.

A production implementation would normally use a maintained allowlist, watchlist, lookup, or similar mechanism rather than hard-coding the value.

---

## False-Positive Validation

The backup-service source was tested against the tuned logic.

| Source IP | Failed Attempts | Targeted Users | Tuned Result |
|---|---:|---:|---|
| `10.10.20.15` | 4 | 1 | No |

The activity no longer satisfied the tuned criteria.

The source was excluded based on the known service behavior represented in the controlled lab scenario.

---

## Suspicious-Pattern Validation

The external source was tested against the same tuned logic.

| Source IP | Failed Attempts | Targeted Users | Tuned Result |
|---|---:|---:|---|
| `185.220.101.10` | 5 | 5 | Yes |

The multi-user authentication pattern remained detectable.

This confirmed that the tuning reduced the false-positive candidate without removing the suspicious pattern represented by the synthetic data.

---

