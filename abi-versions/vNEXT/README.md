# Proxy-Wasm vNEXT ABI specification

# Callbacks and Functions

### Callbacks optionally exposed by the Wasm module

All callbacks (entry points named `proxy_on_<event>`) are optional, and will be
called only if exposed by the Wasm module.

All callbacks, other than the integration functions, include unique context
identifier as the first parameter, which can be used to distinguish between
different contexts.

### Functions exposed by the host environment

All functions exposed by the host environment return `proxy_result_t`, which
indicates the status of the call (success, invalid memory access, etc.), and
return values are written into memory pointed by `return_<value>` parameters.


## Integration

### Functions exposed by the host environment

#### `proxy_abi_version_X_Y_Z`

* params:
  - none
* returns:
  - none

Exports ABI version (vX.Y.Z) that the Wasm module was compiled against.

#### `proxy_log`

* params:
  - `i32 (proxy_log_level_t) log_level`
  - `i32 (const char*) message_data`
  - `i32 (size_t) message_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Logs message (`message_data`, `message_size`) at the log level (`log_level`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `log_level`.

Returns `InvalidMemoryAccess` when `message_data` and/or `message_size` point to
invalid memory address.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_memory_allocate`

* params:
  - `i32 (size_t) memory_size`
* returns:
  - `i32 (void*) memory_data`

Allocates continuous memory buffer of size `memory_size` using the in-VM memory
allocator and returns address of the memory buffer to the host environment using
`memory_data`. Returning `0` indicates failure.


## Context lifecycle

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_context_create`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) parent_context_id`
  - `i32 (proxy_context_type_t) context_type`
* returns:
  - `i32 (uint32_t) new_context_id`

Called when the host environment creates a new context (`context_id`) of type
`context_type` (e.g. `VmContext`, `PluginContext`, `StreamContext`,
`HttpContext`).

This callback can be used to associate `context_id` with `parent_context_id`,
or when using custom context identifiers, but otherwise the in-VM context can be
implicitly created in `proxy_on_vm_start`, `proxy_on_plugin_start`,
`proxy_on_new_connection` and `proxy_on_http_request_headers` callbacks.

Return value (`new_context_id`) can be used to assign new context identifier
that will be used by the host environment in subsequent calls. Returning `0`
retains the original context identifier.

#### `proxy_on_context_finalize`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (bool) is_done`

Called when the host environment is done processing context (`context_id`).
Any finalization (e.g. logging) should be done within this callback.

Return value (`is_done`) indicates that the Wasm module also finished processing
context (`context_id`). If it didn't, then it should return `false` and call
`proxy_context_finalize` when it's done.

### Functions exposed by the host environment

#### `proxy_set_effective_context`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Change the effective context to `context_id`. This function is usually used to
change effective context during `proxy_on_queue_ready`, `proxy_on_timer_ready`,
`proxy_on_http_call_response` and `proxy_on_grpc_call_response` callbacks.

Returns `Ok` on success.

Returns `BadArgument` for unknown `context_id`.

#### `proxy_context_finalize`

* params:
  - none
* returns:
  - `i32 (proxy_result_t) call_result`

Indicates to the host environment that the Wasm module finished processing
current context. It should be used after previously returning `false` in
`proxy_on_context_finalize`.

Returns `Ok` on success.


## Configuration

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_vm_start`

* params:
  - `i32 (uint32_t) vm_id`
  - `i32 (size_t) vm_configuration_size`
* returns:
  - `i32 (bool) is_success`

Called when the host environment starts WebAssembly Virtual Machine (WasmVm).

Its configuration (of size `vm_configuration_size`) can be retrieved using
`proxy_get_buffer`.

Return value (`is_success`) indicates that configuration was processed correctly
and that WasmVm started successfully.

#### `proxy_on_plugin_start`

* params:
  - `i32 (uint32_t) plugin_id`
  - `i32 (size_t) plugin_configuration_size`
* returns:
  - `i32 (bool) is_success`

Called when the host environment starts plugin.

Its configuration (of size `plugin_configuration_size`) can be retrieved using
`proxy_get_buffer`.

Return value (`is_success`) indicates that configuration was processed correctly
and that plugin started successfully.


## TCP/UDP/QUIC stream (L4) extensions

Note: `downstream` means the connection between client and proxy, and `upstream`
       means the connection between proxy and backend (aka “next hop”).

Note: The same unique context identifier (`stream_id`) is shared by a pair of
      `downstream` and `upstream` connections.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_new_connection`

* params:
  - `i32 (uint32_t) stream_id`
* returns:
  - `i32 (proxy_action_t) next_action`

Called on a new connection (`stream_id`).

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that this connection should be accepted.
- `Done` means that this connection should be accepted and that this extension
  shouldn't be called again for the current stream.
- `Pause` means that further processing should be paused until
  `proxy_resume_stream` is called.
- `Close` means that this connection should be rejected.

#### `proxy_on_downstream_data`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) data_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each data chunk received from downstream (`stream_id`).

