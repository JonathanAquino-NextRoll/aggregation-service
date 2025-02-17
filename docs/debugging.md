# Debug aggregation runs with encrypted payloads

This document describes the debugging support for the Aggregation Service running in a TEE using
encrypted payloads of aggregatable reports. This allows you to debug your production setup and
understand how the encrypted payloads of aggregatable reports are processed. Reports with
debug_cleartext_payload can be used with the [local testing tool](/docs/local-testing-tool.md) and
are helpful for understanding the content of reports and validating that registrations on the
browser client or device are configured properly.

To test the Aggregation Service, you can enable debug aggregation runs which use encrypted payloads
of aggregatable reports to generate debug summary reports. When executing a debug run, no noise is
added to the debug summary report, and annotations are added to indicate whether keys are present in
domain input and/or reports. This allows developers to:

-   Analyze the reports
-   Determine if the aggregation was completed correctly, per the adtech's specifications
-   Understand the impact of noise in summary reports
-   Determine how to set the proper domain keys

Additionally, debug runs do not enforce the
[No-Duplicates rule](https://github.com/WICG/attribution-reporting-api/blob/main/AGGREGATION_SERVICE_TEE.md#no-duplicates-rule)
across batches. The No-Duplicates rule is still enforced within a batch. This allows adtech to try
different batches without worrying about making them
[disjoint](https://github.com/WICG/attribution-reporting-api/blob/main/AGGREGATION_SERVICE_TEE.md#disjoint-batches)
during testing or debugging.

Once third-party cookies are deprecated, the client (browser or operating system) will no longer
generate aggregatable reports that are enabled for debugging. At that time, debug runs with
encrypted payloads will no longer be supported for reports from real user devices or browsers.

In this document, you'll find code snippets and instructions for how to debug the Aggregation
Service and create debug summary reports.

## Create a debug job

To create an aggregation debug job, add the `debug_run` parameter to the `job_parameters` object of
the `createJob` API request.

`POST https://<frontend_api_id>.execute-api.us-east-1.amazonaws.com/stage/v1alpha/createJob`

```json
{
    "input_data_blob_prefix": "input/reports.avro",
    "input_data_bucket_name": "<data_bucket>",
    "output_data_blob_prefix": "output/summary_report.avro",
    "output_data_bucket_name": "<data_bucket>",
    "job_parameters": {
        "attribution_report_to": "<your_attribution_domain>",
        "output_domain_blob_prefix": "domain/domain.avro",
        "output_domain_bucket_name": "<data_bucket>",
        "debug_run": "true"
    },
    "Job_request_id": "test01"
}
```

If `debug_run` is not present in`job_parameters` or it's set to `false`, a normal noised aggregation
run is created. More details about `createJob` API can be found in
[detailed API spec](/docs/api.md#createjob-endpoint).

## Debuggable aggregatable reports

A debug run only considers reports that have the flag `"debug_mode": "enabled"` in the report
shared_info ([aggregatable report sample](/docs/collecting.md#aggregatable-report-sample)). Reports
with the `debug_mode` flag missing or the `debug_mode` value isn't set to `enabled` aren't included
in the results generated by a debug run.

The count of reports that were not processed during a debug run is returned in the job response,
which can be previewed in the [detailed API spec](/docs/api.md#createjob-endpoint). In the
`error_counts` field, the category `NUM_REPORTS_DEBUG_NOT_ENABLED` shows the numbers of reports not
processed during a debug run.

## Results

Two summary reports are generated from a debug run: a regular summary report and a debug summary
report. The regular summary report format generated from the debug run is consistent with that of a
regular aggregation run. The debug summary report has a [different format](#debug-summary-report).
The path of the summary report is set in the [createJob](/docs/api.md#createjob-endpoint) API. The
debug summary report will be stored in the "debug" folder under the summary report's path with the
same object name.

Considering the following createJob parameters for `output_data_bucket_name` and
`output_data_blob_prefix`:

```json
{
    "output_data_blob_prefix": "output/summary_report.avro",
    "output_data_bucket_name": "<data_bucket>"
}
```

the following objects are created by a debug run:

`s3://<data_bucket>/output/summary_report.avro` and

`s3://<data_bucket>/output/debug/summary_report.avro`.

Note that the regular summary report generated during a debug run will only include reports which
have the flag `"debug_mode": "enabled"` in the reports `shared_info`.

### Debug summary report

The debug summary report includes the following data:

-   `bucket`: The aggregation key
-   `unnoised_metric`: The aggregation value without noise
-   `noise`: The approximate noise applied to the aggregated results in the regular summary report
-   `annotations`: The annotations associated with the bucket

The keys in the debug summary report will include all the keys from the output domain.

If the key is only present in the output domain (not in any of the processed aggregatable reports),
the key will be included in the debug report with `unnoised_metric=0` and
`annotations=["in_domain"]`.

The keys that are only present in aggregatable reports (not in output domain) will also be included
in the debug report with `unnoised_metric=<unnoised aggregated value>` and
`annotations=["in_reports"]`.

Keys that are present in domain and aggregatable reports will have annotations for both
`["in_domain", "in_reports"]`.

The schema of debug summary reports is in the following [Avro](https://avro.apache.org/) format:

```avro
{
"type": "record",
"name": "DebugAggregatedFact",
"fields": [
    {
    "name": "bucket",
    "type": "bytes",
    "doc": "Histogram bucket used in aggregation. 128-bit integer encoded as a 16-byte big-endian bytestring. Leading 0-bits will be left out."
    },
    {
    "name": "unnoised_metric",
    "type": "long",
    "doc": "Unnoised metric associated with the bucket."
    },
    {
    "name": "noise",
    "type": "long",
    "doc": "The noise applied to the metric in the regular result."
    }
    {
    "name":"annotations",
    "type":
       {
       "type": "array",
       "items": {
         "type":"enum",
         "name":"bucket_tags",
         "symbols":["in_domain","in_reports"]
       }
    }
  ]
}
```

### Privacy Budget Service Errors

When aggregation is performed in debug mode, the privacy budget is still consumed. If the budget is
available and consumed successfully, the return code and message in the case of success are the same
as in a non-debug run.

Otherwise, a summary report will still be generated. A custom return code and a corresponding return
message will be present in the getJob API response indicating if budget errors would occur for this
aggregation in a non-debug run.

#### Debug Mode Aggregation Outcomes

**Success:** Returns code `SUCCESS` | Aggregation job successfully processed.

**Success with aggregation errors:** Returns code `SUCCESS_WITH_ERRORS` | Aggregation job
successfully processed but some reports have errors.

**PBS Exception:** Returns code `DEBUG_SUCCESS_WITH_PRIVACY_BUDGET_ERROR` | Aggregation would have
failed in non-debug mode due to a privacy budget error.

**PBS Exhausted:** Returns code `DEBUG_SUCCESS_WITH_PRIVACY_BUDGET_EXHAUSTED` | Aggregation would
have failed in non-debug mode due to privacy budget exhaustion.
