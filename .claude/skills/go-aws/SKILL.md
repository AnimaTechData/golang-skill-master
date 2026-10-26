---
name: go-aws
description: Go cloud-native and AWS SDK for Go v2 guidelines — config/client reuse, smithy error handling, retries, paginators, Lambda handlers, SQS/SNS/S3/DynamoDB consumers, local endpoints and credentials. Use whenever writing or reviewing Go code that talks to AWS or runs on AWS (Lambda, ECS, EKS, EC2).
---

# Go Cloud-Native & AWS SDK v2 Guidelines

Related skills: `go-service-lifecycle` (graceful shutdown, config), `go-observability` (slog, tracing), `go-errors`, `go-concurrency`.

## 1. SDK v2 only

- Use ONLY `github.com/aws/aws-sdk-go-v2/...`. NEVER import `github.com/aws/aws-sdk-go` (v1, end-of-support). When touching v1 code, migrate it.
- Load config once with `config.LoadDefaultConfig(ctx, ...)` at startup. Let the default credential chain resolve credentials (env, SSO, IRSA/Pod Identity, ECS task role, IMDS). NEVER hardcode keys or read them from custom env vars.
- Create each service client **once** and inject it (constructor DI). Clients are safe for concurrent use. Never create clients per request/invocation.

```go
cfg, err := config.LoadDefaultConfig(ctx,
    config.WithRegion(appCfg.Region),
    config.WithRetryMaxAttempts(5),
)
if err != nil {
    return fmt.Errorf("load aws config: %w", err)
}
s3c := s3.NewFromConfig(cfg)
sqsc := sqs.NewFromConfig(cfg, func(o *sqs.Options) {
    if appCfg.SQSEndpoint != "" { // LocalStack / testcontainers
        o.BaseEndpoint = aws.String(appCfg.SQSEndpoint)
    }
})
```

- Use `o.BaseEndpoint` for local/test endpoints (for S3 against LocalStack also set `o.UsePathStyle = true`). Do NOT use the deprecated `EndpointResolver`/`EndpointResolverWithOptions`.
- Pass the caller's `ctx` to every SDK call so deadlines and cancellations propagate.
- Use `aws.String`, `aws.ToString`, etc. for pointer conversions; never dereference SDK pointer fields without nil-safe helpers.

## 2. Depend on narrow interfaces, not SDK clients

Consumers declare only the methods they call (see `go-clean-arch`). This makes fakes trivial:

```go
// in package storage (the consumer)
type objectPutter interface {
    PutObject(ctx context.Context, in *s3.PutObjectInput, optFns ...func(*s3.Options)) (*s3.PutObjectOutput, error)
}
```

Keep SDK types inside the adapter package; map to domain types at the boundary.

## 3. Error handling

- Check modeled service errors with `errors.As` against concrete types first, then fall back to `smithy.APIError`:

```go
var nsk *s3types.NoSuchKey
if errors.As(err, &nsk) {
    return domain.ErrNotFound
}
var apiErr smithy.APIError
if errors.As(err, &apiErr) {
    switch apiErr.ErrorCode() {
    case "ConditionalCheckFailedException":
        return domain.ErrConflict
    }
}
return fmt.Errorf("get object %s: %w", key, err)
```

- For HTTP-level diagnostics use `*awshttp.ResponseError` (request ID, status code). Log the request ID.
- NEVER string-match on `err.Error()`.

## 4. Retries, pagination, waiters

- The SDK retries with backoff by default (standard mode). Tune via `config.WithRetryMaxAttempts` / `retry.NewStandard`; don't wrap SDK calls in your own retry loop on top (retry storms).
- Make operations idempotent where the API allows it (client tokens, conditional writes, `attribute_not_exists`).
- ALWAYS use paginators for list/scan/query APIs — never assume one page:

```go
p := s3.NewListObjectsV2Paginator(client, &s3.ListObjectsV2Input{Bucket: aws.String(bucket)})
for p.HasMorePages() {
    page, err := p.NextPage(ctx)
    if err != nil {
        return fmt.Errorf("list objects: %w", err)
    }
    for _, obj := range page.Contents { /* ... */ }
}
```

- Use waiters (`s3.NewObjectExistsWaiter`, etc.) instead of sleep-and-poll loops.
- Batch APIs (`SendMessageBatch`, `DeleteMessageBatch`, `BatchWriteItem`) can partially fail — ALWAYS inspect `Failed` / `UnprocessedItems`.

## 5. AWS Lambda

- Use `github.com/aws/aws-lambda-go/lambda` and the `provided.al2023` runtime with a binary named `bootstrap` (`GOOS=linux GOARCH=arm64 CGO_ENABLED=0 go build -tags lambda.norpc -o bootstrap`).
- Initialize config, clients, loggers in `main` (or package init via a constructor) — NOT inside the handler. They are reused across warm invocations.
- Handlers take `ctx` first and respect its deadline (`ctx.Deadline()`).
- For SQS event sources, return `events.SQSEventResponse` with `BatchItemFailures` (enable `ReportBatchItemFailures`) instead of failing the whole batch.
- Never keep per-request state in package globals.

## 6. Messaging consumers (SQS)

- Long-poll: `WaitTimeSeconds: 20`, `MaxNumberOfMessages: 10`.
- Process with bounded concurrency (`errgroup.SetLimit`). Delete a message ONLY after successful processing.
- For long jobs, extend `VisibilityTimeout` (`ChangeMessageVisibility`) before it expires.
- Handlers must be idempotent — SQS standard queues deliver at least once. Use a DLQ with `maxReceiveCount`.
- The receive loop exits when `ctx` is cancelled; in-flight messages finish under a separate shutdown deadline (see `go-service-lifecycle`).

## 7. Cloud-native hygiene

- Configuration via environment (12-factor); secrets from Secrets Manager / SSM Parameter Store fetched at startup (and cached/refreshed), never baked into images.
- Containers: static binaries (`CGO_ENABLED=0`), distroless or scratch base, non-root user. Go 1.25+ already respects container CPU limits for `GOMAXPROCS`; set `GOMEMLIMIT` to ~80–90% of the memory limit.
- Expose health/readiness endpoints for ECS/EKS; readiness must fail during shutdown.
- Least-privilege IAM per service; never `*` actions in production policies.

## 8. Testing

- Unit tests: fake the narrow interface (section 2).
- Integration: LocalStack via testcontainers (`github.com/testcontainers/testcontainers-go/modules/localstack`) with `BaseEndpoint` override. See `go-test`.
