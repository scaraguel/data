---
type: integration architecture
title: Provider Contracts, Registry, and External Integrations
description: Explains how provider adapters are discovered, configured, invoked, normalized, and protected by shared resilience controls, and how to extend the provider system safely.
tags: [providers, integrations, registry, credentials, resilience, ohlcv]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-10T03:14:30.944Z
sources:
  - id: openwiki-source-8403868e612826b9fce67a88
    resource: repo://docs/contributing/creating-a-provider.md
  - id: openwiki-source-05ccef8d4cf1698187f20464
    resource: repo://pyproject.toml
  - id: openwiki-source-295e436b17e869c601ed374b
    resource: repo://src/ml4t/data/config/models.py
  - id: openwiki-source-c47d52b44ef5b9a4fb08e8a1
    resource: repo://src/ml4t/data/managers/provider_manager.py
  - id: openwiki-source-8f7b2201a64bcb138f032b12
    resource: repo://src/ml4t/data/providers/AGENTS.md
  - id: openwiki-source-137a6456fab0fde8c2fc55a3
    resource: repo://src/ml4t/data/providers/async_base.py
  - id: openwiki-source-98884e5bb3109415f376bc10
    resource: repo://src/ml4t/data/providers/base.py
  - id: openwiki-source-16b99adce9e4a870239c3062
    resource: repo://src/ml4t/data/providers/mixins/circuit_breaker.py
  - id: openwiki-source-ade50e6f7b885df1e3bfed36
    resource: repo://src/ml4t/data/providers/mixins/rate_limit.py
  - id: openwiki-source-5ef2c29bed81a700d273dda5
    resource: repo://src/ml4t/data/providers/mixins/session.py
  - id: openwiki-source-3d69b94635f7d4bfef769f01
    resource: repo://src/ml4t/data/providers/mixins/validation.py
  - id: openwiki-source-cb2504f5c1984905472542ea
    resource: repo://src/ml4t/data/providers/protocols.py
  - id: openwiki-source-c150037eb8b6ecccdb682066
    resource: repo://src/ml4t/data/providers/registry.py
  - id: openwiki-source-c95d49437bdaacf85f2ef471
    resource: repo://src/ml4t/data/utils/global_rate_limit.py
  - id: openwiki-source-49c646ec9192582427ba7ffd
    resource: repo://tests/test_base_provider_enhanced.py
  - id: openwiki-source-6245822e0f433fc33f967d95
    resource: repo://tests/test_provider_consistency.py
  - id: openwiki-source-62d11250053ddea4943defe4
    resource: repo://tests/test_provider_registry_contract.py
generated: { by: "openwiki/0.9.0", at: "2026-11-10T03:14:30.944Z" }
---

# Provider Contracts, Registry, and External Integrations

The provider subsystem is the boundary between ML4T Data and external or packaged data sources. It has two complementary layers:

- **Structural contracts** describe what a caller may do: fetch OHLCV, factors, events, timestamp-bounded observations, or asynchronous OHLCV.
- **Registry metadata and base behavior** describe how an implementation is discovered, configured, instantiated, protected, and normalized.

A provider is therefore not merely an HTTP client. It is an adapter that preserves the source's real semantics while returning a library-owned data model. The manager path is covered in [Data Manager Runtime](../architecture/data-manager-runtime.md); date, timezone, and symbol rules are expanded in [Data, Time, and Identity Contracts](../concepts/data-time-and-identity-contracts.md).

## Contracts and canonical output

[`protocols.py`](../../src/ml4t/data/providers/protocols.py) uses runtime-checkable `Protocol` types, so implementations can satisfy a contract structurally without inheriting a particular base class:

| Contract | Intended shape |
|---|---|
| `OHLCVProvider` | `fetch_ohlcv(symbol, start, end, frequency)` plus `name` and `capabilities()` |
| `TimestampRangeProvider` | Optional `fetch_ohlcv_range()` with half-open `start <= timestamp < end` bounds |
| `FactorProvider` | Named factor datasets through `fetch()` and `list_datasets()` |
| `EventProvider` | Event or prediction-market data through `fetch_events()` and `list_markets()` |
| `AsyncOHLCVProvider` | Awaitable `fetch_ohlcv_async()` plus `name` and `capabilities()` |