Data (of size `data_size`) can be retrieved using `proxy_get_buffer`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that data chunk should be forwarded upstream.
- `EndStream` means that data chunk should be forwarded upstream and that this
  connection should be closed afterwards.
- `Done` means that data chunk should be forwarded upstream and that this
  extension shouldn't be called again for the current stream.
- `Pause` means that further processing should be paused until
  `proxy_resume_stream` is called.
- `WaitForMoreData` means that further processing should be paused and that host
  should call this callback again when it has more data.
- `WaitForEndOrFull` means that further processing should be paused and that
  host should call this callback again when downstream connection is closed or
  when its internal buffers are full.
- `Close` means that this connection should be closed immediately.

#### `proxy_on_downstream_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (proxy_close_source_type_t) close_source`
* returns:
  - none

Called when the downstream connection (`stream_id`) is closed.

#### `proxy_on_upstream_data`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) data_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each data chunk received from upstream (`stream_id`).

Data (of size `data_size`) can be retrieved using `proxy_get_buffer`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that data should be forwarded downstream.
- `EndStream` means that data should be forwarded downstream and that this
  connection should be closed afterwards.
- `Done` means that data should be forwarded downstream and that this extension
  shouldn't be called again for the current stream.
- `Pause` means that further processing should be paused until
  `proxy_resume_stream` is called.
- `WaitForMoreData` means that further processing should be paused and that host
  should call this callback again when it has more data.
- `WaitForEndOrFull` means that further processing should be paused and that
  host should call this callback again when when upstream connection is closed
  or when its internal buffers are full.
- `Close` means that this connection should be closed immediately.

#### `proxy_on_upstream_close`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (proxy_close_source_type_t) close_source`
* returns:
  - none

Called when the upstream connection (`stream_id`) is closed.

### Functions exposed by the host environment

#### `proxy_resume_stream`

* params:
  - `i32 (proxy_stream_type_t) stream_type`
* returns:
  - `i32 (proxy_result_t) call_result`

Resume processing of paused downstream or upstream (`stream_type`) connection
in current context.

Returns `Ok` on success.

Returns `BadArgument` for unknown `stream_type`.

Returns `InvalidOperation` when stream wasn't paused.

#### `proxy_close_stream`

* params:
  - `i32 (proxy_stream_type_t) stream_type`
* returns:
  - `i32 (proxy_result_t) call_result`

Close downstream or upstream (`stream_type`) connections in current context.

Returns `Ok` on success.

Returns `BadArgument` for unknown `stream_type`.

Returns `InvalidOperation` when stream is already closed.


## HTTP (L7) extensions

Note: The same unique context identifier (`stream_id`) is shared by a pair of
      HTTP request and response.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_http_request_headers`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) num_headers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP request headers for stream `stream_id` are received from
downstream.

Headers (`num_headers` entries) can be retrieved using `proxy_get_map` and/or
`proxy_get_map_value`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that this HTTP request should be accepted and that HTTP
  headers should be forwarded upstream.
- `EndStream` means that this HTTP request should be accepted, that HTTP headers
  should be forwarded upstream and that the underlying stream should be closed
  afterwards.
- `Done` means that this HTTP request should be accepted, that HTTP headers
  should be forwarded upstream and that this extension shouldn't be called again
  for this HTTP request.
- `Pause` instructs the host environment to pause processing until
  `proxy_resume` is called.
- `Close` means that the stream should be closed immediately.

#### `proxy_on_http_request_body`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) body_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each chunk of HTTP request body for stream `stream_id` received from
downstream.

Request body (of size `body_size`) can be retrieved using `proxy_get_buffer`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that this HTTP request body chunk should be forwarded
  upstream.
- `EndStream` means that this HTTP request body chunk should be forwarded
  upstream and that the underlying stream should be closed afterwards.
- `Done` means that this HTTP request body should be forwarded upstream and that
  this extension shouldn't be called again for this HTTP request.
- `Pause` means that further processing should be paused until `proxy_resume` is
  called.
- `WaitForMoreData` means that further processing should be paused and host
  should call this callback again when it has more data.
- `WaitForEndOrFull` means that further processing should be paused and host
  should call this callback again when complete HTTP request body is available
  or when its internal buffers are full.
- `Close` means that the stream should be closed immediately.

#### `proxy_on_http_request_trailers`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) num_trailers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP request trailers for stream `stream_id` are received from
downstream.

Trailers (`num_trailers` entries) can be retrieved using `proxy_get_map` and/or
`proxy_get_map_value`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that HTTP trailers should be forwarded upstream.
- `EndStream` means that HTTP trailers should be forwarded upstream and that
  the underlying stream should be closed afterwards.
- `Done` means that HTTP trailers should be forwarded upstream and that this
  extension shouldn't be called again for this HTTP request.
- `Pause` instructs the host environment to pause processing until
  `proxy_resume` is called.
- `Close` means that the underlying stream should be closed immediately.

#### `proxy_on_http_request_metadata`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) num_elements`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each HTTP/2 METADATA frame for stream `stream_id` received from
downstream.

