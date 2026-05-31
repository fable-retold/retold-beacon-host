# Quick Start

This guide gets Retold Beacon Host running with one or more provider packages and registered against an Ultravisor.

## Prerequisites

- Node.js 18+ (matches the rest of the Retold ecosystem).
- A running [Ultravisor](https://stevenvelozo.github.io/ultravisor/) server you can reach over HTTP.
- At least one `CapabilityProvider` package published to npm (or available in your local `node_modules` tree). See [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) for how those are authored.

## Installation

Install the host together with the provider packages you want to serve. Providers are loaded with `require()` from the same `node_modules` tree as the host, so they must be installed beside it.

```bash
npm install retold-beacon-host
npm install retold-beacon-provider-meadow-integration
```

For container deployments, install the host and providers during the image build, then run the host as the entrypoint (see [Container Deployment](quickstart.md) below).

## Run With a Single Provider

The minimum required flags are a local `--port`, an `--ultravisor-url`, and one `--provider`:

```bash
retold-beacon-host \
	--port 54400 \
	--ultravisor-url http://localhost:54321 \
	--provider retold-beacon-provider-meadow-integration
```

On a successful start you will see log lines similar to:

```
[retold-beacon-host] Loaded provider 'retold-beacon-provider-meadow-integration' (MeadowIntegration / DataAccess)
[retold-beacon-host] listening on port 54400
[retold-beacon-host] beacon registered as 'host' with http://localhost:54321
```

If `--beacon-name` is not supplied, the beacon registers under the default name `host`.

## Run With Multiple Providers

Repeat `--provider` to load several capability packages into the same beacon and Orator server. Give the beacon a stable name with `--beacon-name`:

```bash
retold-beacon-host \
	--port 54400 \
	--beacon-name data-node-1 \
	--ultravisor-url http://localhost:54321 \
	--provider retold-beacon-provider-meadow-integration \
	--provider retold-beacon-provider-shell
```

Each provider registers its own capability with the beacon. The host loads providers in the order the flags appear.

## Supplying Provider Configuration

A single JSON file can be passed to every provider constructor with `--config`. The host reads the file once, `JSON.parse`s it, and hands the resulting object unchanged to each provider; each provider pulls only the keys it cares about.

```bash
retold-beacon-host \
	--port 54400 \
	--ultravisor-url http://localhost:54321 \
	--provider retold-beacon-provider-meadow-integration \
	--config /app/data/config.json
```

The shape of that JSON is defined by the providers you load, not by the host. If the file cannot be read or parsed, the host logs a warning and continues with an empty config object -- so a provider that requires configuration may then fail in its constructor (which stops the host; see [Failure Behavior](quickstart.md)).

## Container Deployment

The host is designed to run as a container entrypoint. The image installs the host and the providers it should serve, and the container `CMD` runs the binary with the right flags. When running against an Ultravisor on the container host, address it with `host.docker.internal`:

```bash
retold-beacon-host \
	--port 54400 \
	--beacon-name mi-beacon-1 \
	--ultravisor-url http://host.docker.internal:54321 \
	--provider retold-beacon-provider-meadow-integration \
	--config /app/data/config.json
```

> The host does not ship a Dockerfile in the published package. The container packaging (base image, install steps, volume mounts for the config file) is left to the deployment that consumes it.

## Failure Behavior

The host fails fast and loudly:

- **Bad arguments** -- a missing or out-of-range `--port`, a missing `--ultravisor-url`, or zero `--provider` flags print an error and exit with code `2`.
- **Provider load failure** -- a provider that cannot be `require()`d, does not export a constructor, or throws in its constructor prints an error and exits with code `1`.
- **Orator start failure** -- if the web server fails to initialize or start, the host prints an error and exits with code `1`.
- **Beacon registration failure** -- a registration error is logged as a warning only; the host stays up and the web server keeps listening.

## Stopping the Host

The host installs handlers for `SIGINT` and `SIGTERM`. On either signal it logs a shutdown message and exits after a short delay. (See the [Unknowns](architecture.md) section for what the host does and does not do to gracefully drain the beacon on shutdown.)

## Next Steps

- [Architecture](architecture.md) -- The full boot-to-register flow and the provider contract.
- [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) -- How to author the provider classes this host loads.
- [ultravisor](https://stevenvelozo.github.io/ultravisor/) -- The orchestration server the beacon registers with.
