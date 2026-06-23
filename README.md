<div align="center">
  <img width="350" alt="image" src="https://github.com/user-attachments/assets/01a8b1d2-83db-4610-8e49-3c98e101e9de" />
</div>

## ▷|◁ Acrity Connector

The **Acrity Connector** is the on-premises agent for **Acrity** (Adversarial Code
Review). It runs inside your own network and lets Acrity's cloud review pipeline reach
your **self-hosted** version control system — GitLab, GitHub Enterprise, Bitbucket Data
Center, or Azure DevOps Server — **without exposing it to the internet**.

The connector makes only **outbound** connections to Acrity over a secure WebSocket
tunnel. Your VCS tokens and tunnel secrets stay **on your infrastructure** and are never
sent to the Acrity cloud.

> **This repository hosts the public distribution artifacts only** (container image,
> native binaries, checksums, and the auto-update feed). It is not the source code.

---

## What you get here

| Channel | Where | Notes |
|---|---|---|
| **Container image** | `ghcr.io/tech-driven-technology/acrity-connector` | Public, multi-arch (`linux/amd64` + `linux/arm64`) |
| **Native binaries** | [Releases](https://github.com/tech-driven-technology/acrity-connector/releases) | Single-file, self-contained — Linux / Windows / macOS, x64 + arm64 |
| **Checksums** | `SHA256SUMS` (per release) | Verify every download |
| **Auto-update feed** | `releases.<rid>.json` (per release) | Powers the native in-app updater (Velopack) |

> **Easiest path:** generate a ready-to-run artifact from the **Acrity Console**
> (your connector's screen → *Download*). It comes pre-filled with your connector ID,
> workspace, broker URL, and a freshly minted token. Everything below is the manual
> equivalent of what the Console generates for you.

---

## How it works

<img width="1671" height="941" alt="image" src="https://github.com/user-attachments/assets/b6505a98-92ff-440d-a706-6be2ef0f1d68" />

The connector receives review requests over the tunnel, talks to your VCS using **your**
credentials (which never leave your network), and streams results back. No inbound ports
are opened.

---

## Install

### Option A — Docker / Docker Compose

The image is public and multi-arch, so no registry login is needed:

```bash
docker pull ghcr.io/tech-driven-technology/acrity-connector:latest
# or pin a version:
docker pull ghcr.io/tech-driven-technology/acrity-connector:0.0.3
```

Minimal `docker-compose.yml` (the Console generates a complete one with your values):

```yaml
services:
  acrity-connector:
    image: ghcr.io/tech-driven-technology/acrity-connector:latest
    container_name: acrity-connector
    restart: unless-stopped
    environment:
      BROKER_WSS_URL: "wss://<your-acrity-broker>/ws/connector"
      CONNECTOR_ID:   "<from the Console>"
      WORKSPACE_ID:   "<from the Console>"
      PROVIDER_KIND:  "gitlab"            # gitlab | github | bitbucket | azuredevops
      CONNECTOR_AUTH_TOKEN: "${CONNECTOR_AUTH_TOKEN}"  # minted in the Console
      VCS_BASE_URL:   "${VCS_BASE_URL}"   # your internal VCS endpoint
      VCS_TOKEN:      "${VCS_TOKEN}"      # your VCS access token (stays here)
```

### Option B — Kubernetes (Helm)

The Console generates a minimal Helm chart whose `values.yaml` points at the public image:

```yaml
image:
  repository: ghcr.io/tech-driven-technology/acrity-connector
  tag: latest
  pullPolicy: IfNotPresent
```

Cloud-side secrets are supplied at install time and never committed:

```bash
helm install acrity-connector ./acrity-connector \
  --set secrets.connectorAuthToken=<minted-in-console> \
  --set secrets.relaySecret=<minted-in-console>
```

### Option C — Native binary (no container)

Download the binary for your platform from the
[latest release](https://github.com/tech-driven-technology/acrity-connector/releases/latest):

| OS | x64 | arm64 |
|---|---|---|
| **Linux** | `acrity-connector-linux-x64` | `acrity-connector-linux-arm64` |
| **Windows** | `acrity-connector-win-x64.exe` | `acrity-connector-win-arm64.exe` |
| **macOS** | `acrity-connector-osx-x64` | `acrity-connector-osx-arm64` |

```bash
# Linux/macOS example
curl -fsSLO https://github.com/tech-driven-technology/acrity-connector/releases/latest/download/acrity-connector-linux-x64
curl -fsSLO https://github.com/tech-driven-technology/acrity-connector/releases/latest/download/SHA256SUMS
sha256sum --ignore-missing -c SHA256SUMS
chmod +x acrity-connector-linux-x64
./acrity-connector-linux-x64 --version
```

The native build supports running as a managed service (systemd on Linux, launchd on
macOS, Windows Service) and updates itself in place via the bundled Velopack feed when run
standalone. The Console's "native" artifact ships install scripts and a service unit for
each platform.

---

## Configuration

The connector reads configuration from environment variables (or a `connector.env` file,
mode `0600`, read **before** the process environment). The essentials:

| Variable | Source | Purpose |
|---|---|---|
| `BROKER_WSS_URL` | Console | Acrity broker tunnel endpoint |
| `CONNECTOR_ID` / `WORKSPACE_ID` | Console | Identifies this connector |
| `PROVIDER_KIND` | Console | `gitlab` / `github` / `bitbucket` / `azuredevops` |
| `CONNECTOR_AUTH_TOKEN` | **Minted in Console** | Authenticates the connector to Acrity |
| `RELAY_SECRET` / `RELAY_SECRET_KID` | **Minted in Console** | Signs relayed webhook traffic |
| `VCS_BASE_URL` | You | Your internal VCS base URL |
| `VCS_TOKEN` | You | Your VCS access token (**never leaves your network**) |
| `VCS_WEBHOOK_SECRET` | You | Validates inbound webhooks from your VCS |

The cloud-side values (`CONNECTOR_AUTH_TOKEN`, `RELAY_SECRET`, `RELAY_SECRET_KID`) are
produced by the **mint** step in the Acrity Console and pasted in once. The VCS values are
yours and stay local.

Optional networking / TLS knobs:

| Variable | Default | Purpose |
|---|---|---|
| `CONNECTOR_CA_BUNDLE` | *(unset)* | Path to a PEM with extra CA(s) to trust — for a private-CA / self-signed VCS or a TLS-inspecting proxy |
| `CONNECTOR_TLS_INSECURE_SKIP_VERIFY` | `false` | `true` skips **all** VCS TLS validation — **dev/local only**; prefer `CONNECTOR_CA_BUNDLE` |
| `HTTPS_PROXY` / `HTTP_PROXY` / `NO_PROXY` | *(unset)* | Route the connector's outbound traffic through a forward proxy |

### TLS for a self-hosted / self-signed VCS

Self-managed VCS instances often present a certificate signed by a **private CA** or a
**self-signed** certificate (and corporate networks may re-sign egress via a TLS-inspecting
proxy such as Zscaler). The connector verifies VCS TLS by default, so an untrusted chain
makes it report `validation_status: network_error` — the tunnel and heartbeat are fine, but
the connector shows as **degraded** in the Console until it can validate your VCS.

**1. Trust the CA (recommended).** Mount a PEM bundle and point `CONNECTOR_CA_BUNDLE` at it.
This *extends* the OS trust store; it never disables hostname/identity checks, so the
certificate's SAN must still cover the host in `VCS_BASE_URL`.

```yaml
# docker-compose
environment:
  CONNECTOR_CA_BUNDLE: "/etc/acrity/ca-bundle.pem"
volumes:
  - ./ca-bundle.pem:/etc/acrity/ca-bundle.pem:ro
```

Grab the certificate with, for example,
`openssl s_client -connect <vcs-host>:443 -showcerts </dev/null | openssl x509 > ca-bundle.pem`.

**2. Skip verification (dev/local only).** Set `CONNECTOR_TLS_INSECURE_SKIP_VERIFY=true` to
accept **any** VCS certificate — both chain and hostname checks are skipped. The connector
logs a loud warning at startup. The generated artifacts ship this line **commented out**;
uncomment it to opt in. Never use it in production — prefer the CA bundle above.

---

## Verifying downloads

Every release ships a `SHA256SUMS` manifest. When GPG signing is enabled by the publisher,
a detached `SHA256SUMS.sig` is included as well:

```bash
sha256sum --ignore-missing -c SHA256SUMS
# optional, if SHA256SUMS.sig is present:
gpg --verify SHA256SUMS.sig SHA256SUMS
```

---

## Versioning

Releases are tagged `connector-v<semver>` (e.g. `connector-v0.0.3`). The container image is
published with both a mutable `:latest` tag and an immutable `:<version>` tag — pin
`:<version>` if you need reproducible deployments.

---

## Security model

- **Egress-only.** The connector opens no inbound ports; it dials out to the Acrity broker.
- **Secrets stay local.** Your VCS token and webhook secret live only on your host/cluster.
- **TLS on by default.** VCS connections are verified; trust a private CA via
  `CONNECTOR_CA_BUNDLE`. The insecure skip (`CONNECTOR_TLS_INSECURE_SKIP_VERIFY`) is an
  explicit, warned, dev-only opt-in and ships disabled.
- **Least privilege.** Give the VCS token only the scopes the connector needs (read diffs/
  files, post review comments).

---

## Support

Open an issue in this repository, or contact your Acrity administrator.