Metadata (`num_elements` entries) can be retrieved using `proxy_get_map` and/or
`proxy_get_map_value`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that HTTP/2 METADATA elements should be forwarded upstream.
 and that
  the underlying stream should be closed afterwards.
- `Done` means that HTTP/2 METADATA elements should be forwarded upstream and
  that this extension shouldn't be called again for this HTTP request.
- `Pause` instructs the host environment to pause processing until
  `proxy_resume` is called.
- `Close` means that the underlying stream should be closed immediately.

#### `proxy_on_http_response_headers`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) num_headers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP response headers for stream `stream_id` are received from
upstream.

Headers (`num_headers` entries) can be retrieved using `proxy_get_map` and/or
`proxy_get_map_value`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that HTTP headers should be forwarded downstream.
- `EndStream` means that HTTP headers should be forwarded downstream and that
  the underlying stream should be closed afterwards.
- `Done` means that HTTP headers should be forwarded downstream and that this
  extension shouldn't be called again for this HTTP response.
- `Pause` instructs the host environment to pause processing until
  `proxy_resume` is called.
- `Close` means that the underlying stream should be closed immediately.

#### `proxy_on_http_response_body`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) body_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each chunk of HTTP response body for stream `stream_id` received from
upstream.

Response body (of size `body_size`) can be retrieved using `proxy_get_buffer`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that this HTTP response body chunk should be forwarded
  downstream.
- `EndStream` means that this HTTP response body chunk should be forwarded
  downstream and that the underlying stream should be closed afterwards.
- `Done` means that this HTTP response body chunk should be forwarded downstream
  and that this extension shouldn't be called again for this HTTP response.
- `Pause` means that further processing should be paused until `proxy_resume` is
  called.
- `WaitForMoreData` means that further processing should be paused and host
  should call this callback again when it has more data.
- `WaitForEndOrFull` means that further processing should be paused and host
  should call this callback again when complete HTTP response body is available
  or when its internal buffers are full.
- `Close` means that the underlying stream should be closed immediately.

#### `proxy_on_http_response_trailers`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) num_trailers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP response trailers for stream `stream_id` are received from
upstream.

Trailers (`num_trailers` entries) can be retrieved using `proxy_get_map` and/or
`proxy_get_map_value`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that HTTP trailers should be forwarded downstream.
- `EndStream` means that HTTP trailers should be forwarded downstream and that
  the underlying stream should be closed afterwards.
- `Done` means that HTTP trailers should be forwarded downstream and that this
  extension shouldn't be called again for this HTTP response.
- `Pause` instructs the host environment to pause processing until
  `proxy_resume` is called.
- `Close` means that the underlying stream should be closed immediately.

#### `proxy_on_http_response_metadata`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (size_t) num_elements`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each HTTP/2 METADATA frame for stream `stream_id` received from
upstream.

Metadata (`num_elements` entries) can be retrieved using `proxy_get_map` and/or
`proxy_get_map_value`.

Return value (`next_action`) instructs the host environment what action to take:
- `Continue` means that HTTP/2 METADATA elements should be forwarded downstream.
- `EndStream` means that HTTP/2 METADATA elements should be forwarded downstream
  and that the underlying stream should be closed afterwards.
- `Done` means that HTTP/2 METADATA elements should be forwarded downstream and
  that this extension shouldn't be called again for this HTTP response.
- `Pause` instructs the host environment to pause processing until
  `proxy_resume` is called.
- `Close` means that the underlying stream should be closed immediately.

### Functions exposed by the host environment

#### `proxy_send_http_response`

* params:
  - `i32 (uint32_t) response_code`
  - `i32 (const char*) response_code_details_data`
  - `i32 (size_t) response_code_details_size`
  - `i32 (const char*) response_body_data`
  - `i32 (size_t) response_body_size`
  - `i32 (const char*) additional_headers_map_data`
  - `i32 (size_t) additional_headers_size`
  - `i32 (uint32_t) grpc_status`
* returns:
  - `i32 (proxy_result_t) call_result`

Sends HTTP response to the client, without forwarding request to the upstream.

Returns `Ok` on success.

Returns `BadArgument` for invalid `response_code` and/or `grpc_status`.

Returns `InvalidMemoryAccess` when `response_code_details_data`,
`response_code_details_size`, `response_body_data`, `response_body_size`,
`additional_headers_map_data` and/or `additional_headers_size` point to invalid
memory address.

#### `proxy_resume_http_stream`

* params:
  - `i32 (proxy_stream_type_t) stream_type`
* returns:
  - `i32 (proxy_result_t) call_result`

Resume processing of paused HTTP request or response (`stream_type`) in current
context.

Returns `Ok` on success.

Returns `BadArgument` for unknown `stream_type`.

Returns `InvalidOperation` when stream wasn't paused.

#### `proxy_close_http_stream`

* params:
  - `i32 (proxy_stream_type_t) stream_type`
* returns:
  - `i32 (proxy_result_t) call_result`

Close HTTP request or response (`stream_type`) in current context.

Returns `Ok` on success.

Returns `BadArgument` for unknown `stream_type`.

Returns `InvalidOperation` when stream is already closed.


## Buffers and maps

