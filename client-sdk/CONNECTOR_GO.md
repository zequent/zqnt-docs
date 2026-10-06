# Zequent Client SDK (Go) — Connector

`connector.New(conn)` gives your Go application asset lookup by serial number and read access to
the Skill Registry. Updating assets and their payloads is available in the Java and Python client
SDKs — see [CONNECTOR.md](CONNECTOR.md) / [CONNECTOR_PYTHON.md](CONNECTOR_PYTHON.md).

Full method-by-method reference: [Connector API Reference](../api-reference/client-sdk-connector-go.md).

Every method takes a `context.Context` and returns `(result, error)`; there's no separate
`hasErrors` flag to check the way Java/Python expose — `err` covers both transport failures and
platform-side business errors (see "Error handling" below).

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "github.com/Zequent/zqnt-client-sdk-go/v2/auth"
    "github.com/Zequent/zqnt-client-sdk-go/v2/connector"
)

// auth.DialOptions sends the client credential on every call; "" reads ZQNT_CLIENT_TOKEN.
opts := append(auth.DialOptions(""), grpc.WithTransportCredentials(insecure.NewCredentials()))
conn, _ := grpc.NewClient("localhost:8010", opts...)
c := connector.New(conn)
```

## Looking up an asset

```go
asset, err := c.GetAssetBySn(ctx, "YOUR_DEVICE_SN")
```

A client credential reaches only its own organization's assets: looking up any other serial number
is refused with `codes.PermissionDenied`.

## Capabilities

Ask what an asset supports before offering it as an option — `GetCapabilities(ctx, sn)` on the
Remote Control client returns a snapshot listing each command id with its target type and input
schema. What an asset reports comes from its edge adapter.

## Skill Registry

The Skill Registry is the platform's catalog of every command an edge adapter has reported, one
entry per `(command_id, schema_version)`, each with a lifecycle status:

```go
contracts, err := c.ListSkillContracts(ctx, nil, "") // all statuses, whole registry
for _, sc := range contracts {
    fmt.Println(sc.GetCommandId(), sc.GetSchemaVersion(), sc.GetStatus())
}
```

Changing the registry (`ObserveSkillContract`, `SetSkillContractStatus`,
`SetSkillContractPermissions`) is refused for a client credential — the registry is platform-wide.

## Error handling

A non-nil `error` covers both a platform-side error and a transport failure. A refusal of the
credential keeps its gRPC code:

```go
asset, err := c.GetAssetBySn(ctx, sn)
if status.Code(err) == codes.PermissionDenied {
    log.Printf("not one of this organization's assets: %v", err)
    return
}
if err != nil {
    // e.g. "connector: GetAssetBySn(YOUR_DEVICE_SN): <the platform's error message>"
    log.Printf("asset lookup failed: %v", err)
    return
}
```

## See also

- [Connector API Reference](../api-reference/client-sdk-connector-go.md) — every method
- [Quickstart](QUICKSTART_GO.md) — how `connector.New()` gets wired up in the first place
