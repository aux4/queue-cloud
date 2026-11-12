#### Description

`aux4/queue-cloud` is a provider plugin for `aux4/queue`. It adds `cloud` as a `--provider` choice
to every `aux4 queue` work-queue command — `create`, `delete`, `list`, `send`, `lease`, `ack`,
`nack`, `renew`, `dlq` — by delegating to `aux4/queue-sqs` (a declared dependency) with
aux4.cloud-appropriate defaults applied automatically. There is no new top-level command: once this
package is installed alongside `aux4/queue`, pass `--provider cloud` to the commands you already
use.

What this plugin adds on top of plain `--provider sqs`:

- **Scope-prefixed queue names** — every queue name is prefixed with `<scope>-`, so two scopes
  can both use a queue named `jobs` without colliding on the same underlying SQS queue. The scope
  comes from `--scope`, or from the `AUX4_SCOPE` environment variable. **There is no documented
  aux4.cloud-wide "current scope" runtime variable today** — this package follows the same
  `--scope`/`AUX4_SCOPE` convention the rest of the aux4 CLI uses (e.g. `aux4/cloud`'s own
  `AUX4_CLOUD_SCOPE`), noted here as an assumption for whoever wires this into an actual deployed
  machine. `--scope`/`AUX4_SCOPE` is required; omitting both fails fast with a clear error instead
  of silently creating an unscoped queue.
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

#### Usage

```bash
aux4 queue create --name <name> --provider cloud --scope <scope> [--region <region>] [--maxAttempts <n>]
aux4 queue delete --name <name> --provider cloud --scope <scope> [--region <region>]
aux4 queue list --provider cloud --scope <scope> [--region <region>]
aux4 queue send --name <name> --provider cloud --scope <scope> [--group <key>]
aux4 queue lease --name <name> --provider cloud --scope <scope> [--visibility <seconds>]
aux4 queue ack --name <name> --provider cloud --scope <scope> --leaseId <leaseId>
aux4 queue nack --name <name> --provider cloud --scope <scope> --leaseId <leaseId>
aux4 queue renew --name <name> --provider cloud --scope <scope> --leaseId <leaseId> [--visibility <seconds>]
aux4 queue dlq --name <name> --provider cloud --scope <scope>
```

--name         Queue name (prefixed with `<scope>-` before being passed to `aux4/queue-sqs`)
--scope        aux4.cloud scope prefix (or set `AUX4_SCOPE`); required
--region       AWS region (default: `us-east-1`)
--maxAttempts  Maximum delivery attempts before dead-lettering (default: `5`)
--group        Group key for one-at-a-time delivery; a random group is used per message when omitted
--visibility   Visibility timeout in seconds (default: 30)
--leaseId      The receipt handle returned by `queue lease` (also accepted as `--id`)

#### Example

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_ENDPOINT_URL=http://localhost:19781

aux4 queue create --name jobs --provider cloud --scope myscope --maxAttempts 3
echo '{"task":"build"}' | aux4 queue send --name jobs --provider cloud --scope myscope
aux4 queue lease --name jobs --provider cloud --scope myscope --visibility 30
```

```text
{"id":"34b69a0a-...","leaseId":"34b69a0a-...#4a77763f-...","group":"6A286799-...","attempts":1,"body":{"task":"build"}}
```

The underlying SQS queue created above is `myscope-jobs.fifo` (plus `myscope-jobs-dlq.fifo`).
