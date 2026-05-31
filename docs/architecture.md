# Architecture

Retold Beacon Host is a single command-line process. It has no `source/` tree -- the entire program is the `bin/retold-beacon-host.js` entry point. This page describes the boot sequence, the provider contract, and the dependency wiring that the entry point makes observable, and ends with an [Unknowns](architecture.md) section listing everything that cannot be confirmed from the published package.

## Dependencies

The host declares four runtime dependencies:

| Package | Role in the host |
|---------|------------------|
| [pict](https://fable-retold.github.io/pict/) | The service manager and application container. The host builds one Pict instance and registers all other services on it. |
| [orator](https://fable-retold.github.io/orator/) | The HTTP server. Loaded as a Pict service type and started on `--port`. |
| `orator-serviceserver-restify` | The Restify-backed service server Orator uses for its web server. |
| [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) | The beacon client/service. The host registers it as a Pict service type and uses it to advertise capabilities and connect to the Ultravisor. |

The provider packages themselves are **not** dependencies of the host. They are resolved at runtime with `require(<package-name>)` from whatever `node_modules` tree the host is installed into, which is why the deployment image (or developer box) must install the providers alongside the host.

## Boot Sequence

The entry point runs top to bottom in this order:

### 1. Parse arguments

`process.argv` is scanned for the recognized flags. The parser collects:

- `Port` from `--port` (parsed as a base-10 integer; defaults to `0`)
- `BeaconName` from `--beacon-name` (defaults to `host`)
- `UltravisorURL` from `--ultravisor-url` (defaults to empty)
- `Providers` -- an array; each `--provider` appends one package name
- `ConfigPath` from `--config` (defaults to empty)

Unrecognized arguments are ignored. There are no short aliases and no environment-variable fallbacks.

### 2. Validate arguments

The host exits with code `2` and an error message if any of the following hold:

- `--port` is missing or outside the range 1-65535
- `--ultravisor-url` is missing
- no `--provider` was supplied

### 3. Read provider config

If `--config` was given, the file is read synchronously and parsed with `JSON.parse` into a single config object. A read or parse error is logged as a warning and the host continues with an empty object (`{}`). The same object is later passed to every provider constructor.

### 4. Build the Pict instance

A Pict instance is constructed with:

```javascript
let _pict = new libPict(
	{
		Product:       'Retold-Beacon-Host',
		LogNoisiness:  1,
		APIServerPort: _args.Port
	});
```

Pict supplies the service manager that the Orator and beacon services are registered on.

### 5. Load providers

Providers are loaded **after** the Pict instance exists, so a provider constructor that needs to register services has a real host to attach to. For each `--provider` package name, the host:

1. `require()`s the package. A failure prints an error and exits with code `1`.
2. Resolves the constructor: it uses the module export directly if it is a function, otherwise it uses the module's `default` export. If neither is a function, it prints an error and exits with code `1`.
3. Instantiates it as `new Provider(pProviderConfig, pPict)` -- the parsed config object first, the host Pict instance second. A constructor that throws prints an error and exits with code `1`.
4. Logs the loaded provider, reading the instance's `Name` and `Capability` properties for the log line (falling back to `unnamed` / `unknown` when absent).

### 6. Wire and start Orator

The host registers two service types on the Pict service manager and starts the server:

```javascript
_pict.serviceManager.addServiceType('OratorServiceServer', libOratorRestify);
_pict.serviceManager.addServiceType('Orator', libOrator);
_pict.serviceManager.instantiateServiceProvider('OratorServiceServer');
let _orator = _pict.serviceManager.instantiateServiceProvider('Orator');
```

Orator is initialized; on success the host installs the Restify body parser and calls `startWebServer`. A failure at either step prints an error and exits with code `1`. On a successful start the host logs the listening port and proceeds to beacon registration.

### 7. Register the beacon

The host registers the `UltravisorBeacon` service type and instantiates a beacon configured to reach the Ultravisor:

```javascript
let tmpBeacon = _pict.serviceManager.instantiateServiceProvider('UltravisorBeacon',
	{
		ServerURL:      _args.UltravisorURL,
		Name:           _args.BeaconName,
		MaxConcurrent:  5
	});
```

It then registers each loaded provider's capability (see [Provider Contract](architecture.md) below) and calls `tmpBeacon.enable(...)` to bring the beacon online. A registration error from `enable` is logged as a **warning only** -- the host keeps running and the web server keeps listening.

> The `MaxConcurrent: 5` value is hard-coded in the host and is not exposed as a CLI flag.

## Provider Contract

The host accepts two provider shapes so that provider classes do not have to implement a host-specific helper. When registering each provider with the beacon, the host checks, in order:

1. **`register(pBeacon)` method.** If the provider instance has a `register` function, the host calls `tmpProvider.register(tmpBeacon)` and lets the provider register itself however it likes.
2. **`Capability` + `actions`.** Otherwise, if the instance exposes both a `Capability` value and an `actions` value, the host calls:

   ```javascript
   tmpBeacon.registerCapability(
   	{
   		Capability: tmpProvider.Capability,
   		Name:       tmpProvider.Name || tmpProvider.Capability,
   		actions:    tmpProvider.actions
   	});
   ```

3. **Neither.** A provider with no `register()` and no `Capability`/`actions` pair is skipped with a warning, and the host continues.

In summary, a provider package must export a class (as the module itself or as `default`) whose constructor takes `(pProviderConfig, pPict)`, and the resulting instance should offer either a `register(pBeacon)` method or a `Capability`/`actions` surface. The instance may also expose a `Name` (used for logging and as the registered capability name).

## Lifecycle and Shutdown

The host installs handlers for `SIGINT` and `SIGTERM`. Each handler logs a shutdown message and exits the process after a short fixed delay.

## Boot Flow Summary

```
parse argv
  └─ validate (exit 2 on bad args)
       └─ read --config JSON (warn + empty object on failure)
            └─ new Pict({ Product, LogNoisiness, APIServerPort })
                 └─ load each --provider  (exit 1 on require/export/ctor failure)
                      └─ register + init Orator (exit 1 on failure)
                           └─ startWebServer  →  "listening on port N"
                                └─ instantiate UltravisorBeacon { ServerURL, Name, MaxConcurrent: 5 }
                                     └─ register each provider's capability
                                          └─ beacon.enable()  (warn-only on failure)
```

## Unknowns

The published package contains only the binary, so the following could not be confirmed and are intentionally left undocumented rather than guessed:

- **How providers receive and respond to work items.** The host registers capabilities and calls `enable()`, but the work-item dispatch, execution, and result-reporting loop lives entirely inside `ultravisor-beacon`. Whether providers use the convention-based action methods of [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) or the raw `actions` descriptor surface is up to each provider package; the host accepts both shapes but defines neither.
- **The exact structure of the `actions` object** the host forwards to `registerCapability`. The host passes it through verbatim; its schema is defined by `ultravisor-beacon`, not by the host.
- **The provider config JSON schema.** The host reads and parses the `--config` file and hands the object to every provider unchanged. The expected keys are defined by the providers, and the comment in the entry point notes the blob is "user-supplied via the per-type form" -- but no such form or schema ships with the host.
- **Authentication / password handling.** The host instantiates the beacon with `ServerURL`, `Name`, and `MaxConcurrent` only. Whether and how a password or token reaches the beacon (for example, through the provider config or an `ultravisor-beacon` default) is not visible in the host code.
- **What HTTP routes, if any, the Orator server exposes.** The host starts Orator with a body parser but registers no routes of its own in the entry point. Any routes would have to be added by a provider through the Pict instance it receives; whether providers do this, and what they expose, is not determined by the host.
- **Graceful beacon drain on shutdown.** The signal handlers exit after a fixed delay; the host does not call a beacon `disable`/`disconnect` in the shutdown path. Whether in-flight work is drained before exit depends on `ultravisor-beacon` behavior that is not exercised by the host.
- **Tags, bind addresses, heartbeat, and reconnect settings.** Sibling beacon modules expose options such as `Tags`, `BindAddresses`, `PollIntervalMs`, and `HeartbeatIntervalMs`. The host does not set any of these, so they fall to `ultravisor-beacon` defaults; the host provides no flags to override them.
- **Tests, build, and a Dockerfile.** None ship in the published package, so no test, build, or container-packaging conventions are documented here.

## Related Modules

- [ultravisor](https://stevenvelozo.github.io/ultravisor/) -- Orchestration server the beacon registers with.
- [ultravisor-beacon](https://stevenvelozo.github.io/ultravisor-beacon/) -- Beacon client/service the host instantiates.
- [ultravisor-beacon-capability](https://stevenvelozo.github.io/ultravisor-beacon-capability/) -- Base class for authoring provider capabilities.
- [retold-databeacon](https://fable-retold.github.io/retold-databeacon/) -- Purpose-built data beacon application.
