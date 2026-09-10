# Changes to Windows11-CIS-Audit

## September 2026 - 2.3.11.6

- 2.3.11.6 asserts ForceLogoffWhenHourExpire, not LanManServer EnableForcedLogOff
- 2.3.11.6 asserted on hosts that are not domain joined only
- Collector captures ForceLogoffWhenHourExpire
- Section 1 account policy and 2.3.11.6 reported as skipped on a domain joined host, with the reason in meta.skip_reason

## 1.0.0 based on CIS Benchmark v3.0.0

- Initial release - beta, pending feedback. Please raise an issue or reach us on
  Discord with anything it gets wrong, reports unexpectedly, or misses
