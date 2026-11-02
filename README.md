# bastn connector

The bastn connector is a small agent that runs next to your internal
services and dials **out** to your [bastn](https://bastn.ai) gateway over
mutual TLS. Agents then reach those services, such as MCP servers, REST APIs
and approved commands, through the gateway, which checks every call against
your policies, approvals and audit log. Nothing listens on your network:
there are no inbound ports to open.

This repository is the public home of the connector's releases and image.
The connector is installed and enrolled from the bastn dashboard.

## Install

In the dashboard, open **Connect › Connectors › New connector**, pick a team
and a name, and mint a join token. The dashboard shows the commands below
with your token, hub address and version filled in. The token is single-use
and expires.

### Linux (amd64, arm64)

```sh
curl -fsSL https://dl.bastn.ai/install.sh | sudo BASTN_CONNECTOR_TOKEN='wpj_...' sh -s -- --hub HUB:PORT --name NAME
```

The installer:

- checks the Ed25519 signature of the release's `SHA256SUMS` (with
  OpenSSL 3) and the binary's checksum before installing anything;
- installs `/usr/local/bin/bastn-connector` and a `bastn-connector` system
  user, and keeps its identity in `/var/lib/bastn-connector`;
- enrolls the host (its private key is generated there and never leaves
  it), writes `/etc/bastn-connector/connector.yaml`, and starts a hardened
  systemd service.

The service never runs as root. A small root helper applies upgrades.

It needs `sudo`, `curl` or `wget`, and OpenSSL 3. Without systemd, it
prints the command to run the connector instead.

### Docker (amd64, arm64)

```sh
BASTN_CONNECTOR_TOKEN='wpj_...' docker run --rm -e BASTN_CONNECTOR_TOKEN \
  -v bastn-connector:/var/lib/bastn-connector ghcr.io/dflux-io/bastn-connector:latest \
  enroll --hub HUB:PORT --name NAME --state-dir /var/lib/bastn-connector

docker run -d --name bastn-connector --restart unless-stopped \
  -v bastn-connector:/var/lib/bastn-connector \
  -v "$PWD/connector.yaml:/etc/bastn-connector/connector.yaml:ro" \
  ghcr.io/dflux-io/bastn-connector:latest
```

The image runs as a non-root user and exposes no ports. Its tags are
`X.Y.Z`, `X.Y` and `latest`.

## Configure

`connector.yaml` lists the services the connector offers the gateway:

```yaml
name: NAME
hubs: HUB:PORT
state_dir: /var/lib/bastn-connector
capabilities:
  # an MCP server this machine can reach (Streamable HTTP) ...
  - name: notes
    kind: mcp_upstream
    url: http://127.0.0.1:8080/mcp
  # ... or one the connector starts itself (stdio)
  - name: files
    kind: mcp_upstream
    command: [npx, -y, "@modelcontextprotocol/server-filesystem", /srv/files]
```

After a change, run `sudo systemctl restart bastn-connector` (or
`docker restart bastn-connector`).

## Upgrade

Each connector has an update policy in the dashboard: *Manual*, *Notify* or
*Auto*.

- **Installed with `install.sh`:** use **Upgrade now** in the connector's
  panel. The connector:
  1. downloads the release through your gateway;
  2. checks it against the release key built into it;
  3. hands it to the root helper, which checks it again, keeps the previous
     binary and restarts the service. If the new version doesn't reconnect
     within two minutes, the previous one is put back.

  You can also run the installer again:

  ```sh
  curl -fsSL https://dl.bastn.ai/install.sh | sudo sh                        # the latest release
  curl -fsSL https://dl.bastn.ai/install.sh | sudo sh -s -- --version vX.Y.Z  # a given release
  ```

- **Docker:** pull a newer tag and recreate the container with the same
  volume.

## Verify a release yourself

Releases are at `https://dl.bastn.ai/connector/vX.Y.Z/`:
- `bastn-connector-linux-amd64` and `bastn-connector-linux-arm64`;
- `SHA256SUMS` and `manifest.json`, each with a `.sig` file.

A `.sig` file is the base64 of the raw Ed25519 signature over the exact
bytes of the file it signs. The current release key:

```
-----BEGIN PUBLIC KEY-----
MCowBQYDK2VwAyEA/veVg3KHajGRfel3wepW5E+DP+UTg4Aef7QbClG0CKc=
-----END PUBLIC KEY-----
```

```sh
v=vX.Y.Z; base=https://dl.bastn.ai/connector/$v
curl -fsSLO $base/SHA256SUMS -fsSLO $base/SHA256SUMS.sig -fsSLO $base/bastn-connector-linux-amd64
# save the key above as release.pem
openssl base64 -d -A -in SHA256SUMS.sig -out SHA256SUMS.bin
openssl pkeyutl -verify -pubin -inkey release.pem -rawin -in SHA256SUMS -sigfile SHA256SUMS.bin
sha256sum -c --ignore-missing SHA256SUMS
```

## Questions and issues

Open an issue in this repository, or write to support@bastn.ai. Please
report security issues privately, by email, not as an issue.