### Functions exposed by the host environment

#### `proxy_get_buffer`

* params:
  - `i32 (proxy_buffer_type_t) buffer_type`
  - `i32 (size_t) offset`
  - `i32 (size_t) max_size`
  - `i32 (const char**) return_buffer_data`
  - `i32 (size_t*) return_buffer_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Returns content of the buffer `buffer_type`. At most `max_size` bytes will be
returned, starting from the `offset`. Returned bytes are written into provided
memory slice (`return_buffer_data`, `return_buffer_size`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `buffer_type`.

Returns `Empty` when `buffer_type` doesn't contain any data.

Returns `InvalidMemoryAccess` when `return_buffer_data` and/or
`return_buffer_size` point to invalid memory address.

#### `proxy_set_buffer`

* params:
  - `i32 (proxy_buffer_type_t) buffer_type`
  - `i32 (offset_t) offset`
  - `i32 (size_t) size`
  - `i32 (const char*) buffer_data`
  - `i32 (size_t) buffer_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Sets content of the buffer `buffer_type` to the provided value (`buffer_data`,
`buffer_size`), replacing `size` bytes in the existing buffer, starting at the
`offset`. Host environment must adjust the size of the destination buffer
(e.g. shrink the buffer if target `size` is larger than `buffer_size`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `buffer_type`.

Returns `InvalidMemoryAccess` when `buffer_data` and/or `buffer_size` point to
invalid memory address.

#### `proxy_get_map`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char**) return_map_data`
  - `i32 (size_t*) return_map_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Returns all key-value pairs from the given map (`map_type`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `map_type`.

Returns `Empty` when `map_type` doesn't contain any data.

Returns `InvalidMemoryAccess` when `return_map_data` and/or `return_map_size`
point to invalid memory address.

#### `proxy_set_map`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) map_data`
  - `i32 (size_t) map_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Sets all key-value pairs in the given map (`map_type`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `map_type`.

Returns `InvalidMemoryAccess` when `map_data` and/or `map_size` point to invalid
memory address.

#### `proxy_get_map_values`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char**) return_values_data`
  - `i32 (size_t*) return_values_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Returns content of the key (`key_data`, `key_size`) from the map (`map_type`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `map_type`.

Returns `NotFound` when key (`key_data`, `key_size`) doesn't exist.

Returns `InvalidMemoryAccess` when `key_data`, `key_size`, `return_values_data`
and/or `return_values_size` point to invalid memory address.

#### `proxy_set_map_key_values`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) values_data`
  - `i32 (size_t) values_size`
  - `i32 (bool) multiple_values`
* returns:
  - `i32 (proxy_result_t) call_result`

Sets content of the key (`key_data`, `key_size`) in the map (`map_type`) to the
provided values (`values_data`, `values_size`).

If `multiple_values` is `false`, then `values_data`, `values_size` contains
a single value, otherwise it contains multiple values serialized as a list.

Returns `Ok` on success.

Returns `BadArgument` for unknown `map_type`.

Returns `InvalidMemoryAccess` when `key_data`, `key_size`, `values_data` and/or
`values_size` point to invalid memory address.

#### `proxy_add_map_key_values`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) values_data`
  - `i32 (size_t) values_size`
  - `i32 (bool) multiple_values`
* returns:
  - `i32 (proxy_result_t) call_result`

Adds new values (`values_data`, `values_size`) to the key (`key_data`,
`key_size`) in the map (`map_type`).

If `multiple_values` is `false`, then `values_data`, `values_size` contains
a single value, otherwise it contains multiple values serialized as a list.

Returns `Ok` on success.

Returns `BadArgument` for unknown `map_type`.

Returns `InvalidMemoryAccess` when `key_data`, `key_size`, `values_data` and/or
`values_size` point to invalid memory address.

#### `proxy_remove_map_key`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Removes the key (`key_data`, `key_size`) from the map (`map_type`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `map_type`.

Returns `InvalidMemoryAccess` when `key_data` and/or `key_size` point to invalid
memory address.


## Shared key-value store

Note: Global key-value store can be always accessed using `kvstore_id` `0`.

### Functions exposed by the host environment

#### `proxy_open_shared_kvstore`

* params:
  - `i32 (const char*) kvstore_name_data`
  - `i32 (size_t) kvstore_name_size`
  - `i32 (bool) create_if_not_exist`
  - `i32 (uint32_t*) return_kvstore_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Opens named key-value store (`kvstore_name_data`, `kvstore_name_size`).

If `create_if_not_exist` is `true` and there is no shared key-value store with
that name, then it will be created.

Key's value can be set using `proxy_set_shared_kvstore_key_values` and retrieved
using `proxy_get_shared_kvstore_key_values` from the key-value store using
returned unique key-value store identifier (`return_kvstore_id`).

Returns `Ok` on success.

Returns `NotFound` when `create_if_not_exist` is `false` and no shared key-value
store with the given name exists.

Returns `InvalidMemoryAccess` when `kvstore_name_data`, `kvstore_name_size`
and/or `return_kvstore_id` point to invalid memory address.

#### `proxy_get_shared_kvstore_key_values`

* params:
  - `i32 (uint32_t) kvstore_id`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char**) return_values_data`
  - `i32 (size_t*) return_values_size`
  - `i32 (uint32_t*) return_cas`
* returns:
  - `i32 (proxy_result_t) call_result`

Returns value of the key (`key_data`, `key_size`) from a shared key-value store
(`kvstore_id`). Returned values are written into (`return_values_data`,
`return_values_size`). The compare-and-switch value (`return_cas`) can be used
when updating key's values with `proxy_set_shared_kvstore_key_values` and
`proxy_add_shared_kvstore_key_values` and `proxy_remove_shared_kvstore_key`.

Returns `Ok` on success.

Returns `BadArgument` when the key-value store (`kvstore_id`) doesn't exist.

Returns `NotFound` when the key (`key_data`, `key_size`) doesn't exist.

Returns `InvalidMemoryAccess` when `key_data`, `key_size`, `return_values_data`,
`return_values_size` and/or `return_cas` point to invalid memory address.

#### `proxy_set_shared_kvstore_key_values`

* params:
  - `i32 (uint32_t) kvstore_id`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) values_data`
  - `i32 (size_t) values_size`
  - `i32 (bool) multiple_values`
  - `i32 (uint32_t) cas`
* returns:
  - `i32 (proxy_result_t) call_result`

Sets the values of the key (`key_data`, `key_size`) in a shared key-value store
(`kvstore_id`) to the values (`values_data`, `values_size`).

If `multiple_values` is `false`, then `values_data`, `values_size` contains
a single value, otherwise it contains multiple values serialized as a list.

If compare-and-switch value (`cas`) is provided and non-zero, then it must match
the current value in order for update to succeed.

Returns `Ok` on success.

Returns `BadArgument` when the key-value store (`kvstore_id`) doesn't exist.

Returns `CompareAndSwapMismatch` when the compare-and-switch value isn't
current.

Returns `InvalidMemoryAccess` when `key_data`, `key_size`, `values_data` and/or
`values_size` point to invalid memory address.

#### `proxy_add_shared_kvstore_key_values`

* params:
  - `i32 (uint32_t) kvstore_id`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) values_data`
  - `i32 (size_t) values_size`
  - `i32 (bool) multiple_values`
  - `i32 (uint32_t) cas`
