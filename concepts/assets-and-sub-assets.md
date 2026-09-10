# Assets & Sub-Assets

This page defines the two entities every Zequent telemetry frame is described in terms of, and
shows real, fully-populated telemetry for each of the supported configurations.

## What is the difference between an Asset and a SubAsset?

**An Asset is a registered top-level entity. A SubAsset is an optional child entity associated
with an Asset.**

A drone can therefore be either:

- a **top-level Asset**, when it is operated independently, or
- a **SubAsset**, when it belongs to another Asset such as a Dock.

Both are valid, fully supported configurations. Which one you have depends on how the device was
registered, not on what kind of device it is.

## Asset is not a synonym for "Dock"

An `Asset` is the primary, registered physical or logical entity. It can be a drone, a dock, a
ground vehicle, a sensor gateway, a station, a camera, or any other device your adapter registers.
The platform's own `AssetTypeEnum` makes this explicit:

```
ASSET_TYPE_AIRCRAFT   ASSET_TYPE_DOCK     ASSET_TYPE_SENSOR
ASSET_TYPE_CAMERA     ASSET_TYPE_JAMMER   ASSET_TYPE_SAPIENT
ASSET_TYPE_RNS        ASSET_TYPE_OTHER    ASSET_TYPE_UNKNOWN
```

`ASSET_TYPE_AIRCRAFT` is a first-class Asset type. A drone registered directly as an Asset is a
normal, supported deployment — not a degenerate case.

A `SubAsset` is **optional**. In the platform's own data model the parent carries
`optional SubAssetProtoDTO sub_asset_dto` — an Asset with no SubAsset is the default, not an
incomplete record. An Asset and a SubAsset are described by the same field shape (both carry their
own `sn`, `name`, `type`, `vendor`, `model`, `organization`, `online`), which is why the same
device model can appear in either position.

## The two supported configurations

### 1. Standalone Asset

The device is registered directly as an Asset and has no parent.

```
Asset  (ASSET_TYPE_AIRCRAFT, sn = SIM-DRONE-001)
```

Telemetry for it uses:

| Field | Value |
| --- | --- |
| `sourceType` | `ASSET` |
| `asset` | populated |
| `subAsset` | `null` |

### 2. Hierarchical Asset → SubAsset

A Dock/Station is the Asset; a Drone is a SubAsset belonging to it.

```
Asset  (ASSET_TYPE_DOCK, sn = ZQT-DOCK-0417)
└── SubAsset  (ASSET_TYPE_AIRCRAFT, sn = ZQT-DRONE-1123)
```

This configuration produces **two independent telemetry streams**, one describing the dock and one
describing the drone:

| Describing | `sourceType` | `asset` | `subAsset` |
| --- | --- | --- | --- |
| the Dock | `ASSET` | populated | `null` |
| the Drone | `SUB_ASSET` | `null` | populated |

## How `sourceType` is decided

`sourceType` is **derived, not set by hand**. It follows directly from which detail object is
populated:

- `sourceType = ASSET` — the telemetry originates from or describes the **Asset itself**.
  `asset` is populated, `subAsset` is `null`.
- `sourceType = SUB_ASSET` — the telemetry describes a **child SubAsset**.
  `subAsset` is populated, `asset` is `null`.
- `sourceType = UNSPECIFIED` — neither or both are set. This is invalid; the Edge SDK's
  `validate()` rejects it with `"Exactly one telemetry source must be provided"`.

`asset` and `subAsset` describe the **source/context** of the telemetry — which entity the reading
is about. They do not imply a device category.

## Worked examples

The three frames below are complete, realistic telemetry payloads. Field names use the JSON/Java
form (`absoluteAltitude`); the Python Edge SDK uses the snake_case equivalent
(`absolute_altitude`). Exact timestamp rendering depends on your JSON mapper configuration — the
epoch-with-nanoseconds form shown here is what the default platform serializer produces.

Note that `batteryInformation.percentage` and `batteryInformation.returnToHomePower` are
**strings**, not numbers, in this contract.

### Example 1 — Standalone drone registered as an Asset

`SIM-DRONE-001` is a MAVLink drone registered directly as an Asset. It has no parent Dock, so
`subAsset` is `null` and `sourceType` is `ASSET`. This frame is a correct, complete
representation — not a partially-filled one.

