# Release Notes

## 0.2.3 (unreleased)

- **Renamed package**: `aux4/queue-cloud` → `aux4/cloud-queue` (aux4's standard is the `cloud-`
  prefix, matching `aux4/cloud-file-sync` and `aux4/cloud-state-provider`). The GitHub repo moved
  to `aux4/cloud-queue` (with a redirect from the old name); the old `aux4/queue-cloud` dev-hub
  entry is now orphaned and will never be promoted to prod.
- **Dependency change**: no longer depends on `aux4/queue-sqs` — only on `aux4/queue`. The
  aux4.cloud bot deployment installs `aux4/queue-sqs` and `aux4/cloud-queue` side by side and
  configures `queue.provider: cloud`. Every `queue:cloud` operation that delegates to `aux4 queue
  sqs <op>` now checks first that `aux4/queue-sqs` is installed and fails with a clear
  `aux4 aux4 pkger install aux4/queue-sqs` message (same pattern `aux4/queue`'s own provider
  dispatcher uses) instead of a confusing "command not found".

## 0.2.2 (unreleased)

- Test fix only (no code change here): updated this package's own `lease` test to expect
  `attempts: 0` on the first delivery, matching `aux4/queue-sqs`'s fix for the `attempts`
  off-by-one (see `aux4/queue-sqs` RELEASE notes) -- `cloud-queue`'s `lease` delegates directly to
  `queue sqs lease`, so it inherited the same fix automatically once `aux4/queue-sqs` is updated.

## 0.2.0

- **BREAKING naming change**: the underlying SQS queue name is now `<env>--<scope>--<name>`
  (DOUBLE dash separators), not `<scope>-<name>`. Double dashes let an IAM policy safely prefix on
  `<env>--<scope>--*` even though scope names may contain a single `-` (a single-dash scheme would
  let a `dev-` prefix also match an unrelated scope literally named `foo-bar`). The DLQ is
  `<env>--<scope>--<name>-dlq`; `.fifo` is appended as before.
- New required `--env` (or `AUX4_CLOUD_ENV`) alongside the existing `--scope`
  (or `AUX4_CLOUD_SCOPE`, renamed from `AUX4_SCOPE`). Omitting either fails fast with a clear
  error; neither is ever guessed/defaulted.
- `--env`/`--scope`/`--name` are rejected if they contain `--` (reserved as the field separator),
  and a resulting name over 80 characters (including the `-dlq.fifo`/`.fifo` suffix) is rejected
  rather than truncated.
- The machine/VM a command runs on is never part of the queue name — queues are shared across
  every machine at a given env+scope.
- `queue list --provider cloud` is supported: it calls `aws sqs list-queues --queue-name-prefix
  <env>--<scope>--` directly, so it only returns queues belonging to the calling env+scope and
  never performs a bare, account-wide `ListQueues`. There is currently no per-queue access control
  — anything reachable at a given env+scope can see/use every queue created there; this is
  documented as the current access model, not a bug.

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
