#### Description

`aux4/queue-cloud` is a provider plugin for `aux4/queue`. It adds `cloud` as a `--provider` choice
to every `aux4 queue` work-queue command — `create`, `delete`, `list`, `send`, `lease`, `ack`,
`nack`, `renew`, `dlq` — by delegating to `aux4/queue-sqs` (a declared dependency) with
aux4.cloud-appropriate defaults applied automatically. There is no new top-level command: once this
package is installed alongside `aux4/queue`, pass `--provider cloud` to the commands you already
use.

What this plugin adds on top of plain `--provider sqs`:

- **Env+scope-prefixed queue names** — every queue name is `<env>--<scope>--<name>` (DOUBLE dash
  separators), so two environments or two scopes can both use a queue named `jobs` without
  colliding on the same underlying SQS queue, and a single-dash scheme couldn't safely be used as
  an IAM prefix anyway: scope names may themselves contain a single `-`, so `dev-` would also
  match a scope literally named `foo-bar`. The env comes from `--env`, or the `AUX4_CLOUD_ENV`
  environment variable aux4.cloud machines set on deploy; the scope comes from `--scope`, or
  `AUX4_CLOUD_SCOPE`. Both are required; omitting either fails fast with a clear error instead of
  silently creating an unscoped/unenved queue or guessing a default. `--env`/`--scope`/`--name`
  must not themselves contain `--` (reserved as the field separator), and the resulting name —
  including the `-dlq.fifo`/`.fifo` suffix SQS adds — must not exceed AWS's 80-character SQS
  queue-name limit; both are rejected with a clear error rather than truncated.
- **No machine/VM in the name** — queues are shared across every machine running at a given
  env+scope, not scoped per-machine.
- **Region default** — `us-east-1`, overridable with `--region`. Passed to the underlying `aws`
  CLI call as `AWS_DEFAULT_REGION` for that one call only (it never overrides an explicit
  `AWS_REGION`/`AWS_DEFAULT_REGION` already set in the environment unless you pass `--region`
  yourself).
- **Credentials from the environment only** — identical to `aux4/queue-sqs`: resolved entirely by
  the AWS CLI's own standard mechanism (env vars, profile, or attached role). This package never
  stores or asks for credentials.
- **FIFO, always**, and **bounded retries by default** (`--maxAttempts` defaults to `5`, instead
  of `aux4/queue-sqs`'s own unbounded default) — both inherited/adjusted from `aux4/queue-sqs` for
  a cloud deployment, where a stuck message should dead-letter rather than retry forever.
- **`list` is supported**, scoped to the prefix — it calls `aws sqs list-queues --queue-name-prefix
  <env>--<scope>--` directly (never a bare, account-wide `ListQueues`), so it only ever returns
  queues (including dead-letter queues) created at the calling env+scope.

**Access model:** there is no per-queue access control today. Anything that can reach a given
env+scope can see and use every queue created at that env+scope; access is enforced at the
env+scope boundary (the IAM prefix below), not per queue. Per-queue access control may be added
later but is out of scope for this plugin as it stands.

#### Usage

```bash
aux4 queue create --name <name> --provider cloud --env <env> --scope <scope> [--region <region>] [--maxAttempts <n>]
aux4 queue delete --name <name> --provider cloud --env <env> --scope <scope> [--region <region>]
aux4 queue list --provider cloud --env <env> --scope <scope> [--region <region>]
aux4 queue send --name <name> --provider cloud --env <env> --scope <scope> [--group <key>]
aux4 queue lease --name <name> --provider cloud --env <env> --scope <scope> [--visibility <seconds>]
aux4 queue ack --name <name> --provider cloud --env <env> --scope <scope> --leaseId <leaseId>
aux4 queue nack --name <name> --provider cloud --env <env> --scope <scope> --leaseId <leaseId>
aux4 queue renew --name <name> --provider cloud --env <env> --scope <scope> --leaseId <leaseId> [--visibility <seconds>]
aux4 queue dlq --name <name> --provider cloud --env <env> --scope <scope>
```

--name         Queue name (final SQS name is `<env>--<scope>--<name>`; must not contain `--`, and
               the final name including `-dlq.fifo`/`.fifo` must be 80 characters or fewer)
--env          aux4.cloud environment prefix (or set `AUX4_CLOUD_ENV`); required
--scope        aux4.cloud scope prefix (or set `AUX4_CLOUD_SCOPE`); required
--region       AWS region (default: `us-east-1`)
--maxAttempts  Maximum delivery attempts before dead-lettering (default: `5`)
--group        Group key for one-at-a-time delivery; a random group is used per message when omitted
--visibility   Visibility timeout in seconds (default: 30)
--leaseId      The receipt handle returned by `queue lease` (also accepted as `--id`)

**Known limitation of calling through the top-level `aux4 queue <cmd> --provider cloud` dispatcher:**
aux4's core `queue` commands only forward flags they already know about, so `--env`/`--scope`
passed on the top-level command are silently dropped — only `AUX4_CLOUD_ENV`/`AUX4_CLOUD_SCOPE` (or
`AWS_DEFAULT_REGION` for `--region`) reach this plugin through that path. Calling the plugin's own
leaf profile directly (`aux4 queue cloud create --env ... --scope ...`, bypassing core) accepts the
flags directly, as shown in the Example below.

#### Example

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_ENDPOINT_URL=http://localhost:19781

aux4 queue cloud create --name jobs --env dev --scope myscope --maxAttempts 3
echo '{"task":"build"}' | aux4 queue cloud send --name jobs --env dev --scope myscope
aux4 queue cloud lease --name jobs --env dev --scope myscope --visibility 30
```

```text
{"id":"34b69a0a-...","leaseId":"34b69a0a-...#4a77763f-...","group":"6A286799-...","attempts":1,"body":{"task":"build"}}
```

The underlying SQS queue created above is `dev--myscope--jobs.fifo` (plus
`dev--myscope--jobs-dlq.fifo`).

#### IAM

```
arn:aws:sqs:<region>:<account-id>:<env>--<scope>--*
```
