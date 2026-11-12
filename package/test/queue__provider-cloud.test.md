# aux4 queue work queue (cloud provider)

These tests run against [ElasticMQ](https://github.com/softwaremill/elasticmq), an SQS-API-compatible
server, via `AWS_ENDPOINT_URL` -- the same local stand-in `aux4/queue-sqs`'s own tests use. No real
AWS account or network access is used.

Requires `aux4/queue-cloud` installed alongside `aux4/queue-sqs` and `aux4/queue`, the AWS CLI, and
an ElasticMQ server reachable at `http://localhost:19781`. In CI this is started as a job-level
`services:` container (see `.github/workflows/publish.yml`); to run locally, start it yourself
first:

```bash
docker run -d --name queue-cloud-test-elasticmq -p 19781:9324 softwaremill/elasticmq-native
```

````beforeAll
curl -sf http://localhost:19781 >/dev/null || (echo "ElasticMQ is not reachable on localhost:19781 -- start it first, see the top of this file" && exit 1)
````

````beforeEach
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
export AUX4_SCOPE=cloudtest
aux4 queue create --name wq --provider cloud --maxAttempts 2 >/dev/null
````

````afterEach
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
export AUX4_SCOPE=cloudtest
aux4 queue delete --name wq --provider cloud >/dev/null 2>&1
````

### should scope-prefix the queue name with AUX4_SCOPE

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
aws sqs get-queue-url --queue-name cloudtest-wq.fifo --output json | jq -r '.QueueUrl' | grep -o 'cloudtest-wq.fifo'
````

````expect
cloudtest-wq.fifo
````

### should refuse to operate on an unscoped queue

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
unset AUX4_SCOPE
aux4 queue list --provider cloud
echo "exit=$?"
````

````expect:partial
Error: --scope is required (or set AUX4_SCOPE) *?
exit=1
````

### should lease a sent message through the top-level --provider cloud dispatcher

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
export AUX4_SCOPE=cloudtest
echo '{"task":"hello"}' | aux4 queue send --name wq --provider cloud >/dev/null
aux4 queue lease --name wq --provider cloud --visibility 30 | jq '{attempts, body}'
````

````expect:json
{
  "attempts": 1,
  "body": {
    "task": "hello"
  }
}
````

### should nack for redelivery and eventually dead-letter, same as the underlying sqs provider

````timeout
15000
````

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=elasticmq
export AWS_ENDPOINT_URL=http://localhost:19781
export AUX4_SCOPE=cloudtest
echo '{"task":"doomed"}' | aux4 queue send --name wq --provider cloud >/dev/null
LEASE1=$(aux4 queue lease --name wq --provider cloud --visibility 1)
LID1=$(echo "$LEASE1" | jq -r '.leaseId')
aux4 queue nack --name wq --provider cloud --leaseId "$LID1" >/dev/null
LEASE2=$(aux4 queue lease --name wq --provider cloud --visibility 1)
LID2=$(echo "$LEASE2" | jq -r '.leaseId')
aux4 queue nack --name wq --provider cloud --leaseId "$LID2" >/dev/null
aux4 queue lease --name wq --provider cloud --visibility 1
echo "main-queue-exit=$?"
aux4 queue dlq --name wq --provider cloud | jq '.body'
````

````expect:partial
{}
main-queue-exit=4
{
  "task": "doomed"
}
````

### should use the default --region (us-east-1) when none is given

````execute
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_ENDPOINT_URL=http://localhost:19781
export AUX4_SCOPE=cloudtest
aux4 queue cloud list --scope cloudtest 2>&1 | head -1
echo "exit=$?"
````

````expect:partial
exit=0
````
