# Zequent Client SDK (Go) — Connector

`connector.New(conn)` gives your Go application direct access to the platform's system of record —
the narrowest of the three Connector surfaces: asset lookup (read-only), scheduler management, and
technical configuration/policies. No asset payloads, organizations, asset registration, missions, or
tasks. See [CONNECTOR.md](CONNECTOR.md) / [CONNECTOR_PYTHON.md](CONNECTOR_PYTHON.md) for the
Java/Python equivalents — same backend service, same underlying data.

Full method-by-method reference: [Connector API Reference](../api-reference/client-sdk-connector-go.md).

Every method takes a `context.Context` and returns `(*Result, error)`; there's no separate
`hasErrors` flag to check the way Java/Python expose — `err` covers both transport failures and
platform-side business errors (see "Error handling" below).

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "github.com/Zequent/zqnt-client-sdk-go/connector"
)

conn, _ := grpc.NewClient("localhost:8010", grpc.WithTransportCredentials(insecure.NewCredentials()))
c := connector.New(conn)
```

## Looking up an asset

```go
asset, err := c.GetAssetBySn(ctx, "YOUR_DEVICE_SN")
```

Read-only — registration/update is normally done by an edge adapter, not a customer app.

## Capabilities

Ask what an asset supports before offering it as an option — the returned snapshot lists each
command id with its target type and input schema. What an asset reports comes from its edge adapter.

## Scheduler example

```go
scheduler := &schedulerdto.SchedulerProtoDTO{
    Name: "Nightly patrol",
    // Cron, AssetSn, etc. — see SchedulerProtoDTO for the full field set
}
created, err := c.CreateScheduler(ctx, scheduler)
```

## Error handling

Unlike the Java/Python SDKs (which surface a `hasErrors`/`error` pair on the response and leave
checking it to you), this Go package unwraps that convention internally — a non-nil `error` return
already carries the platform-side message:

```go
asset, err := c.GetAssetBySn(ctx, sn)
if err != nil {
    // err's message already includes the wire-level error text, e.g.
    // "connector: GetAssetBySn: asset not found: YOUR_DEVICE_SN"
    log.Printf("asset lookup failed: %v", err)
    return
}
```

## See also

- [Connector API Reference](../api-reference/client-sdk-connector-go.md) — every method
- [Quickstart](QUICKSTART_GO.md) — how `connector.New()` gets wired up in the first place
