# Metric uses numeric type with optional min/max instead of a scale_1_10 type

The `scale_1_10` data type from the original design is removed. Metrics are either `numeric` or `boolean`. Numeric Metrics gain optional `min` and `max` fields on `MetricDefinition` to constrain input and drive UI rendering (a segmented control or bounded number input). This supports arbitrary scales (1–5, 0–100, etc.) without multiplying type enums. The `scale_1_10` type was rejected because the distinction was purely a UI hint — it stored in the same column as `numeric` and carried no additional semantic meaning.
