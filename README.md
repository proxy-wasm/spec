# WebAssembly for Proxies (ABI specification)

This repository contains specification of the low-level Application Binary Interface (ABI) and
conventions to use between L4/L7 proxies (and/or other host environments) and their extensions
delivered as WebAssembly modules.

The event-driven streaming APIs and convenient utility functions were originally developed for
the [WebAssembly in Envoy] project, but they are proxy-agnostic, and consumers can use the same
Proxy-Wasm extensions across different proxies.

[WebAssembly in Envoy]: docs/WebAssembly-in-Envoy.md

## Implementations

### SDKs

* [AssemblyScript SDK]
* [C++ SDK]
* [Go (TinyGo) SDK]
* [Rust SDK]

[AssemblyScript SDK]: https://github.com/solo-io/proxy-runtime
[C++ SDK]: https://github.com/proxy-wasm/proxy-wasm-cpp-sdk
[Go (TinyGo) SDK]: https://github.com/tetratelabs/proxy-wasm-go-sdk
[Rust SDK]: https://github.com/proxy-wasm/proxy-wasm-rust-sdk

### Host environments

#### Servers

* [Envoy]
* [Istio Proxy] (Envoy-based)
* [MOSN]
* [ATS] (proof-of-concept)

[Envoy]: https://github.com/envoyproxy/envoy
[Istio Proxy]: https://github.com/istio/proxy
[MOSN]: https://github.com/mosn/mosn
[ATS]: https://github.com/jplevyak/trafficserver/tree/wasm

#### Libraries

* [C++ Host]
* [Go Host]

[C++ Host]: https://github.com/proxy-wasm/proxy-wasm-cpp-host
[Go Host]: https://github.com/mosn/proxy-wasm-go-host
