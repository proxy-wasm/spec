# WebAssembly for Proxies (ABI specification)

This repository contains specification of the low-level Application Binary Interface (ABI) and
conventions to use between L4/L7 proxies (and/or other host environments) and their extensions
delivered as WebAssembly modules.

The event-driven streaming APIs and convenient utility functions were originally developed for
the [WebAssembly in Envoy] project, but they are proxy-agnostic, and consumers can use the same
Proxy-Wasm extensions across different proxies.

## Implementations

### Host environments

* [Envoy / Istio Proxy]
* [ATS] (proof-of-concept)

### SDKs

* [C++]
* [Rust]
* [AssemblyScript]
* [TinyGo]

[WebAssembly in Envoy]: docs/WebAssembly-in-Envoy.md
[Envoy / Istio Proxy]: https://github.com/envoyproxy/envoy
[ATS]: https://github.com/jplevyak/trafficserver/tree/wasm
[C++]: https://github.com/proxy-wasm/proxy-wasm-cpp-sdk
[Rust]: https://github.com/proxy-wasm/proxy-wasm-rust-sdk
[AssemblyScript]: https://github.com/solo-io/proxy-runtime
[TinyGo]: https://github.com/tetratelabs/proxy-wasm-go-sdk