* returns:
  - `i32 (proxy_result_t) call_result`

Adds new values (`values_data`, `values_size`) to the key (`key_data`,
`key_size`) in a shared key-value store (`kvstore_id`).

If `multiple_values` is `false`, then `values_data`, `values_size` contains
a single value, otherwise it contains multiple values serialized as a list.

If compare-and-switch value (`cas`) is provided and non-zero, then it must match
the current value in order for update to succeed.

Returns `Ok` on success.

Returns `BadArgument` when the key-value store (`kvstore_id`) doesn't exist.

Returns `CompareAndSwapMismatch` when the compare-and-switch value isn't
current.

Returns `InvalidMemoryAccess` when `key_data`, `key_size`, `values_data` and/or
`values_size` point to invalid memory address.

#### `proxy_remove_shared_kvstore_key`

* params:
  - `i32 (uint32_t) kvstore_id`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (uint32_t) cas`
* returns:
  - `i32 (proxy_result_t) call_result`

Removes the key (`key_data`, `key_size`) from a shared key-value store
(`kvstore_id`).

If compare-and-switch value (`cas`) is provided and non-zero, then it must match
the current value in order for update to succeed.

Returns `Ok` on success.

Returns `BadArgument` when the key-value store (`kvstore_id`) doesn't exist.

Returns `CompareAndSwapMismatch` when the compare-and-switch value isn't
current.

Returns `InvalidMemoryAccess` when `key_data` and/or `key_size` point to invalid
memory address.

#### `proxy_delete_shared_kvstore`

* params:
  - `i32 (uint32_t) kvstore_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Deletes previously created shared key-value store (`kvstore_id`).

Returns `Ok` on success.

Returns `BadArgument` when the key-value store (`kvstore_id`) doesn't exist.


## Shared queue

Note: Global queue can be always accessed using `queue_id` `0`.

### Functions exposed by the host environment

#### `proxy_open_shared_queue`

* params:
  - `i32 (const char*) queue_name_data`
  - `i32 (size_t) queue_name_size`
  - `i32 (bool) create_if_not_exist`
  - `i32 (uint32_t*) return_queue_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Opens shared queue with the given name (`queue_name_data`, `queue_name_size`).
If `create_if_not_exist` is `true` and there is no shared queue with that name,
then it will be created.

Items can be queued using `proxy_enqueue_shared_queue_item` and dequeued using
`proxy_dequeue_shared_queue_item` from the queue using returned unique queue
identifier (`return_queue_id`).

Returns `Ok` on success.

Returns `NotFound` when `create_if_not_exist` is `false` and no shared queue
with the given name exists.

Returns `InvalidMemoryAccess` when `queue_name_data`, `queue_name_size` and/or
`return_queue_id` point to invalid memory address.

#### `proxy_dequeue_shared_queue_item`

* params:
  - `i32 (uint32_t) queue_id`
  - `i32 (const char**) return_payload_data`
  - `i32 (size_t*) return_payload_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Removes item from the tail of the queue (`queue_id`). Returned value is written