```json
{
  "tid": "8f14e45f-ceea-467a-9575-9f2a1b0c3d4e",
  "eventType": "TELEMETRY",
  "hasErrors": false,
  "sn": "SIM-DRONE-001",
  "assetId": "c1d7f3a2-95b4-4c1e-8f6d-2a7b9e0c4513",
  "timestamp": 1789024317.385502435,
  "telemetry": {
    "id": "SIM-DRONE-001",
    "sn": "SIM-DRONE-001",
    "timestamp": 1789024317.385502435,
    "latitude": 52.52000045776367,
    "longitude": 13.404999732971191,
    "absoluteAltitude": 34.0,
    "relativeAltitude": 34.0,
    "windSpeed": 3.305775,
    "heading": 0.0,
    "asset": {
      "mode": "ASSET_MODE_IDLE",
      "subAssetPercentage": 100.0,
      "hasActiveManualControlSession": false,
      "positionValid": true,
      "positionState": {
        "gpsNumber": 14,
        "rtkNumber": 0,
        "quality": 5
      },
      "networkInformation": {
        "type": "NETWORK_TYPE_4_G",
        "rate": 12.4,
        "quality": "NETWORK_STATE_QUALITY_GOOD"
      },
      "manualControlState": "MANUAL_CONTROL_STATE_DISCONNECTED",
      "debugModeOpen": false,
      "environmentTemp": 21.5,
      "humidity": 48.0,
      "rainfall": "RAINFALL_NO",
      "subAssetInformation": null,
      "subAssetAtHome": null,
      "subAssetCharging": null,
      "insideTemp": null,
      "coverState": null,
      "workingVoltage": null,
      "workingCurrent": null,
      "supplyVoltage": null,
      "airConditioner": null,
      "wirelessLink": null,
      "sdrState": null
    },
    "subAsset": null,
    "sourceType": "ASSET"
  },
  "error": null
}
```

The `null` fields in `asset` above are not missing data. `coverState`, `airConditioner`,
`workingVoltage`, `insideTemp`, `subAssetAtHome` and similar describe **dock enclosure hardware**
that a standalone drone does not have. `AssetDetails` is a superset covering every kind of Asset;
each device populates the fields that physically apply to it. A standalone drone reporting
`coverState: null` is correct behaviour.

### Example 2 — Dock registered as an Asset

`ZQT-DOCK-0417` is a dock with a paired drone. This frame describes **the dock itself**, so
`sourceType` is `ASSET` — exactly as in Example 1, even though the device is a completely
different kind of thing. Every `AssetDetails` field is populated here because a dock genuinely has
all of this hardware.

```json
{
  "tid": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "eventType": "TELEMETRY",
  "hasErrors": false,
  "sn": "ZQT-DOCK-0417",
  "assetId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1789024322.104881000,
  "telemetry": {
    "id": "ZQT-DOCK-0417",
    "sn": "ZQT-DOCK-0417",
    "timestamp": 1789024322.104881000,
    "latitude": 41.015137,
    "longitude": 28.979530,
    "absoluteAltitude": 38.2,
    "relativeAltitude": 0.0,
    "windSpeed": 4.1,
    "heading": 275.0,
    "asset": {
      "environmentTemp": 24.8,
      "insideTemp": 27.3,
      "humidity": 52.0,
      "mode": "ASSET_MODE_WORKING",
      "rainfall": "RAINFALL_NO",
      "subAssetInformation": {
        "sn": "ZQT-DRONE-1123",
        "model": "Matrice 30T",
        "paired": true,
        "online": true
      },
      "subAssetAtHome": false,
      "subAssetCharging": false,
      "subAssetPercentage": 78.0,
      "debugModeOpen": false,
      "hasActiveManualControlSession": true,
      "coverState": "COVER_STATE_OPENED",
      "workingVoltage": 23940,
      "workingCurrent": 1420,
      "supplyVoltage": 24000,
      "positionValid": true,
      "networkInformation": {
        "type": "NETWORK_TYPE_ETHERNET",
        "rate": 94.6,
        "quality": "NETWORK_STATE_QUALITY_EXCELLENT"
      },
      "airConditioner": {
        "state": "AIR_CONDITIONER_COOL",
        "switchTime": 312
      },
      "manualControlState": "MANUAL_CONTROL_STATE_CONNECTED",
      "positionState": {
        "gpsNumber": 18,
        "rtkNumber": 12,
        "quality": 5
      },
      "wirelessLink": {
        "fourthGenerationFreqBand": 2.4,
        "fourthGenerationGndQuality": 4,
        "fourthGenerationLinkState": true,
        "fourthGenerationQuality": 4,
        "fourthGenerationUavQuality": 3,
        "dongleNumber": 1,
        "linkWorkmode": "SDR",
        "sdrFreqBand": 5.8,
        "sdrLinkState": true,
        "sdrQuality": 5
      },
      "sdrState": {
        "downQuality": 5,
        "upQuality": 4,
        "frequencyBand": 5.8
      }
    },
    "subAsset": null,
    "sourceType": "ASSET"
  },
  "error": null
}
```