`ProviderCapabilities` is the object-level description exposed by a provider. The registry's string capability set is separate static routing/discovery metadata. Keep both truthful: callers use registry capabilities before construction, while direct users can inspect the instance.

For the common OHLCV path, `BaseProvider` owns a template method and adapters implement either `_fetch_and_transform_data()` or the pair `_fetch_raw_data()` and `_transform_data()`. The boundary result starts with these columns, in this order:

```text
timestamp, symbol, open, high, low, close, volume
```

`ValidationMixin` turns that shape into a stronger contract. It normalizes timestamps to `Datetime("us", "UTC")`, symbols to uppercase strings, and OHLCV values to `Float64`; sorts by timestamp and symbol; preserves optional columns after the canonical seven; and rejects nulls, non-finite numeric values, negative volume, mixed or incorrect symbol identity, and duplicate `(timestamp, symbol)` rows. A zero-column empty sentinel is normalized into a typed empty canonical frame. OHLC relationship failures raise by default; the explicitly selected `drop` and `warn` modes instead remove or retain bad rows. Do not use those modes to conceal an adapter mapping error.

Date bounds are not globally uniform: `fetch_ohlcv()` documents that most adapters treat both dates as inclusive while Yahoo handles an exclusive upstream end internally. An adapter must preserve and test its source-specific boundary, pagination, timezone, and no-data meaning rather than guessing them.

## Registry: inspect first, import only when selected

[`registry.py`](../../src/ml4t/data/providers/registry.py) is the canonical static inventory. Each immutable `ProviderSpec` records the stable name, module and class, description, capability strings, credential requirements, required non-secret configuration, optional package extra, and whether the adapter is advertised, deprecated, or manager-compatible. `PROVIDER_REGISTRY` is a read-only mapping, `get_provider_spec()` performs case-insensitive lookup, and `advertised_provider_specs()` preserves registry order while hiding internal entries such as `mock`.

Registry inspection itself does not import an adapter. `ProviderSpec.load_class()` performs the module import only when code asks for the class. This allows CLI discovery and configuration checks to inspect requirements without network access. The public `ml4t.data.providers` namespace also tolerates missing optional imports by exporting `None` for guarded integrations, while registry contract tests ensure advertised exports resolve when their extras are installed.

> **Import invariant:** importing provider metadata or the provider package must not contact a remote service, read required credentials as a precondition, or initialize paid work. Constructors may configure a client, but network activity belongs to an explicit operation.

`ProviderManager` loads manager-compatible classes lazily as a class map, skips classes whose optional import raises `ImportError`, and copies the result into each manager. Availability then requires both a loadable class and a `ProviderSpec.is_configured()` result. That check is local only: `enabled: false` disables the instance; every required credential must have a non-empty configured value or documented environment value; unresolved `${NAME}` placeholders do not count; and required configuration fields must be present.

### Managed providers, direct APIs, and aliases

A registry entry with `manager_compatible=True` can participate in `DataManager`'s symbol-based OHLCV path. Entries such as factor, macro, event-only, tick, or artifact-oriented adapters may be marked `manager_compatible=False`; they remain registered and discoverable but must be used through their direct class API. The manager rejects a direct-API-only type and checks `required_capability` before construction, so an `aqr` request cannot accidentally flow into the OHLCV fetch path.

Configuration can name multiple instances of one implementation. If a provider block's key is an instance name and its `type` is a different registered provider name, `_register_configured_aliases()` binds that local name to the registered class and spec. The alias is rejected for direct-API-only types and remains unavailable if its optional implementation cannot be imported. These configured aliases are manager-instance state; `register_provider()` likewise modifies only one manager's copied class map, not the global registry.

`ProviderRouter` is a separate symbol-routing helper. Explicit overrides win, decisions are cached, and changing patterns clears the cache. Its defaults infer only unambiguous formats (`AAA_BBB` to OANDA and continuous `ROOT.v.N` futures to Databento); bare tickers and compact pairs are intentionally not guessed.

## Credentials and optional extras

A `CredentialRequirement` pairs one constructor/config field with one or more accepted environment names. Multiple requirements are conjunctive—for example, Alpaca needs both key and secret—while environment names within one requirement are alternatives. Optional credential environments represent services that can operate without a key but may gain authenticated access. `required_configuration` is for non-credential constructor inputs such as a parquet path or samples.

