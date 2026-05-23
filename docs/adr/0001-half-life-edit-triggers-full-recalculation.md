# Half-life edit triggers full ActiveDoseSnapshot recalculation

When a user edits a Supplement's `half_life_days`, the server recalculates all ActiveDoseSnapshots for that Supplement from the first Injection Log forward. A confirmation warning is shown in the UI before the edit is submitted, because the recalculation is immediate and irreversible. Partial recalculation (from edit date only) was rejected — half-life affects the entire decay history, so a split history would be silently wrong. Stale snapshots were rejected for the same reason.
