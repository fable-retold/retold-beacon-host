# Retold Beacon Host

> Generic container host for ultravisor-beacon CapabilityProvider classes published as npm packages

Retold Beacon Host is a small, ready-made process that turns a published beacon capability into a running beacon. It boots an [Orator](https://fable-retold.github.io/orator/) web server, loads one or more `CapabilityProvider` packages by name, wraps them in a single beacon, and registers that beacon with a target [Ultravisor](https://stevenvelozo.github.io/ultravisor/).

The host exists so a capability published to npm can be deployed without hand-writing a boot script for every worker. The expected deployment shape is a container: an image installs the host plus the desired provider packages, and the container entrypoint runs the host with flags that select the providers and point at the Ultravisor.

## What It Does

- **Loads providers by name** -- pass one or more `--provider <npm-package>` flags; each package is `require()`d and instantiated.
- **Serves a single beacon** -- all loaded providers register their capabilities through one beacon and one Orator server.
- **Registers with Ultravisor** -- the beacon connects to the `--ultravisor-url` you give it and comes online.
- **Stays out of the way** -- a single JSON `--config` file is handed unchanged to every provider; each provider reads only the keys it needs.

## Where It Fits

| Module | Role |
|--------|------|
| [ultravisor](https://stevenvelozo.github.io/ultravisor/) | The orchestration server the host's beacon registers with. |
| [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) | The beacon client/service the host instantiates to register and run capabilities. |
| [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) | The convention-based base class used to author the provider classes the host loads. |
| **retold-beacon-host** | The generic process that loads those provider packages and stands them up as a beacon. |

For a purpose-built beacon application (rather than a generic provider host), see [retold-databeacon](https://fable-retold.github.io/retold-databeacon/).

## Quick Look

```bash
retold-beacon-host \
	--port 54400 \
	--beacon-name mi-beacon-1 \
	--ultravisor-url http://host.docker.internal:54321 \
	--provider retold-beacon-provider-meadow-integration \
	--config /app/data/config.json
```

## Documentation

- [Quick Start](quickstart.md) -- Install, run with one or more providers, register with an Ultravisor.
- [Architecture](architecture.md) -- The boot, load-providers, and register flow, with the parts that cannot be confirmed from the published code marked as unknown.

## A Note on This Documentation

This documentation is reconstructed from the host's command-line entry point (`bin/retold-beacon-host.js`) and its `package.json`. The published package ships only that binary -- there is no `source/` tree -- so behavior that lives inside the provider packages or the `ultravisor-beacon` dependency is described from the host's point of view and, where it cannot be confirmed, called out explicitly in the [Unknowns](architecture.md) section of the architecture page.

## License

MIT.