into memory slice (`payload_data`, `payload_size`).

Returns `Ok` on success.

Returns `BadArgument` when the queue (`queue_id`) doesn't exist.

Returns `InvalidMemoryAccess` when `return_payload_data` and/or
`return_payload_size` point to invalid memory address.

#### `proxy_enqueue_shared_queue_item`

* params:
  - `i32 (uint32_t) queue_id`
  - `i32 (const char*) payload_data`
  - `i32 (size_t) payload_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Adds item (`payload_data`, `payload_size`) at the front of the queue
(`queue_id`).

Returns `Ok` on success.

Returns `BadArgument` when the queue (`queue_id`) doesn't exist.

Returns `InvalidMemoryAccess` when `payload_data` and/or `payload_size` point to
invalid memory address.

#### `proxy_delete_shared_queue`

* params:
  - `i32 (uint32_t) queue_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Deletes previously created shared queue (`queue_id`).

Returns `Ok` on success.

Returns `BadArgument` when the queue (`queue_id`) doesn't exist.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_queue_ready`

* params:
  - `i32 (uint32_t) queue_id`
* returns:
  - none

Called when new item is available in the queue (`queue_id`).


## Timers

### Functions exposed by the host environment

#### `proxy_create_timer`

* params:
  - `i32 (uint32_t) period`
  - `i32 (bool) one_time`
  - `i32 (uint32_t*) return_timer_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Creates a new timer. When a timer is created as a one-time alarm (`one_time`),
then callback will be called only once after `period` milliseconds. Otherwise,
the callback is going to be called every `period` milliseconds until the timer
is deleted using `proxy_delete_timer` with the returned unique timer identifier
(`return_timer_id`).

Returns `Ok` on success.

Returns `InvalidMemoryAccess` when `return_timer_id` points to invalid memory
address.

#### `proxy_delete_timer`

* params:
  - `i32 (uint32_t) timer_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Deletes previously created timer (`timer_id`).

Returns `Ok` on success.

Returns `BadArgument` when the timer (`timer_id`) doesn't exist.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_timer_ready`

* params:
  - `i32 (uint32_t) timer_id`
* returns:
  - none

Called after configured period by the timer (`timer_id`) created using
`proxy_create_timer`.


## Stats/metrics

### Functions exposed by the host environment

#### `proxy_create_metric`

* params:
  - `i32 (proxy_metric_type_t) metric_type`
  - `i32 (const char*) metric_name_data`
  - `i32 (size_t) metric_name_size`
  - `i32 (uint32_t*) return_metric_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Creates metric (`metric_name_data`, `metic_name_size`) of type `metric_type`.

It can be referred to in `proxy_get_metric_value`, `proxy_set_metric_value`,
and `proxy_increment_metric_value` using returned unique metric identifier
(`metric_id`).

Returns `Ok` on success.

Returns `BadArgument` for unknown `metric_type`.

Returns `InvalidMemoryAccess` when `metric_name_data`, `metric_name_size`
and/or `return_metric_id` point to invalid memory address.

#### `proxy_get_metric_value`

* params:
  - `i32 (uint32_t) metric_id`
  - `i32 (uint64_t*) return_value`
* returns:
  - `i32 (proxy_result_t) call_result`

Get the value of the metric (`metric_id`).

Returns `Ok` on success.

Returns `BadArgument` when the metric (`metric_id`) doesn't exist.

#### `proxy_set_metric_value`

* params:
  - `i32 (uint32_t) metric_id`
  - `i64 (uint64_t) value`
* returns:
  - `i32 (proxy_result_t) call_result`

Set the value of the metric (`metric_id`) to the value (`value`).

Returns `Ok` on success.

Returns `BadArgument` when the metric (`metric_id`) doesn't exist.

Returns `InvalidOperation` when the `metric_type` doesn't support setting
values.

#### `proxy_increment_metric_value`

* params:
  - `i32 (uint32_t) metric_id`
  - `i64 (int64_t) offset`
* returns:
  - `i32 (proxy_result_t) call_result`

Increment/decrement value of the metric (`metric_id`) by offset (`offset`).

Returns `Ok` on success.

Returns `BadArgument` when the metric (`metric_id`) doesn't exist.

Returns `InvalidOperation` when `metric_type` doesn't support incrementing
and/or decrementing values.

#### `proxy_delete_metric`

* params:
  - `i32 (uint32_t) metric_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Deletes previously created metric (`metric_id`).

Returns `Ok` on success.

Returns `BadArgument` when the metric (`metric_id`) doesn't exist.


## HTTP callouts

### Functions exposed by the host environment

#### `proxy_dispatch_http_call`

