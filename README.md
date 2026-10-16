# Agenda Log SDK

Go structured logging built on Zap, with JSON output, optional rotating files,
and HTTP request trace propagation. This module is independent of the private
Agenda platform. It contains no deployment configuration or backend credentials.

## Install

Requires Go 1.25 or newer (including the optional Gin adapter).

```sh
go get github.com/FredrickUnderwood/agenda-v3-log-sdk@v0.1.0
```

```go
package main

import (
    "context"

    log "github.com/FredrickUnderwood/agenda-v3-log-sdk"
    "go.uber.org/zap"
)

func main() {
    if err := log.Init(log.Config{AppName: "example-api", Env: "test", Level: "info"}); err != nil {
        panic(err)
    }
    defer log.Shutdown()
    log.Info(context.Background(), "service started", zap.Int("port", 8080))
}
```

The default destination is stderr, suitable for Docker and host log collectors.
The application does not connect to Elasticsearch or Filebeat. Configuration
fields override their environment variable equivalents:

| Config | Environment variable | Meaning |
|---|---|---|
| `AppName` | `AGENDA_APP_NAME` | Application name, JSON field `app` |
| `Env` | `AGENDA_ENV` | Deployment environment, JSON field `env` |
| `ServiceName` | `AGENDA_SERVICE_NAME` | Container service, JSON field `service` |
| `InstanceName` | `AGENDA_INSTANCE_NAME` | Deployment instance, JSON field `instance` |
| `LogDir` | `AGENDA_LOG_DIR` | Optional additional rotating JSON file |
| `ReplicaID` | `AGENDA_REPLICA_ID` | Distinguishes per-replica log filenames |

Set `AGENDA_LOG_PER_REPLICA=1` to use the hostname as the replica suffix when
`ReplicaID` is unset. File rotation is 100 MB, five backups, up to 14 days, with
compression. These are local file settings; centralized retention is configured
on the log backend. Collect either stderr or the additional file to avoid duplicates.

## HTTP tracing

```go
handler := log.TraceMiddleware(mux) // standard net/http
client := &http.Client{Transport: log.NewTransport(nil)}
```

Use `log.Info(r.Context(), ...)` and pass the request context to outbound HTTP
requests to propagate `X-Agenda-Trace-Id` and emit `trace_id`. Gin applications
can use `ginlog.Middleware()` from the `/ginlog` subpackage. The core package
does not import Gin. A trace ID is a correlation value, never an authenticated
identity. The middleware propagates tracing only; it does not emit an access log.

## Optional browser log receiver

The `/clientlog` subpackage exposes a `net/http` handler for batched client logs:

```go
mux.Handle("POST /client-logs", clientlog.Handler(clientlog.Options{}))
```

Payload: `{"logs":[{"level":"info","msg":"page ready","fields":{"page":"home"}}]}`.
The receiver limits payload and field sizes and nests client fields under
`client`. Protect the endpoint using your application's authentication and rate
limits. Never log passwords, authorization headers, tokens, or full configuration.
The SDK does not automatically redact arbitrary user-supplied fields.

## Behavior and maintenance

- Initialize before logging; otherwise the logger is a no-op.
- Default level is `info`; configure it at startup. Invalid levels currently fall
  back to the default. Reinitializing is supported; automatic Nacos watching is not included.
- Zap production sampling applies to stderr; this is operational logging, not a
  lossless audit trail. `Shutdown` flushes the logger.
- Run `go test ./...` and `go vet ./...` before tagging a release.
- This repository receives reviewed source snapshots of the logging package.
  It does not mirror the platform repository or its Git history.

## License

The inherited GNU Affero General Public License v3 is retained in `LICENSE`.
The logging code originates from Agenda. Dependency licenses remain their own.
