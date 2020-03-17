# Proxy-Wasm ABI specification

## ABI versioning

The ABI is in the form of: `<major>.<minor>.<patch>`

Breaking changes require increasing the major version.

Adding new functions requires increasing the minor version.

Adding new enum values requires increasing the patch version.

Wasm modules should export `proxy_abi_version_<major>_<minor>_<patch>` function, indicating which
version of the ABI they were compiled against.

Host environments should be able to support multiple versions of the ABI.

## Naming conventions

All functions are using `snake_case`, and they start with the `proxy_` prefix. Implementation
specific extensions should use different prefixes (e.g. `envoy_`, `nginx_`, `ats_`, etc.) in order
to avoid polluting the common namespace. However, this is highly discouraged, and should be used
only for implementing optional proxy-specific functionality, otherwise we’re going to lose
portability.

Function names are in the form of: `proxy_<verb>_<namespace>_<field>_<field>` (e.g.
`proxy_on_http_request_headers`, `proxy_get_map_value`, etc).

## Memory ownership

No memory-managed data is passed to Wasm module in any of the event callbacks, and Wasm module must
request memory-managed data from the host environment if it’s interested in it, e.g.
`proxy_on_http_request_body` only passes the information about the number of available request body
bytes, but the module must request them using `proxy_get_buffer`. When that happens, the host
environment will request the Wasm module to allocate memory using `proxy_on_memory_allocate` and
copy requested data into it. In doing so, it passes memory ownership of that data to the Wasm
module, which is expected to free this memory when it no longer needs it.

SDKs implementing this ABI are expected to automatically handle freeing of the memory-managed data
for the developers.