* params:
  - `i32 (const char*) upstream_name_data`
  - `i32 (size_t) upstream_name_size`
  - `i32 (const char*) headers_map_data`
  - `i32 (size_t) headers_map_size`
  - `i32 (const char*) body_data`
  - `i32 (size_t) body_size`
  - `i32 (const char*) trailers_map_data`
  - `i32 (size_t) trailers_map_size`
  - `i32 (uint32_t) timeout_milliseconds`
  - `i32 (uint32_t*) return_callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Dispatches a HTTP call to upstream (`upstream_name_data`, `upstream_name_size`).

Once the response is returned to the host, `proxy_on_http_call_response` will be
called with an unique call identifier (`return_callout_id`).

Returns `Ok` on success.

Returns `NotFound` for unknown upstream.

Returns `NotAllowed` when requests to given upstream are not allowed.

Returns `InvalidMemoryAccess` when `upstream_name_data`, `upstream_name_size`,
`headers_map_data`, `headers_map_size`, `body_data`, `body_size`,
`trailers_map_data`, `trailers_map_size` and/or `return_callout_id` point to
invalid memory address.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_http_call_response`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) num_headers`
  - `i32 (size_t) body_size`
  - `i32 (size_t) num_trailers`
* returns:
  - none

Called when the complete response to the HTTP call (`callout_id`) is received.

Headers (`num_headers` entries) and trailers (`num_trailers` entries) can be
retrieved using `proxy_get_map` and/or `proxy_get_map_value`.

Response body (of size `body_size`) can be retrieved using `proxy_get_buffer`.


## gRPC callouts

### Functions exposed by the host environment

#### `proxy_dispatch_grpc_call`

* params:
  - `i32 (const char*) grpc_service_data`
  - `i32 (size_t) grpc_service_size`
  - `i32 (const char*) service_name_data`
  - `i32 (size_t) service_name_size`
  - `i32 (const char*) method_data`
  - `i32 (size_t) method_size`
  - `i32 (const char*) initial_metadata_map_data`
  - `i32 (size_t) initial_metadata_map_size`
  - `i32 (const char*) grpc_message_data`
  - `i32 (size_t) grpc_message_size`
  - `i32 (uint32_t) timeout_milliseconds`
  - `i32 (uint32_t*) return_callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Dispatch a gRPC call to a service (`service_name_data`, `service_name_size`).

Once the response is returned to the host, `proxy_on_grpc_call_response` and/or
`proxy_on_grpc_call_close` will be called with an unique call identifier
(`return_callout_id`). The unique call identifier can also be used to cancel
outstanding requests using `proxy_cancel_grpc_call` or close outstanding request
using or `proxy_close_grpc_call`.

Returns `Ok` on success.

Returns `NotFound` for unknown service (`service_name_data`,
`service_name_size`).

Returns `NotAllowed` when requests to given service are not allowed.

Returns `InvalidMemoryAccess` when `grpc_service_data`, `grpc_service_size`,
`service_name_data`, `service_name_size`, `method_data`, `method_size`,
`initial_metadata_map_data`, `initial_metadata_map_size`, `grpc_message_data`,
`grpc_message_size` and/or `return_callout_id` point to invalid memory address.

#### `proxy_open_grpc_stream`

* params:
  - `i32 (const char*) grpc_service_data`
  - `i32 (size_t) grpc_service_size`
  - `i32 (const char*) service_name_data`
  - `i32 (size_t) service_name_size`
  - `i32 (const char*) method_data`
  - `i32 (size_t) method_size`
  - `i32 (const char*) initial_metadata_map_data`
  - `i32 (size_t) initial_metadata_map_size`
  - `i32 (uint32_t*) return_callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Opens connection to the gRPC service (`service_name_data`, `service_name_size`).

Returns `Ok` on success.

Returns `NotFound` for unknown service (`service_name_data`,
`service_name_size`).

Returns `NotAllowed` when requests to given service are not allowed.

Returns `InvalidMemoryAccess` when `grpc_service_data`, `grpc_service_size`,
`service_name_data`, `service_name_size`, `method_data`, `method_size`,
`initial_metadata_map_data`, `initial_metadata_map_size` and/or
`return_callout_id` point to invalid memory address.

#### `proxy_send_grpc_stream_message`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (const char*) grpc_message_data`
  - `i32 (size_t) grpc_message_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Send gRPC message (`grpc_message_data`, `grpc_message_size`) on the existing
gRPC stream (`callout_id`).

Returns `Ok` on success.

Returns `BadArgument` when the gRPC call (`callout_id`) doesn't exist.

Returns `InvalidMemoryAccess` when `grpc_message_data` and/or
`grpc_message_size` point to invalid memory address.

#### `proxy_cancel_grpc_call`

* params:
  - `i32 (uint32_t) callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Cancel outstanding gRPC request or stream (`callout_id`).

Returns `Ok` on success.

Returns `BadArgument` when the gRPC call (`callout_id`) doesn't exist.

#### `proxy_close_grpc_call`

