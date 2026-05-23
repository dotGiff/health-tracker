# Health Tracker

A multi-user app for tracking supplement injection protocols and arbitrary health metrics over time.

## Language

**Supplement**:
A user-registered injectable substance with a configurable name, half-life, unit, and color. Represents anything the user injects and wants to track active dose for — peptides, hormones, medications, etc.
_Avoid_: peptide, compound, drug, medication, injectable

**Dose Log**:
A single recorded dose of a Supplement on a given date, including amount and optional notes. Route-agnostic — covers injections, oral, sublingual, or any other administration method.
_Avoid_: InjectionEntry, InjectionLog, injection entry, injection log, DoseEntry, record

**Active Dose**:
The estimated total amount of a Supplement currently remaining in the body, calculated via continuous exponential decay from all prior Injection Logs. Expressed in the Supplement's unit (mg, mcg, IU).
_Avoid_: concentration, in-system concentration, plasma concentration

**Active Dose Snapshot**:
A pre-computed Active Dose value stored at the moment of each Dose Log. Recalculated server-side at write time so the frontend never runs decay math. One row per Dose Log.
_Avoid_: ConcentrationSnapshot, concentration snapshot

**Metric**:
A user-defined health measurement category (e.g. "Weight", "Waist", "Energy Level"). Has a data type of `numeric` (with optional min/max) or `boolean`. Created once; Metric Logs are recorded against it indefinitely.
_Avoid_: MetricDefinition, tracker, health metric

**Metric Log**:
A single recorded value for a Metric on a given date. One row per Metric per date.
_Avoid_: MetricEntry, metric entry, record