Examples 1 and 2 are **both** `sourceType = ASSET`. That is the point: `ASSET` means "this reading
is about the registered top-level entity", whether that entity is a drone or a dock.

### Example 3 — Drone reported as a SubAsset of that Dock

`ZQT-DRONE-1123` is the drone belonging to `ZQT-DOCK-0417`. Because the reading describes the
**child** entity, `sourceType` is `SUB_ASSET`, `subAsset` is populated and `asset` is `null`.

Note that `sn` at the frame level is the **parent Asset's** serial number, while
`telemetry.id` / `telemetry.sn` identify the SubAsset the reading is actually about.

```json
{
  "tid": "9c8b7a65-4d3e-2f1a-0b9c-8d7e6f5a4b3c",
  "eventType": "TELEMETRY",
  "hasErrors": false,
  "sn": "ZQT-DOCK-0417",
  "assetId": "550e8400-e29b-41d4-a716-446655440000",
  "timestamp": 1789024327.551200000,
  "telemetry": {
    "id": "ZQT-DRONE-1123",
    "sn": "ZQT-DRONE-1123",
    "timestamp": 1789024327.551200000,
    "latitude": 41.015137,
    "longitude": 28.979530,
    "absoluteAltitude": 42.6,
    "relativeAltitude": 12.4,
    "windSpeed": 4.6,
    "heading": 187.5,
    "asset": null,
    "subAsset": {
      "horizontalSpeed": 2.8,
      "verticalSpeed": 1.1,
      "windDirection": "NORTH_WEST",
      "gear": 1,
      "mode": "SUBASSET_MODE_TAKEOFF_AUTO",
      "country": "TR",
      "heightLimit": 120,
      "homeDistance": 18.4,
      "totalMovementDistance": 1284.75,
      "totalMovementTime": 412.5,
      "batteryInformation": {
        "percentage": "78",
        "remainingTime": 1320,
        "returnToHomePower": "22"
      },
      "payloadTelemetry": {
        "id": "ZQT-DRONE-1123-PAYLOAD-1",
        "name": "H20T",
        "timestamp": 1789024327.551200000,
        "cameraData": {
          "currentLens": "zoom",
          "gimbalPitch": -32.5,
          "gimbalYaw": 187.2,
          "gimbalRoll": 0.0,
          "zoomFactor": 4.0
        },
        "rangeFinderData": {
          "targetLatitude": 41.015982,
          "targetLongitude": 28.980614,
          "targetDistance": 96.3,
          "targetAltitude": 12.8
        },
        "sensorData": {
          "targetTemperature": 31.4
        }
      }
    },
    "sourceType": "SUB_ASSET"
  },
  "error": null
}
```

## Quick reference

| Question | Answer |
| --- | --- |
| Is an Asset always a Dock? | No. An Asset is any registered top-level entity — drone, dock, vehicle, sensor gateway, camera, station. |
| Is a SubAsset always a Drone? | No. A SubAsset is any child entity belonging to an Asset. |
| Must every Asset have a SubAsset? | No. `subAsset` is optional; standalone Assets are a normal configuration. |
| Can a drone be an Asset? | Yes — when it is operated independently, as in Example 1. |
| Can a drone be a SubAsset? | Yes — when it belongs to a parent Asset such as a Dock, as in Example 3. |
| Can `asset` and `subAsset` both be set? | No. Exactly one, per frame. Both or neither is rejected as `UNSPECIFIED`. |
| What decides `sourceType`? | Which detail object is populated. It is derived, never set by hand. |

## See also

- [Java Edge SDK — Live Data](../edge-sdk/edge-sdk-live-data.md) — publishing telemetry from an adapter
- [Java Edge SDK — Models Reference](../edge-sdk/edge-sdk-models.md#telemetrydata) — the complete `TelemetryData` field list
- [Python Edge SDK — Live Data](../edge-sdk/edge-sdk-python-live-data.md) — the `AssetTelemetry` / `SubAssetTelemetry` dataclasses
- [Java Client SDK — Streaming responses](../client-sdk/FUNCTIONAL_RESPONSES.md) — consuming telemetry in a customer application
