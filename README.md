# Retold Beacon Host

Generic container host for [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) `CapabilityProvider` classes that are published as npm packages. It boots an [Orator](https://fable-retold.github.io/orator/) web server, `require()`s one or more provider packages by name, wraps them in a single beacon, and registers that beacon with a target [Ultravisor](https://stevenvelozo.github.io/ultravisor/).

The intended deployment shape is a container: build an image that installs `retold-beacon-host` plus the provider packages you want, then run the host as the container entrypoint with the flags that select the providers and point at your Ultravisor. It also runs on a developer box via `npx retold-beacon-host ...` when the providers are installed in the same `node_modules` tree.

## Why

The [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) module makes it easy to write a beacon capability as a class. The Retold Beacon Host is the other half: a ready-made process that loads those provider classes and stands them up as a running beacon, so a capability published to npm can be deployed without writing a bespoke boot script for every worker.

## Installation

The host is normally installed alongside one or more provider packages, then run from the install tree:

```bash
npm install retold-beacon-host
npm install retold-beacon-provider-meadow-integration
```

In a container image, install the host and providers during the build and run the `retold-beacon-host` binary as the entrypoint.

## Usage

Run the host with at least one provider and a target Ultravisor:

```bash
retold-beacon-host \
	--port 54400 \
	--beacon-name mi-beacon-1 \
	--ultravisor-url http://host.docker.internal:54321 \
	--provider retold-beacon-provider-meadow-integration \
	--config /app/data/config.json
```

Load several providers into one beacon by repeating `--provider`:

```bash
retold-beacon-host \
	--port 54400 \
	--ultravisor-url http://localhost:54321 \
	--provider retold-beacon-provider-meadow-integration \
	--provider retold-beacon-provider-shell
```

### CLI Flags

| Flag | Required | Default | Description |
|------|----------|---------|-------------|
| `--port <N>` | Yes | -- | Local HTTP port for the host's Orator server. Must be 1-65535. |
| `--ultravisor-url <URL>` | Yes | -- | Base URL of the Ultravisor API to register the beacon with. |
| `--provider <npm-package>` | Yes (one or more) | -- | npm package name of a `CapabilityProvider` module. Repeat the flag to load multiple providers into one beacon. |
| `--beacon-name <name>` | No | `host` | Name the beacon registers under with the Ultravisor. |
| `--config <path-to-json>` | No | -- | Path to a JSON file passed unchanged to every provider constructor. |

The flag parser reads only the flags listed above; there are no short aliases and no environment-variable fallbacks.

## How It Works

1. Parse and validate the CLI flags. A missing or out-of-range `--port`, a missing `--ultravisor-url`, or zero `--provider` flags causes the process to print an error and exit with code `2`.
2. If `--config` is given, read and `JSON.parse` that file into a single config object. A read or parse failure logs a warning and continues with an empty config object.
3. Construct a [Pict](https://fable-retold.github.io/pict/) instance configured with the requested `APIServerPort`.
4. Load each `--provider` package: `require()` it, find the exported constructor, and instantiate it as `new Provider(pProviderConfig, pPict)`. The same parsed config object is handed to every provider; each provider reads only the keys it cares about. Any provider that fails to require, does not export a constructor, or throws in its constructor causes the process to exit with code `1`.
5. Wire up Orator (with the Restify service server), initialize it, and start the web server on `--port`.
6. Register the beacon: create an `UltravisorBeacon` service pointed at `--ultravisor-url` under `--beacon-name`, register each provider's capability, and call `enable()` to bring the beacon online. A registration error is logged as a warning and does not stop the host.

See [docs/architecture.md](docs/architecture.md) for the full boot sequence and the provider contract.

## Provider Contract

A provider package must export a class (either as the module itself or as a `default` export) whose constructor accepts `(pProviderConfig, pPict)`. After construction, the host registers the provider's capability using whichever of these the instance offers:

- a `register(pBeacon)` method, which the host calls and lets the provider register itself; or
- a `Capability` string together with an `actions` object, which the host passes to the beacon's `registerCapability(...)`.

A provider that offers neither is skipped with a warning. See [docs/architecture.md](docs/architecture.md) for details and the unknowns around how providers are expected to be authored.

## Related Modules

- [ultravisor](https://stevenvelozo.github.io/ultravisor/) -- Process supervision and workflow orchestration server; the registration target for the host's beacon.
- [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) -- Beacon client and Fable service the host uses to register and run capabilities.
- [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) -- Convention-based base class for authoring the `CapabilityProvider` classes this host loads.
- [retold-databeacon](https://fable-retold.github.io/retold-databeacon/) -- A purpose-built data beacon application; a comparison point for the generic, provider-loading host.

## License

MIT. See [LICENSE](LICENSE).

Repository: [github.com/fable-retold/retold-beacon-host](https://github.com/fable-retold/retold-beacon-host)