The manager strips structural fields (`enabled`, `name`, `type`) before construction and merges provider-specific `extra` values underneath top-level fields. Configuration validation forbids credentials and other reserved fields inside `extra`. `get_provider_info()` exposes requirements and whether a key exists, but removes credential fields from both the top level and nested `extra` before returning the configuration view.

Credentials must never be included in logs, exception context, serialized resolved configuration, or URLs when authenticated headers are supported. Use documented environment references such as `${TIINGO_API_KEY}` for persisted configuration. A warning about a plaintext-looking credential does not make logging the value acceptable.

Some adapters require installation extras. The registry's `extra` value is the user-facing installation hint and must agree with `[project.optional-dependencies]` in `pyproject.toml` (currently including `yahoo`, `databento`, and `oanda`). Keep imports for such dependencies inside the adapter boundary or guarded public export. A missing extra should make only that selected integration unavailable and should produce an actionable installation message; it must not break base package import or unrelated providers.

## Invocation, resilience, and lifecycle

`ProviderManager.get_provider()` validates availability and capability, constructs the provider once from sanitized provider-specific configuration, caches it, and returns the same instance thereafter. This reuses the adapter's `httpx.Client` and its connection pool. `SessionMixin` defaults to a 30-second timeout, ten connections, and five keep-alive connections. Use a provider as a context manager or call `close()`; `ProviderManager.close_all()` closes cached providers and clears the cache. “Shared session” means reused by calls to that provider instance, not one cross-provider global HTTP client.

`BaseProvider.fetch_ohlcv()` is decorated by the shared `provider_retry` policy. Each attempt validates inputs, acquires the provider's rate limiter, then invokes fetch, transformation, and canonical validation inside the instance circuit breaker. `RateLimitMixin` obtains its limiter from the process-wide `GlobalRateLimitManager`; instances with the same provider name and rate configuration therefore share quota accounting. Different configurations produce distinct limiter keys.

The retry policy makes at most three attempts for `NetworkError` instances whose `retryable` flag remains true. It uses exponential delay (normally 4–10 seconds), honors a longer provider `retry_after` hint, and caps one wait at 60 seconds. `_request()` maps transport failures, HTTP 429, 404, 5xx, and other HTTP failures to shared exceptions; malformed or non-object JSON from `_request_json()` is a `DataValidationError`. Adapters should use these helpers instead of adding private sleep loops or incompatible exception trees.

The circuit breaker starts `CLOSED`, counts transient `NetworkError` failures, and excludes `RateLimitError` plus permanent/client/data failures from service-health counts. At the default threshold of five it becomes `OPEN` and rejects calls. After the reset timeout (default 300 seconds), one call moves it to `HALF_OPEN`; success closes and resets it, while another counted failure reopens it. The breaker and its counters are instance-local, whereas the rate limiter is process-global by provider/configuration key.

```mermaid
sequenceDiagram
    participant Caller
    participant Manager as Provider Manager
    participant Provider as Base Provider
    participant Limiter as Global Rate Limiter
    participant Breaker as Circuit Breaker
    participant Adapter
    participant Remote as External Service
    participant Validator as Canonical Validator

    Caller->>Manager: request provider and OHLCV capability
    Manager->>Manager: check registry availability and capability
    alt unavailable or unsupported
        Manager-->>Caller: ValueError before construction or fetch
    else provider available
        Manager-->>Caller: cached or newly constructed provider
        Caller->>Provider: fetch_ohlcv
        loop at most three attempts for retryable NetworkError
            Provider->>Provider: validate symbol and date inputs
            alt invalid input
                Provider-->>Caller: ValueError without remote call
            else valid input
                Provider->>Limiter: acquire shared quota
                Limiter-->>Provider: permission after any blocking wait
                Provider->>Breaker: execute protected fetch
                alt circuit open before reset timeout
                    Breaker-->>Provider: CircuitBreakerOpenError
                    Provider-->>Caller: fail without remote call
                else circuit closed or half-open probe
                    Breaker->>Adapter: fetch and transform
                    Adapter->>Remote: request through reused HTTP client
                    alt transient transport, 429, or server failure
                        Remote-->>Adapter: error response or transport error
                        Adapter-->>Breaker: classified shared exception
                        Breaker-->>Provider: count only service-health failure
                        alt error remains retryable and attempts remain
                            Provider->>Provider: bounded backoff then retry
                        else permanent or attempts exhausted
                            Provider-->>Caller: classified failure
                        end
                    else successful response
                        Remote-->>Adapter: source payload
                        Adapter-->>Breaker: transformed DataFrame
                        Breaker->>Validator: normalize and validate
                        alt malformed or noncanonical data
                            Validator-->>Breaker: DataValidationError
                            Breaker-->>Provider: ignore for health count
                            Provider-->>Caller: validation failure without retry
                        else canonical frame
                            Validator-->>Breaker: validated frame
                            Breaker-->>Provider: success and close or reset circuit
                            Provider-->>Caller: canonical OHLCV DataFrame
                        end
                    end
                end
            end
        end
    end
```

