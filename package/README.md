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

- **Scope-prefixed queue names** — every queue name is prefixed with the calling machine's
  aux4.cloud scope (`<scope>-<name>`), so two scopes can both create a queue named `jobs` without
  colliding on the same underlying SQS queue. The scope comes from `--scope`, or from the
  `AUX4_SCOPE` environment variable the deployed machine runs with (there is no cloud-wide
  "current scope" API today — this package reads the same env var/flag convention as the rest of
  the aux4 CLI; see the man page for the fallback note). `--scope`/`AUX4_SCOPE` is **required** —
  `aux4/queue-cloud` refuses to create or address an unscoped queue.
- **Region default** — `us-east-1` unless overridden with `--region`.
- **Credentials from the environment only** — exactly like `aux4/queue-sqs`, never stored or
  prompted for; on a deployed machine this is whatever role/env the aux4.cloud runtime already
  injects.
- **FIFO, always** — inherited from `aux4/queue-sqs`; `--group` on `send` delivers messages for the
  same group in order and one at a time.
- **Bounded retries by default** — `--maxAttempts` defaults to `5` (not `aux4/queue-sqs`'s own
  unbounded default), since a cloud-deployed queue should dead-letter a stuck message rather than
  retry forever.

## Usage

```bash
aux4 queue create --name jobs --provider cloud --scope myscope --maxAttempts 5
echo '{"task":"build"}' | aux4 queue send --name jobs --provider cloud --scope myscope
aux4 queue lease --name jobs --provider cloud --scope myscope --visibility 30
# {"id":"...","leaseId":"...","group":"...","attempts":1,"body":{"task":"build"}}
aux4 queue ack --name jobs --provider cloud --scope myscope --leaseId '...'
```

The underlying SQS queues created above are `myscope-jobs.fifo` and `myscope-jobs-dlq.fifo` — the
same naming aux4/queue-sqs uses, just with the scope prefix applied automatically.

## Configuration

```yaml
config:
  queue:
    provider: cloud
    scope: myscope      # or leave this to AUX4_SCOPE on the deployed machine
    region: us-east-1    # default; override per call with --region
```

## Compatibility with aux4/queue and aux4/queue-sqs

Every flag `aux4/queue-sqs` accepts (`--group`, `--visibility`, `--leaseId`/`--id`) works the same
way here. `--owner` and `--workQueue` are accepted elsewhere in `aux4 queue` but have no effect
under `--provider cloud`, for the same reasons they have no effect under `--provider sqs` (see
aux4/queue-sqs's own README).