* params:
  - `i32 (uint32_t) callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Cancel outstanding gRPC request or close existing gRPC stream (`callout_id`).

Returns `Ok` on success.

Returns `BadArgument` when the gRPC call (`callout_id`) doesn't exist.

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_grpc_call_response_header_metadata`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) num_elements`
* returns:
  - none

Called when header metadata in the response to the gRPC call (`callout_id`) is
received.

Header metadata (`num_headers` entries) can be retrieved using `proxy_get_map`
and/or `proxy_get_map_value`.

#### `proxy_on_grpc_call_response_message`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) message_size`
* returns:
  - none

Called when the response message (of size `message_size`) to the gRPC call
(`callout_id`) is received.

Content of the gRPC message (of size `data_size`) can be retrieved using
`proxy_get_buffer`.

#### `proxy_on_grpc_call_response_trailer_metadata`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) num_elements`
* returns:
  - none

Called when trailer metadata in the response to the gRPC call (`callout_id`) is
received.

Trailer metadata (`num_headers` entries) can be retrieved using `proxy_get_map`
and/or `proxy_get_map_value`.

#### `proxy_on_grpc_call_close`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (uint32_t) status_code`
* returns:
  - none

Called when the request to the gRPC call (`callout_id`) fails or if the gRPC
connection is closed.


## Custom extension points

### Callbacks optionally exposed by the Wasm module

#### `proxy_on_custom_callback`

* params:
  - `i32 (uint32_t) custom_callback_id`
  - `i32 (size_t) parameters_size`
* returns:
  - `i32 (uint32_t) custom_return_code`

Called for a well-known custom callback (`custom_callback_id`).

Callback parameters (of size `parameters_size`) can be retrieved using
`proxy_get_buffer`.

Return value (`custom_return_code`) is defined as part of specification for each
custom callback. Returning `0` indicates unknown `custom_callback_id`.

### Functions exposed by the host environment

#### `proxy_call_custom_function`

* params:
  - `i32 (uint32_t) custom_function_id`
  - `i32 (const char *) parameters_data`
  - `i32 (size_t) parameters_size`
  - `i32 (const char**) return_results_data`
  - `i32 (size_t*) return_results_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Calls a well-known custom function (`custom_function_id`) with provided
parameters (`parameters_data`, `parameters_size`).

Returns `Ok` on success.

Returns `NotFound` for unknown `function_id`.

Returns `InvalidMemoryAccess` when `parameters_data`, `parameters_size`,
`return_results_data` and/or `return_results_size` point to invalid memory
address.


# Serialization

The encoding of integers is little endian.

### List of values.

List of values is serialized as: 32-bit number of elements in the list,
followed by a series of 32-bit lengths of each value, followed by a series of
payload of each value.

e.g. `{"a", "b"}` is serialized as: `{ 0x2, 0x1, 0x1, "a", "b" }`.

### Map with multiple values.

Map with multiple values is serialized as: 32-bit number of keys in the map,
followed by a series of pairs of 32-bit lengths of each key and its serialized
list of values, followed by a series of key names and its serialized list of
values.

e.g. `{"key": {"a", "b"}}` is serialized as:
`{ 0x1, 0x3, 0x5, "k", "e", "y", 0x2, 0x1, 0x1, "a", "b" }`.


# Built-in types

### `proxy_result_t`

* `0` - `Ok`
* `1` - `Empty`
* `2` - `NotFound`
* `3` - `NotAllowed`
* `4` - `BadArgument`
* `5` - `InvalidMemoryAccess`
* `6` - `InvalidOperation`
* `7` - `CompareAndSwapMismatch`

### `proxy_action_t`

* `1` - `Continue`
* `2` - `EndStream`
* `3` - `Done`
* `4` - `Pause`
* `5` - `WaitForMoreData`
* `6` - `WaitForEndOrFull`
* `7` - `Close`

### `proxy_log_level_t`

* `1` - `Trace`
* `2` - `Debug`
* `3` - `Info`
* `4` - `Warning`
* `5` - `Error`

### `proxy_context_type_t`

* `1` - `VmContext`
* `2` - `PluginContext`
* `3` - `StreamContext`
* `4` - `HttpContext`

### `proxy_stream_type_t`

* `1` - `Downstream`
* `2` - `Upstream`
* `3` - `HttpRequest`
* `4` - `HttpResponse`

### `proxy_close_source_type_t`

* `1` - `Local`
* `2` - `Remote`

### `proxy_buffer_type_t`

* `1` - `VmConfiguration`
* `2` - `PluginConfiguration`
* `3` - `DownstreamData`
* `4` - `UpstreamData`
* `5` - `HttpRequestBody`
* `6` - `HttpResponseBody`
* `7` - `HttpCalloutResponseBody`

### `proxy_map_type_t`

* `1` - `HttpRequestHeaders`
* `2` - `HttpRequestTrailers`
* `3` - `HttpRequestMetadata`
* `4` - `HttpResponseHeaders`
* `5` - `HttpResponseTrailers`
* `6` - `HttpResponseMetadata`
* `7` - `HttpCallResponseHeaders`
* `8` - `HttpCallResponseTrailers`
* `9` - `HttpCallResponseMetadata`

### `proxy_metric_type_t`

* `1` - `Counter`
* `2` - `Gauge`
* `3` - `Histogram`