*The managed OHLCV call fails early on discovery or input errors, applies shared quota and service-health controls, retries only classified transient failures, and returns only canonical data.*

Native asynchronous adapters use `AsyncBaseProvider`: it lazily creates an async session, limits in-flight work with a per-instance semaphore (default ten), validates through the same OHLCV validator, and can gather independent symbols. The synchronous `BaseProvider.fetch_ohlcv_async()` is only a thread wrapper; a genuinely asynchronous adapter should override it. Note that `AsyncBaseProvider` currently composes rate limiting, validation, and async sessions, but not the synchronous base's retry and circuit-breaker template—an async adapter must deliberately use supported async resilience helpers rather than assume parity.

## Safe adapter changes and additions

Treat an adapter change as a contract change at an external boundary:

1. **Record the upstream contract.** Confirm authentication, instruments, frequency vocabulary, historical limits, inclusive/exclusive bounds, pagination, timestamp timezone, quotas, tier restrictions, and error responses from provider documentation or fixtures.
2. **Choose the right interface.** Use `BaseProvider` for manager-compatible synchronous OHLCV, `AsyncBaseProvider` for truly async acquisition, or implement the factor/event/range protocol or an existing direct API pattern. Do not force non-OHLCV data through the OHLCV manager.
3. **Transform at the boundary.** Map source fields and identity explicitly. Never fabricate timezone, volume, date semantics, or missing records merely to pass validation. Return `_create_empty_dataframe()` for a legitimate empty result.
4. **Reuse shared controls.** Prefer `_request()` and `_request_json()`, the shared client, exception classes, rate limiting, retry policy, circuit breaker, and validation. If a service encodes errors unusually, override `_classify_error_response()` narrowly.
5. **Keep import and secret boundaries clean.** No import-time I/O or credential requirement; no secret logging; no unrelated optional import. Add an optional extra only when necessary and make failure actionable.
6. **Register metadata atomically.** Add or update the `ProviderSpec`, `ProviderType`, public export, package extra, and provider documentation together. Set capability and `manager_compatible` flags from behavior, not aspiration. Consider aliases as configured instances, not new global provider types.
7. **Test the public path offline.** Drive `fetch_ohlcv()` with `httpx.MockTransport` or a deterministic fixture. Cover success, empty and malformed responses, auth, missing symbols, 429 and transient failures, pagination, bounds, timezone conversion, symbol identity, canonical schema, and client closure.
8. **Run focused and cross-provider gates.** Start with the adapter test, then `tests/test_provider_registry_contract.py`, `tests/test_provider_consistency.py`, and `tests/test_base_provider_enhanced.py`; run lint and type checks before the full suite. See [Test Strategy and Quality Gates](../testing/test-strategy-and-quality-gates.md) and [Acquire, Validate, and Persist](../workflows/acquire-validate-and-persist.md).

The default pytest lane explicitly excludes `slow`, `paid_tier`, `integration`, and `requires_api_key`. Live, credentialed, paid, expensive, or slow checks must remain in marked, opt-in lanes; ordinary provider tests mock the external service. This protects contributors from accidental spend, nondeterminism, secret exposure, and network dependence while still allowing bounded live contract evidence when it is intentionally scheduled.
