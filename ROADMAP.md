# Proxy-Wasm Roadmap

The Proxy-Wasm community and maintainers envision an evolution path that has the
following tracks:

-   [Spec / ABI](#abi)
-   [SDKs / language support](#sdks)
-   [Host features](#host)
-   [Envoy integration](#envoy)

Each track is described in more detail below, with owners and ETAs listed for
efforts currently in development. This roadmap should not be construed as a set
of commitments, but rather a set of directions that are subject to change in
response to community interest and contributions.

The overarching goals of this document are to:

-   Publish areas of current investment.
-   Encourage external contributors by pointing out feature gaps.
-   Align this repository with the vision of WebAssembly: a portable technology
    that is cross-language, cross-platform, and cross-provider.

<a name="abi"></a>

## Spec / ABI

-   (@piotrsikora, @mpwarres) Publish ABI v0.3. The list of
    [planned changes](https://github.com/proxy-wasm/spec/milestone/1) includes:
    -   Feature negotiation (proxy-wasm/spec#71 and proxy-wasm/spec#56)
    -   Better header/body buffering support (proxy-wasm/spec#63)
    -   Support for HTTP fields with multiple values (proxy-wasm/spec#53)
-   (Help wanted) WASI convergence. We want to adopt the component model at WASI
    1.0. There is a lot of overlap between Proxy-Wasm and some WASI proposals
    ([wasi-http](https://github.com/WebAssembly/wasi-http),
    [wasi-keyvalue](https://github.com/WebAssembly/wasi-keyvalue), etc). In the
    short term, we'd like to define the Proxy-Wasm ABI in
    [WIT](https://github.com/WebAssembly/component-model/blob/main/design/mvp/WIT.md),
    to understand:
    -   How does the ABI surface map to WASI types and components?
    -   Can WASI ABIs wholly replace Proxy-Wasm ABIs? Is performance on par?
    -   Is there room for language SDKs, or is wit-bindgen the future?
    -   How should we evolve Proxy-Wasm to become WASI-compatible? What are good
        incremental steps?
    -   See also: https://github.com/krinkinmu/wasi-http
-   (Help wanted) Evaluate uses of foreign functions to identify feature gaps.
    -   For example, Envoy
        [registers foreign functions](https://github.com/search?q=repo%3Aenvoyproxy%2Fenvoy%20RegisterForeignFunction&type=code)
        for signature checking, compression, filter state, route cache, and CEL
        expressions.
    -   Are there similar extensions in Nginx? Apache Traffic Server?
    -   When/how do we promote these FFIs? Do they become negotiated features or
        full members of the ABI?

<a name="sdks"></a>

## SDKs / language support

-   (@leonm1, done!) Fork the abandoned Go SDK + support full Golang.
-   (Google exploring) Build a Python SDK using a MicroPython port.
-   (Help wanted) Stop using Emscripten in the C++ SDK. Instead use Clang /
    wasi-sdk (proxy-wasm/proxy-wasm-cpp-sdk#167).
-   (Help wanted) Build a Lua SDK using a Lua interpreter. Creates a safer
    alternative to Envoy's
    [Lua filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/lua_filter),
    and could benefit Nginx's Lua-based [OpenResty](https://openresty.org/)
    ecosystem.

<a name="host"></a>

## Host features

-   (@mpwarres) CppHost maintenance.
    -   Update v8 and upstream some patches for v8 warming / extension.
    -   Update the protobuf dependency.
    -   Set up dependabot.
-   (Help wanted) Prototype
    [HyperLight](https://github.com/hyperlight-dev/hyperlight-wasm) as a
    KVM-based sandboxing layer around wasm runtimes. The allure is getting an
    inexpensive and transparent second layer of security at a thread boundary,
    which makes it more feasible to run fully untrusted workloads with
    Proxy-Wasm.
-   (Help wanted) Performance benchmarks. One of Proxy-Wasm's strengths is its
    ability to swap between multiple wasm runtimes. Help users make an informed
    decision by benchmarking cold start and execution costs across runtimes.
-   (Help wanted) Adopt CPU metering as a first-class feature. Leverage
    instruction counting where available. For other engines (e.g. v8), use a
    watchdog thread.
-   (Help wanted) Support dynamic (per VM) limits for RAM and CPU.
-   (Help wanted) Expand the use of SharedArrayBuffer to reduce memcpy into wasm
    runtimes. This is promising for HTTP body chunks (see relevant
    [WASI issue](https://github.com/WebAssembly/WASI/issues/594)) and
    [wasm binaries](https://github.com/proxy-wasm/proxy-wasm-cpp-host/blob/21a5b089f136712f74bfa03cde43ae8d82e066b6/src/v8/v8.cc#L272).
-   (Help wanted) Implement NullVM for Rust and/or Go. For proxy owners with
    trusted extensions, achieve native performance while maintaining
    WebAssembly's portability.

<a name="envoy"></a>

## Envoy integration

-   (@mpwarres, @botengyao) Get Envoy's inline wasm filter out of alpha
    (envoyproxy/envoy#36996). Documentation, security scanning, tests, bug
    fixes, etc.
-   (@mpwarres) Implement the v0.3 Proxy-Wasm ABI.
-   (Help wanted) Decouple from the thread-local execution model. As wasm
    modules become more CPU intensive and leverage multiple async APIs, consider
    managing a separate Proxy-Wasm threadpool. Each VM needs a work queue, and
    requests need affinity to a single VM. This architecture allows for
    independent thread scaling (expensive wasms get more CPU), improved
    parallelism (multiple requests' wasm at the same time), and reduced memory
    costs (one VM serves multiple Envoy threads). It adds performance risks (CPU
    scheduling latency, CPU cache misses, NUMA hopping).
-   (Help wanted) Envoy has a single implementation for the entire Proxy-Wasm
    host. Add extension points for different Proxy-Wasm interfaces (telemetry,
    network calls, key value, shared queue), so that Envoy operators may provide
    their own implementations.
