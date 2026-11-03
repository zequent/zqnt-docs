# Enrolling and managing Edge Nodes (preview)

> **Preview.** Edge Nodes are new after 3.0.0 and need an installation that runs the Edge Link
> service. This page describes the preview builds and may still change.

An **Edge Node** is a device on site, such as a gateway, a companion computer, a vehicle computer
or a device a person carries, that connects assets to the platform through **Edge Link**. Device
adapters attach to the node locally. The node carries their telemetry and command events to the
platform and runs the commands the platform sends, keeping everything in a local outbox while its
link is down.

Every node has its own **identity key**. The device generates it and it never leaves the device;
the platform keeps only the public key. A node joins an organization once, with a one-time
**enrollment code**, and from then on proves who it is every time it connects. One action in the
console shuts out a lost device.

- [Before you start](#before-you-start)
- [Node classes and key protection](#node-classes-and-key-protection)
- [Enroll a node](#enroll-a-node)
- [Check that it is online](#check-that-it-is-online)
- [Bind assets](#bind-assets)
- [Suspend and reinstate](#suspend-and-reinstate)
- [A lost or stolen device](#a-lost-or-stolen-device)
- [Revoke a node](#revoke-a-node)
- [Bring a node back](#bring-a-node-back)
- [Troubleshooting](#troubleshooting)

## Before you start

### Permissions

| Permission | Allows | Built-in roles |
| --- | --- | --- |
| `edge-nodes:read` | See the Edge Nodes page, each node's status, connection, key and assets | viewer, operator, approver, org-admin |
| `edge-nodes:manage` | Create enrollment codes, rename, suspend, reinstate, revoke, report lost, bind and unbind assets | org-admin |

Binding and unbinding assets also needs `assets:edit`. A system admin can manage the nodes of every
organization and picks the organization when creating an enrollment code.

### The installation

Your platform operator sets up Edge Link once:

- **Edge Link runs** (`edge-link-service`, gRPC on port `8006`; with Docker Compose:
  `--profile edge-link`). Nodes connect to it from outside, so it needs a public address.
- **TLS terminates in Edge Link itself**, with TLS 1.3 and a certificate set in
  `QUARKUS_HTTP_SSL_CERTIFICATE_FILES` / `QUARKUS_HTTP_SSL_CERTIFICATE_KEY_FILES`. Every enrollment
  and every session is bound to the TLS connection between the node and Edge Link. A load
  balancer, ingress or reverse proxy in front of it must **pass the TCP stream through** (TLS
  passthrough). If it terminates TLS itself, Edge Link refuses every node.
- **An anchor key.** With OpenBao it is the Transit key `edge-link-anchor` (ECDSA P-256), otherwise
  `EDGE_LINK_ANCHOR_PRIVATE_KEY` (PKCS#8 P-256). Nodes pin this anchor when they enroll and trust
  only Edge Link keys signed by it, so keep it and back it up. A node enrolled under one anchor does
  not accept a replaced anchor.
- **admin-console knows Edge Link:**

  | Variable | Meaning |
  | --- | --- |
  | `EDGE_LINK_SERVICE_HOST`, `EDGE_LINK_SERVICE_PORT` (default `8006`) | Where admin-console asks Edge Link for connection status and the anchor fingerprints. Unset means the console cannot show connection status and creates codes without a provisioning file. |
  | `EDGE_LINK_PUBLIC_ENDPOINT` | `host:port` that nodes reach Edge Link at, written into provisioning files and QR codes |
  | `EDGE_LINK_PUBLIC_SERVER_NAME` | Optional: the TLS server name, when it differs from the endpoint's host |
  | `EDGE_LINK_PUBLIC_CA_FILE` | Optional: path to a PEM file with the CA certificate(s) of Edge Link's TLS certificate, when that certificate is not publicly trusted. Written into provisioning files and QR codes, so devices need no `ca_file`. admin-console does not start if the file holds no certificate. |

- **The weakest key protection accepted**, `EDGE_NODES_MINIMUM_KEY_PROTECTION`, set to the same value
  on connector and Edge Link. The default is `SOFTWARE_SEALED`. See
  [Node classes and key protection](#node-classes-and-key-protection).

### The device

The node runtime is `zqnt-edge-node`: a static Linux binary for `amd64`, `arm64` (Raspberry Pi 4/5,
Jetson with a 64-bit OS) and `armv7`, released with the
[Go SDK](https://github.com/zequent/zqnt-sdk-go#installing-an-edge-node) together with a systemd unit
and `install.sh`:

```bash
VERSION=vX.Y.Z ARCH=arm64
curl -fsSLO "https://github.com/zequent/zqnt-sdk-go/releases/download/${VERSION}/zqnt-edge-node_${VERSION}_linux_${ARCH}.tar.gz"
curl -fsSLO "https://github.com/zequent/zqnt-sdk-go/releases/download/${VERSION}/SHA256SUMS"
sha256sum -c --ignore-missing SHA256SUMS
tar -xzf "zqnt-edge-node_${VERSION}_linux_${ARCH}.tar.gz"
sudo "./zqnt-edge-node_${VERSION}_linux_${ARCH}/install.sh"
```

`install.sh` creates the system user `zqnt-edge`, installs the binary to `/usr/local/bin`, the unit
`zqnt-edge-node.service` and, if none exists yet, an example `/etc/zqnt-edge-node/node.yaml`.
Run the node commands below as `zqnt-edge`, the user the service runs as, so that the identity
files belong to it.

A TPM 2.0 is strongly recommended. Keep the device's clock synchronised (NTP).

## Node classes and key protection

The class is chosen when you create the enrollment code and **cannot be changed later**. It must
match `identity.node_class` in the device's `node.yaml`.

| Class | For | Rules |
| --- | --- | --- |
| **Infrastructure** (default) | A fixed or vehicle-mounted device | Serves several assets. Can have the roles relay and gateway. Its node credential lasts 365 days and is renewed by the node. |
| **Personal** | A device a person carries | Serves exactly one asset. No roles. Its credential lasts 24 hours, a session at most 8 hours, and the platform checks its status more often. |

**Key protection** says where the identity key lives. You choose it on the device with
`identity.key_store` in `node.yaml`:

| `key_store` | The console shows | Accepted by default |
| --- | --- | --- |
| `tpm`: generated in the TPM 2.0 and never leaves it | TPM | yes |
| `software-tpm-sealed`: a software key sealed by the TPM | Sealed software key | yes (the minimum) |
| `software-dev`: a plain key file, only with `identity.dev: true` | Development key (warning) | no |

A **development key** is for development and simulators only. A platform accepts it only when its
minimum is lowered to `SOFTWARE_DEV`. The console marks such nodes and counts them at the top of the
Edge Nodes page. The console shows key protection as the device reports it ("reported by the
device, not verified").

For `tpm` and `software-tpm-sealed`, open the TPM lines in the systemd unit
(`SupplementaryGroups=tss`, `PrivateDevices=no`, `DeviceAllow=/dev/tpmrm0 rw`).

## Enroll a node

### 1. Create an enrollment code

1. In the console: **Manage → Fleet Setup → Edge Nodes → Enrol Edge Node**.
2. Choose:
   - **Organization** (system admins only).
   - **Node class**: Infrastructure or Personal.
   - **Label**, optional, for example "rooftop gateway".
   - **Devices**: how many devices may enroll with this code, at most 10 per code. The dialog then
     says what the code is good for.
   - **Valid for**: 15 minutes, 1 hour (default), 8 hours or 24 hours.
3. **Create code.** The dialog shows:
   - the **enrollment code** (`ZQ-XXXXX-XXXXX`),
   - a **QR code** and a **Provisioning file** download (`zqnt-provisioning.json`),
   - the **platform anchor** fingerprint.

The code, the QR code and the provisioning file are **shown once**. The platform keeps only a hash
of the code. If you lose them, create a new code.

The provisioning file and the QR code carry the code, the installation id, the anchor keys, the
Edge Link endpoint and, when the installation sets `EDGE_LINK_PUBLIC_CA_FILE`, the CA certificate
of Edge Link's TLS certificate. A device that enrolls with them enrolls only with this installation
and trusts that CA for it. When the content is too large for a QR code (a long CA chain), the dialog
offers only the provisioning file. If the dialog says **No provisioning file**, admin-console could
not reach Edge Link (see [Troubleshooting](#troubleshooting)). The code alone still works.

### 2. Configure the device

Edit `/etc/zqnt-edge-node/node.yaml` (`sudoedit`):

```yaml
identity:
  key_store: tpm                # tpm | software-tpm-sealed | software-dev
  node_class: infrastructure    # infrastructure | personal: the class of the code
assets:
  - DOCK-1                      # created in the organization at enrollment, see below
links:
  - id: lte
    driver: ip-session
    endpoint: edge-link.example.com:443
    # ca_file: /etc/zqnt-edge-node/ca.pem   # when the Edge Link certificate is not publicly trusted
                                            # and you enroll without a provisioning file that carries it
```

The serials under `assets` are bound to the node at enrollment:

- A serial the platform does not know yet is **created as a new asset** of the code's organization.
- An **existing asset of the same organization** is bound as it is, as long as no other active node
  serves it (an asset of a revoked node is taken over). The code's vendor and type limits apply.
- Any other existing serial (another organization's asset, or one another active node serves) stops
  the enrollment with an error that names the serial. Nothing is created and the code is not used
  up: fix `node.yaml`, or unbind the asset from the other node, and enroll again.

A personal node lists exactly one asset.

Check the file:

```bash
sudo -u zqnt-edge zqnt-edge-node --config /etc/zqnt-edge-node/node.yaml --check-config
```

### 3. Enroll

Copy the provisioning file to the device where `zqnt-edge` can read it, then:

```bash
sudo -u zqnt-edge zqnt-edge-node enroll --config /etc/zqnt-edge-node/node.yaml \
  --provisioning /tmp/zqnt-provisioning.json
```

Or with the content of the QR code (it starts with `ZQNT-PROV1:`):

```bash
sudo -u zqnt-edge zqnt-edge-node enroll --config /etc/zqnt-edge-node/node.yaml \
  --provisioning-qr 'ZQNT-PROV1:…'
```

Or with the code alone. The node then enrolls at the `endpoint` of its `ip-session` link (or
`--platform host:port`):

```bash
sudo -u zqnt-edge zqnt-edge-node enroll --config /etc/zqnt-edge-node/node.yaml \
  --claim-code ZQ-XXXXX-XXXXX
```

`--display-name` sets the name the console shows (you can rename the node in the console later).
The device generates its key, proves it holds it, and prints:

```
enrolled in <organization>
node id:             …
organization id:     …
installation id:     …
node class:          infrastructure
key protection:      KEY_PROTECTION_TPM
identity key:        3f9a 0c41 …
platform anchor:     a9c6 451c …
```

The provisioning file is read only by `enroll`; the running service does not need it. Delete it
afterwards, because it contains the code.

### 4. Start the node and compare fingerprints

```bash
sudo systemctl enable --now zqnt-edge-node
journalctl -u zqnt-edge-node -f
```

In the console, open the node and compare:

- **Key fingerprint** with `identity key` on the device,
- the **Platform anchor** at the top of the Edge Nodes page with `platform anchor` on the device.

When you enrolled with the code alone, this comparison is how you know the device talked to your
platform; `enroll` reminds you of it. With a provisioning file or QR code, the device checks the
anchor itself and refuses a platform that does not match.

`zqnt-edge-node identity --config /etc/zqnt-edge-node/node.yaml` prints the same values later, plus
when the node credential expires.

## Check that it is online

**Manage → Fleet Setup → Edge Nodes** lists the organization's nodes and refreshes every 30 seconds:

| Column | Meaning |
| --- | --- |
| Name | Display name and node id; opens the node |
| Class | Infrastructure or Personal |
| Status | Active, Suspended or Revoked (revoked nodes only with **Show revoked**) |
| Connection | **Online** while the node has a session, **Offline** without one, **Unknown** when the platform could not ask Edge Link |
| Last seen | **Now** while online, otherwise when its last session ended, or **Never connected** |
| Key protection, Key fingerprint | The active identity key |
| Serves | Serials of the bound assets |
| Enrolled | When it enrolled |

Search by name, node id, key fingerprint or asset serial, and filter by status, class and
connection.

The node's page adds the link it is connected over and since when, the **Runtime version** (as
reported in the current session, otherwise at enrollment), the key generation, the roles and the
served assets. An asset served by a node shows **Served by Edge Node …** on its own page.

## Bind assets

On the node's page, **Served assets → Bind asset**, choose the asset, **Bind**. The node serves it at
once.

- An asset is served by **one node at a time**. If another node serves it, the dialog says
  **Served by … now** and the button becomes **Move here**: that node stops serving it.
- A **personal** node serves exactly one asset. Unbind its asset before binding another.
- **Unbind** next to an asset: the node stops serving it at once.
- A revoked node cannot get assets.

## Suspend and reinstate

**Suspend** on the node's page ends its sessions and accepts no new one until you **Reinstate** it.
Its key and assets stay as they are. Use it for maintenance or while you check something out.
Enter a reason; it is recorded in the audit log.

## A lost or stolen device

**Report lost** on the node's page. In one action, without a second approval:

- the node is **revoked**, its key ends, and its node credential is deny-listed,
- its sessions end within seconds,
- the audit log and the node record "Reported lost" with your reason.

Whoever holds the device can no longer connect with it. If it turns up, [bring it back](#bring-a-node-back)
with a re-enrollment code; never reinstate a device that was out of your hands.

## Revoke a node

**Revoke** is for a node you retire or no longer trust. It has the same effect as Report lost,
recorded as a revocation: the key ends, the credential is deny-listed and sessions end within
seconds. It **cannot be undone**: a revoked node cannot be reinstated, only re-enrolled. Revoked
nodes are hidden from the list unless you switch on **Show revoked**.

## Bring a node back

For a device that was revoked or reported lost and is back in your hands, or for replaced hardware:

1. On the node's page, **Re-enrollment code**. Choose how long it is valid; it works **once**.
2. On the device:

   ```bash
   sudo -u zqnt-edge zqnt-edge-node enroll --config /etc/zqnt-edge-node/node.yaml \
     --reenroll --claim-code ZQ-XXXXX-XXXXX
   ```

   (or `--reenroll --provisioning <file>`). Without `--reenroll` a device that is already enrolled
   refuses to enroll again.

The node comes back **Active** with the same id, class and bound assets and a **new key**. The old
key stays revoked.

To replace the key of a healthy node without the console, run
`zqnt-edge-node rotate-key --config /etc/zqnt-edge-node/node.yaml` on the device. The key generation
on the node's page goes up by one.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Connection and Last seen show **Unknown**; no platform anchor at the top of the page; codes come without a provisioning file | admin-console cannot reach Edge Link: `EDGE_LINK_SERVICE_HOST` is unset, or Edge Link does not answer admin-console within 5 seconds. The nodes themselves may be online. Check the variable and the network between admin-console and Edge Link. |
| `enroll` fails with `enrollment refused` | Edge Link does not say why, by design. Usual causes: the code is wrong, expired or used up; the key protection is below the platform's minimum (for example `software-dev` on a production platform); TLS does not terminate in Edge Link (see below). Create a new code if needed and try again. |
| `Asset … exists already and does not belong to this code's organization` | A serial under `assets` is an asset of another organization. Remove it from `node.yaml`, or enroll with a code of that organization. The code was not used up. |
| `Asset … is served by Edge Node "…"; unbind it there first` | Another active node serves this asset. Unbind it on that node's page (or revoke that node), then run `enroll` again with the same code. |
| `This code allows only … assets; asset … is not one` | The code was created for one vendor or type and the asset is another. Use a code without that limit. |
| `too many failed attempts, back off` | 10 failed attempts within 15 minutes from one address or for one code. Wait and fix the cause first. |
| `This code enrolls a … Edge Node; this node is configured as … node (node_class)` | `identity.node_class` in `node.yaml` differs from the code's class. Nothing was created and the code was not used up: set the class in `node.yaml` (or create a code of the other class) and run `enroll` again. |
| `node class …, configured …` | The same mismatch, reported by an older node runtime after the platform had accepted the enrollment: the node appears in the console. Revoke it, set the class in `node.yaml`, and enroll with a new code. |
| `platform does not match the provisioning file` | The provisioning file comes from another installation, or the installation's anchor changed. Use a file from this installation's console. |
| `node is already enrolled` | The device holds an enrollment. Use a re-enrollment code with `--reenroll`. |
| Every enrollment and connection fails although the endpoint is right | A proxy or ingress terminates TLS in front of Edge Link, or the connection is not TLS 1.3. Configure TLS passthrough so the node's TLS connection ends in Edge Link. |
| TLS certificate errors on the device | The Edge Link certificate is not trusted by the device: enroll with a provisioning file from an installation that sets `EDGE_LINK_PUBLIC_CA_FILE`, or set `ca_file` on the link (or `--ca-file`); and `server_name` (or `--server-name`) when the name in the certificate differs from the endpoint host. |
| The node stays **Offline** although the service runs | Read `journalctl -u zqnt-edge-node`. Check the status (a Suspended or Revoked node is refused), the link `endpoint`, and TLS passthrough. |
| Every connection takes an extra round trip | The device's clock is more than 2 minutes off (`EDGE_LINK_HANDSHAKE_FRESHNESS`, default `120s`). Edge Link then answers with its own time and a nonce, and the node still connects. Synchronise the clock (NTP). |
| Last seen looks old | The list refreshes every 30 seconds. Last seen is when the node's last session ended. Suspending, revoking and reporting lost take effect within seconds. Edge Link otherwise re-checks a node's status at the latest every 60 seconds (10 seconds for a personal node). |
