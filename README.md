# WebAssembly for Proxies (ABI specification)

This repository contains specification of the low-level Application Binary Interface (ABI) and
conventions to use between L4/L7 proxies (and/or other host environments) and their extensions
delivered as WebAssembly modules.

The event-driven streaming APIs and convenient utility functions were originally developed for
the [WebAssembly in Envoy] project, but they are proxy-agnostic, and consumers can use the same
Proxy-Wasm extensions across different proxies.

[WebAssembly in Envoy]: docs/WebAssembly-in-Envoy.md

## Specification

The latest and widely implemented version of the specification is [v0.2.1].

[v0.2.1]: abi-versions/v0.2.1/README.md

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

* [ATS]
* [Envoy]
* [NGINX]
* [MOSN]
* [OpenResty]

[ATS]: https://docs.trafficserver.apache.org/en/latest/admin-guide/plugins/wasm.en.html
[Envoy]: https://github.com/envoyproxy/envoy
[NGINX]: https://github.com/Kong/ngx_wasm_module
[MOSN]: https://github.com/mosn/mosn
[OpenResty]: https://github.com/api7/wasm-nginx-module

#### Libraries

* [C++ Host]
* [Go Host]

[C++ Host]: https://github.com/proxy-wasm/proxy-wasm-cpp-host
[Go Host]: https://github.com/mosn/proxy-wasm-go-host

## Community

There is a [Proxy-Wasm on Slack]. Feel free to join and engage with the community.

[Proxy-Wasm on Slack]: https://join.slack.com/t/proxy-wasm/shared_invite/zt-2nragshr6-nYH7p8jfBZevFIHpX~LIvg
