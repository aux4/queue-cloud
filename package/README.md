# aux4/queue-cloud

aux4.cloud provider plugin for [aux4/queue](https://hub.aux4.io/r/public/packages/aux4/queue)'s
durable work-queue mode. Installing this package adds `cloud` as a `--provider` choice to the
`aux4 queue` commands you already use — `create`, `delete`, `list`, `send`, `lease`, `ack`, `nack`,
`renew`, `dlq`. It delegates to [aux4/queue-sqs](https://hub.aux4.io/r/public/packages/aux4/queue-sqs)
(a declared dependency, installed automatically) with aux4.cloud-appropriate defaults baked in, so a
package deployed to aux4.cloud gets a durable, isolated work queue with one flag and no extra setup.

This package does not add a new top-level command. Once installed alongside `aux4/queue`, use
`aux4 queue <command> --provider cloud ...` exactly like you would with `--provider memory`,
`--provider repository`, or `--provider sqs`.

## Installation

```bash
aux4 aux4 pkger install aux4/queue-cloud
```

This also installs `aux4/queue-sqs` and `aux4/queue` (declared dependencies) if not already present.

## What it adds on top of aux4/queue-sqs

- **Env+scope-prefixed queue names** — every queue name is `<env>--<scope>--<name>` (DOUBLE dash
  separators), so two environments or two scopes can both create a queue named `jobs` without
  colliding on the same underlying SQS queue, and so an IAM prefix like `dev--*` can't accidentally
  match a differently-named scope that happens to start with `dev-` (scope names may themselves
  contain a single `-`). The env comes from `--env`, or the `AUX4_CLOUD_ENV` environment variable
  the deployed machine runs with; the scope comes from `--scope`, or `AUX4_CLOUD_SCOPE`. Both are
  **required** — `aux4/queue-cloud` refuses to create or address an unscoped queue rather than
  guess a default. The queue name itself (`--name`) must not contain `--` either, since it's
  reserved as the field separator; and the final name, including the `-dlq.fifo`/`.fifo` suffix
  SQS adds, must not exceed AWS's 80-character SQS queue-name limit — both are rejected with a
  clear error rather than silently truncated or collapsed.
- **Region default** — `us-east-1` unless overridden with `--region`.
- **Credentials from the environment only** — exactly like `aux4/queue-sqs`, never stored or
  prompted for; on a deployed machine this is whatever role/env the aux4.cloud runtime already
  injects.
- **FIFO, always** — inherited from `aux4/queue-sqs`; `--group` on `send` delivers messages for the
  same group in order and one at a time.
- **Bounded retries by default** — `--maxAttempts` defaults to `5` (not `aux4/queue-sqs`'s own
  unbounded default), since a cloud-deployed queue should dead-letter a stuck message rather than
  retry forever.
- **`queue list` is supported** — it calls `aws sqs list-queues --queue-name-prefix <env>--<scope>--`,
  so it only ever returns queues (and dead-letter queues) belonging to the calling env+scope's own
  prefix; it never performs an unfiltered, account-wide `ListQueues`.

**Access model:** there is currently no per-queue access control. Anything that can reach a given
env+scope (i.e. anything running on that scope's VM) can see and use every queue created at that
env+scope — access is enforced at the env+scope boundary (via the IAM prefix below), not per queue.
Per-queue access control is a possible future addition, not implemented today.

## Usage

```bash
aux4 queue create --name jobs --provider cloud --env dev --scope myscope --maxAttempts 5
echo '{"task":"build"}' | aux4 queue send --name jobs --provider cloud --env dev --scope myscope
aux4 queue lease --name jobs --provider cloud --env dev --scope myscope --visibility 30
# {"id":"...","leaseId":"...","group":"...","attempts":1,"body":{"task":"build"}}
aux4 queue ack --name jobs --provider cloud --env dev --scope myscope --leaseId '...'
```

The underlying SQS queues created above are `dev--myscope--jobs.fifo` and
`dev--myscope--jobs-dlq.fifo` — the same naming `aux4/queue-sqs` uses, just with the
`<env>--<scope>--` prefix applied automatically. The machine/VM a command happens to run on is
never part of the name — queues are shared across every machine in a scope, not per-machine.

## Configuration

```yaml
config:
  queue:
    provider: cloud
    env: dev             # or leave this to AUX4_CLOUD_ENV on the deployed machine
    scope: myscope        # or leave this to AUX4_CLOUD_SCOPE on the deployed machine
    region: us-east-1     # default; override per call with --region
```

## IAM

Since queue names are always `<env>--<scope>--<name>` (plus a `.fifo`/`-dlq.fifo` suffix), a
deployed machine's IAM policy can be scoped to exactly its own env+scope with a single prefix,
without risking a cross-scope match on a scope name that happens to contain a `-`:

```
arn:aws:sqs:<region>:<account-id>:<env>--<scope>--*
```

## Compatibility with aux4/queue and aux4/queue-sqs

Every flag `aux4/queue-sqs` accepts (`--group`, `--visibility`, `--leaseId`/`--id`) works the same
way here. `--owner` and `--workQueue` are accepted elsewhere in `aux4 queue` but have no effect
under `--provider cloud`, for the same reasons they have no effect under `--provider sqs` (see
aux4/queue-sqs's own README).
