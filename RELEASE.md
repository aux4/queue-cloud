# Release Notes

## 0.1.0

- Initial release: aux4.cloud provider plugin for `aux4/queue`'s work-queue mode.
- Adds `--provider cloud` to `aux4 queue create/delete/list/send/lease/ack/nack/renew/dlq`,
  delegating to `aux4/queue-sqs` with aux4.cloud defaults: scope-prefixed queue names (from
  `--scope` or `AUX4_SCOPE`), region `us-east-1` by default, credentials from the environment only,
  FIFO queues, and a bounded `--maxAttempts` default of `5`.
- `--scope`/`AUX4_SCOPE` is required; refuses to create or address an unscoped queue.
- Note: `aux4/queue`'s own top-level dispatcher only forwards flags it explicitly declares, so
  `--scope`/`--region` only reach this plugin via `AUX4_SCOPE`/`AWS_DEFAULT_REGION`/`AWS_REGION`
  env vars when called through `aux4 queue <cmd> --provider cloud`; calling the plugin's own
  `aux4 queue cloud <cmd> --scope ...` directly accepts the flag too.
