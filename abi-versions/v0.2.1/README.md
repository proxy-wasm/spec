# Proxy-Wasm ABI v0.2.1 specification

---

Status: **FINAL**

---

# Callbacks and Functions

### Callbacks exposed by the Wasm module

All callbacks (entry points named `proxy_on_<event>`) are optional,
and are going to be called only if exposed by the Wasm module.

All callbacks, other than the [integration] and [memory management]
functions, include a unique context identifier as the first parameter,
which can be used to distinguish between different contexts.


### Functions exposed by the host environment

All Proxy-Wasm functions exposed by the host return [`proxy_status_t`],
which indicates status of the call (success, invalid memory access,
etc.). Return values are written into memory pointed by `return_<value>`
parameters.


## Integration

### `proxy_abi_version_0_2_1`

* params:
  - none
* returns:
  - none

Function marker used during linking to advertise support for
Proxy-Wasm ABI v0.2.1.

This function is never called.


### `_initialize`

* params:
  - none
* returns:
  - none

Called when the Wasm module is first loaded.


### `main`

* params:
  - `i32 (uint32_t) ignored`
  - `i32 (uint32_t) ignored`
* returns:
  - `i32 (uint32_t) ignored`

> **Warning**
> This is called only if [`_initialize`] is also exported.

Called when the Wasm module is first loaded, after [`_initialize`].


### `_start`

* params:
  - none
* returns:
  - none

> **Warning**
> This is called only if [`_initialize`] is not exported.

Called when the Wasm module is first loaded.


## Memory management

### `proxy_on_memory_allocate`

* params:
  - `i32 (size_t) memory_size`
* returns:
  - `i32 (uint8_t *) memory_data`

Called to allocate continuous memory buffer of `memory_size` using
the in-VM memory allocator.

Plugin must return `memory_data` pointing to the start of the allocated
memory.

Returning `0` indicates failure.


### `malloc`

* params:
  - `i32 (size_t) memory_size`
* returns:
  - `i32 (uint8_t *) memory_data`

> **Warning**
> This callback has been deprecated in favor of [`proxy_on_memory_allocate`],
> and it's called only in its absence.

Called to allocate continuous memory buffer of `memory_size` using
the in-VM memory allocator.

Plugin must return `memory_data` pointing to the start of the allocated
memory.

Returning `0` indicates failure.


## Context lifecycle

### `proxy_on_context_create`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) parent_context_id`
* returns:
  - none

Called when the host creates a new context (`context_id`).

When `parent_context_id` is `0` then a new plugin ("root") context
is created, otherwise a new per-stream context is created.


### `proxy_on_done`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (bool) completed`

Called when the host is done processing context (`context_id`).

Plugin must return one of the following values:
- `true` to allow the host to finalize and delete context.
- `false` to indicate that the context is still being used,
  and that plugin is going to call [`proxy_done`] later to
  allow the host to finalize and delete that context.


### `proxy_on_log`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - none

Called after the host is done processing context, but before
releasing its state.

This callback can be used for generating final log entries.


### `proxy_on_delete`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - none

Called when the host removes the context (`context_id`).

This is used to signal that the plugin should stop tracking active
context and remove all associated state.


### `proxy_done`

* params:
  - none
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Indicates to the host that the plugin is done processing active
context.

This should be used after returning `false` in [`proxy_on_done`].

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when active context was not pending finalization.


### `proxy_set_effective_context`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Changes the effective context to `context_id`.

This can be used to change active context, e.g. during
[`proxy_on_http_call_response`], [`proxy_on_grpc_receive`]
and/or [`proxy_on_queue_ready`] callbacks.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `context_id`.


## Configuration

### `proxy_on_vm_start`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) vm_configuration_size`
* returns:
  - `i32 (bool) status`

Called when the host starts the WebAssembly Virtual Machine.

Its configuration (of `vm_configuration_size`) can be retrieved using
[`proxy_get_buffer_bytes`] with `buffer_id` set to `VM_CONFIGURATION`.

Plugin must return one of the following values:
- `true` to indicate that the configuration was processed successfully.
- `false` to indicate that the configuration processing failed.


### `proxy_on_configure`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) plugin_configuration_size`
* returns:
  - `i32 (bool) status`

Called when the host starts the Proxy-Wasm plugin.

Its configuration (of `plugin_configuration_size`) can be retrieved
using [`proxy_get_buffer_bytes`] with `buffer_id` set to
`PLUGIN_CONFIGURATION`.

Plugin must return one of the following values:
- `true` to indicate that the configuration was processed successfully.
- `false` to indicate that the configuration processing failed.


## Logging

### `proxy_log`

* params:
  - `i32 (`[`proxy_log_level_t`]`) log_level`
  - `i32 (const char *) message_data`
  - `i32 (size_t) message_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Logs `message` at the `log_level`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `log_level`.
- `INVALID_MEMORY_ACCESS` when `message_data` and/or `message_size`
  point to invalid memory address.


### `wasi_snapshot_preview1.fd_write`

* params:
  - `i32 (`[`wasi_fd_id_t`]`) fd_id`
  - `i32 (wasi_iovec_array_t) iovec`
  - `i32 (size_t) iovec_size`
  - `i32 (size_t *) return_written_bytes`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

Logs message passed in `iovec`.

When `fd_id` is `STDOUT`, then the message is going to be logged
at the `INFO` level.

When `fd_id` is `STDERR`, then the message is going to be logged
at the `ERROR` level.

Returned `errno` value is:
- `SUCCESS` on success.
- `BADF` for unknown or unsupported `fd_id`.
- `FAULT` when `iovec`, `iovec_size` and/or `return_written_bytes`
  point to invalid memory address.


### `proxy_get_log_level`

* params:
  - `i32 (`[`proxy_log_level_t`]` *) return_log_level`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves host's current log level (`return_log_level`).

This can be used to avoid creating log entries that are going to be
discarded by the host.

Returned `status` value is:
- `OK` on success.
- `INVALID_MEMORY_ACCESS` when `return_log_level` points to invalid
  memory address.


## Clocks

### `proxy_get_current_time_nanoseconds`

* params:
  - `i32 (uint64_t *) return_time`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

> **Warning**
> This function has been deprecated in favor of
> [`wasi_snapshot_preview1.clock_time_get`].

Retrieves current time (`return_time`), or the approximation of it.

> **Note**
> Hosts might return approximate time (e.g. frozen at the context
> creation) to improve performance and/or prevent various attacks,
> so consumers shouldn't assume that time changes during callbacks.

Returned `status` value is:
- `OK` on success.
- `INVALID_MEMORY_ACCESS` when `return_time` points to invalid memory
  address.


### `wasi_snapshot_preview1.clock_time_get`

* params:
  - `i32 (`[`wasi_clock_id_t`]`) clock_id`
  - `i64 (uint64_t) ignored`
  - `i32 (uint64_t *) return_time`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

Retrieves current time (`return_time`), or the approximation of it.

> **Note**
> Hosts might return approximate time (e.g. frozen at the context
> creation) to improve performance and/or prevent various attacks,
> so consumers shouldn't assume that time changes during callbacks.

Returned `errno` value is:
- `SUCCESS` on success.
- `NOTSUP` for unknown or unsupported `clock_id`.
- `FAULT` when `return_time` points to invalid memory address.


## Timers

### `proxy_set_tick_period_milliseconds`

* params:
  - `i32 (uint32_t) tick_period`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sets timer period (`tick_period`). When set, the host will call
[`proxy_on_tick`] every `tick_period` milliseconds.

Returned `status` value is:
- `OK` on success.


### `proxy_on_tick`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - none

Called on a timer every tick period.

The tick period can be configured using
[`proxy_set_tick_period_milliseconds`].


## Randomness

### `wasi_snapshot_preview1.random_get`

* params:
  - `i32 (uint8_t *) buffer`
  - `i32 (size_t) buffer_size`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

Generates `buffer_size` bytes of randomness.

Returned `errno` value is:
- `SUCCESS` on success.
- `INVAL` when requested `buffer_size` is too large.
- `FAULT` when `buffer` and/or `buffer_size` point to invalid memory
  address.


## Environment variables

> **Warning**
> Environment variables should be configured for each plugin.
> Exposing the host's environment variables is highly discouraged.


### `wasi_snapshot_preview1.environ_sizes_get`

* params:
  - `i32 (size_t *) return_num_elements`
  - `i32 (size_t *) return_buffer_size`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

Returns `return_num_elements` of environment variables, and
`return_buffer_size` sufficient to serialize them and decorators.

Returned `errno` value is:
- `SUCCESS` on success.
- `FAULT` when `return_num_elements` and/or `return_buffer_size`
  point to invalid memory address.


### `wasi_snapshot_preview1.environ_get`

* params:
  - `i32 (uint8_t **) return_array`
  - `i32 (uint8_t *) return_buffer`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

Returns serialized environment variables.

Returned `errno` value is:
- `SUCCESS` on success.
- `FAULT` when `return_array` and/or `return_buffer` point to
  invalid memory address.


## Buffers

### `proxy_set_buffer_bytes`

* params:
  - `i32 (`[`proxy_buffer_type_t`]`) buffer_id`
  - `i32 (uint32_t) offset`
  - `i32 (uint32_t) size`
  - `i32 (const uint8_t *) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sets content of the buffer `buffer_id` to the `value`, replacing
`size` bytes, starting at `offset` in the existing buffer.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `buffer_id`.
- `NOT_FOUND` when requested `buffer_id` isn't available.
- `INVALID_MEMORY_ACCESS` when `value_data` and/or `value_size`
  point to invalid memory address.


### `proxy_get_buffer_bytes`

* params:
  - `i32 (`[`proxy_buffer_type_t`]`) buffer_id`
  - `i32 (uint32_t) offset`
  - `i32 (uint32_t) max_size`
  - `i32 (uint8_t **) return_value_data`
  - `i32 (size_t *) return_value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves up to` max_size` bytes starting at `offset` from the buffer
`buffer_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `buffer_id`, or in case of buffer overflow
   due to invalid `offset` and/or `max_size` values.
- `NOT_FOUND` when requested `buffer_id` isn't available.
- `INVALID_MEMORY_ACCESS` when `returned_value_data` and/or
  `returned_value_size` point to invalid memory address.


### `proxy_get_buffer_status`

* params:
  - `i32 (`[`proxy_buffer_type_t`]`) buffer_id`
  - `i32 (size_t *) return_buffer_size`
  - `i32 (uint32_t *) return_buffer_flags`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves `return_buffer_size` and `return_buffer_flags`
from the buffer `buffer_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `buffer_id`.
- `NOT_FOUND` when requested `buffer_id` isn't available.
- `INVALID_MEMORY_ACCESS` when `return_buffer_size` and/or
  `return_buffer_flags` point to invalid memory address.


## HTTP fields

### `proxy_get_header_map_size`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (size_t *) return_pairs_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves size (`return_pairs_size`) of all key-value pairs from
the header map `map_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `INVALID_MEMORY_ACCESS` when `return_pairs_size` points to
  invalid memory address.


### `proxy_get_header_map_pairs`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (uint8_t **) return_serialized_pairs_data`
  - `i32 (size_t *) return_serialized_pairs_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves all key-value pairs from the header map `map_id`.

`return_pairs` is [serialized].

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `INVALID_MEMORY_ACCESS` when `return_pairs_data` and/or
  `return_pairs_size` point to invalid memory address.


### `proxy_set_header_map_pairs`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (const uint8_t *) serialized_pairs_data`
  - `i32 (size_t) serialized_pairs_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sets all key-value pairs in the header map `map_id` to `pairs`.

`pairs` must be [serialized].

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `INVALID_MEMORY_ACCESS` when `pairs_data` and/or
  `pairs_size` point to invalid memory address.


### `proxy_get_header_map_value`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (const char *) key_data`
  - `i32 (size_t) key_size`
  - `i32 (uint8_t **) return_value_data`
  - `i32 (size_t *) return_value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves value of the header `key` from the header map `map_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `NOT_FOUND` when the requested `key` was not found.
- `INVALID_MEMORY_ACCESS` when `key_data`, `key_size`,
  `return_value_data` and/or `return_value_size` point to
  invalid memory address.


### `proxy_add_header_map_value`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (const char *) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const uint8_t *) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Adds `key` with `value` to the header map `map_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `INVALID_MEMORY_ACCESS` when `key_data`, `key_size`, `value_data`
  and/or `value_size` point to invalid memory address.


### `proxy_replace_header_map_value`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (const char *) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const uint8_t *) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Adds or replaces `key`'s value with `value` in the header map `map_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `INVALID_MEMORY_ACCESS` when `key_data`, `key_size`, `value_data`
  and/or `value_size` point to invalid memory address.


### `proxy_remove_header_map_value`

* params:
  - `i32 (`[`proxy_header_map_type_t`]`) map_id`
  - `i32 (const char *) key_data`
  - `i32 (size_t) key_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Removes the `key` from the header map `map_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `map_id`.
- `INVALID_MEMORY_ACCESS` when `key_data` and/or `key_size` point to
  invalid memory address.


## Common stream operations

### `proxy_continue_stream`

* params:
  - `i32 (`[`proxy_stream_type_t`]`) stream_type`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Resumes processing of paused `stream_type`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `stream_type`.
- `UNIMPLEMENTED` when continuation of requested `stream_type`
  is not supported.


### `proxy_close_stream`

* params:
  - `i32 (`[`proxy_stream_type_t`]`) stream_type`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Closes or resets `stream_type`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `stream_type`.


## TCP/UDP/QUIC streams

> **Note**
> `downstream` refers to the connection between client and proxy,
> `upstream` refers to the connection between proxy and backend.

> **Note**
> The same unique `context_id` is shared by a pair of `downstream`
> and `upstream` connections.


### `proxy_on_new_connection`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called on a new connection.

Plugin must return one of the following values:
- `CONTINUE` to allow the new connection to be established.
- `PAUSE` to pause processing of the new connection.


### `proxy_on_downstream_data`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) data_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called for each data chunk received from downstream.

Data (of `data_size`) can be retrieved and/or modified using
[`proxy_get_buffer_bytes`] and/or [`proxy_set_buffer_bytes`]
with `buffer_id` set to `DOWNSTREAM_DATA`.

Paused downstream can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`DOWNSTREAM`.

Plugin must return one of the following values:
- `CONTINUE` to forward `DOWNSTREAM_DATA` buffer upstream.
- `PAUSE` to pause processing.


### `proxy_on_downstream_connection_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (`[`proxy_peer_type_t`]`) peer_type`
* returns:
  - none

Called when downstream connection is closed.


### `proxy_on_upstream_data`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) data_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called for each data chunk received from upstream.

Data (of `data_size`) can be retrieved and/or modified using
[`proxy_get_buffer_bytes`] and/or [`proxy_set_buffer_bytes`]
with `buffer_id` set to `UPSTREAM_DATA`.

Paused downstream can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`UPSTREAM`.

Plugin must return one of the following values:
- `CONTINUE` to forward `UPSTREAM_DATA` buffer downstream.
- `PAUSE` to pause processing.


### `proxy_on_upstream_connection_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (`[`proxy_peer_type_t`]`) peer_type`
* returns:
  - none

Called when upstream connection is closed.


## HTTP

> **Note**
> The same unique `context_id` is shared by HTTP request and response.

### `proxy_on_request_headers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_headers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called when HTTP request headers are received from downstream.

All `num_headers` headers can be retrieved and/or modified using
[`proxy_get_header_map_pairs`] and/or [`proxy_set_header_map_pairs`]
with `map_id` set to `HTTP_REQUEST_HEADERS`.

Individual HTTP request headers can be retrieved and/or modified using
[`proxy_get_header_map_value`], [`proxy_replace_header_map_value`],
[`proxy_add_header_map_value`] and/or [`proxy_remove_header_map_value`]
with `map_id` set to `HTTP_REQUEST_HEADERS`.

Paused request can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`HTTP_REQUEST`.

Additionally, instead of forwarding request upstream, the host can send
HTTP response using [`proxy_send_local_response`].

Plugin must return one of the following values:
- `CONTINUE` to forward `HTTP_REQUEST_HEADERS` fields downstream.
- `PAUSE` to pause processing.


### `proxy_on_request_body`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) body_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called for each chunk of HTTP request body received from downstream.

Request body chunk (of `body_size`) can be retrieved and/or modified
using [`proxy_get_buffer_bytes`] and/or [`proxy_set_buffer_bytes`]
with `buffer_id` set to `HTTP_REQUEST_BODY`.

Paused HTTP request can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`HTTP_REQUEST`.

Plugin must return one of the following values:
- `CONTINUE` to forward `HTTP_REQUEST_BODY` buffer upstream.
- `PAUSE` to pause processing.


### `proxy_on_request_trailers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_trailers`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called when HTTP request trailers are received from downstream.

All `num_trailers` trailers can be retrieved and/or modified using
[`proxy_get_header_map_pairs`] and/or [`proxy_set_header_map_pairs`]
with `map_id` set to `HTTP_REQUEST_TRAILERS`.

Individual HTTP request trailers can be retrieved and/or modified using
[`proxy_get_header_map_value`], [`proxy_replace_header_map_value`],
[`proxy_add_header_map_value`] and/or [`proxy_remove_header_map_value`]
with `map_id` set to `HTTP_REQUEST_TRAILERS`.

Paused request can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`HTTP_REQUEST`.

Plugin must return one of the following values:
- `CONTINUE` to forward `HTTP_REQUEST_TRAILERS` fields downstream.
- `PAUSE` to pause processing.


### `proxy_on_response_headers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_headers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called when HTTP response headers are received from upstream.

All `num_headers` headers can be retrieved and/or modified using
[`proxy_get_header_map_pairs`] and/or [`proxy_set_header_map_pairs`]
with `map_id` set to `HTTP_RESPONSE_HEADERS`.

Individual headers can be retrieved and/or modified using
[`proxy_get_header_map_value`], [`proxy_replace_header_map_value`],
[`proxy_add_header_map_value`] and/or [`proxy_remove_header_map_value`]
with `map_id` set to `HTTP_RESPONSE_HEADERS`.

Paused request can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`HTTP_RESPONSE`.

Plugin must return one of the following values:
- `CONTINUE` to forward `HTTP_RESPONSE_HEADERS` fields downstream.
- `PAUSE` to pause processing.


### `proxy_on_response_body`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) body_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called for each chunk of HTTP response body received from upstream.

Response body chunk (of `body_size`) can be retrieved and/or modified
using [`proxy_get_buffer_bytes`] and/or [`proxy_set_buffer_bytes`]
with `buffer_id` set to `HTTP_RESPONSE_BODY`.

Paused HTTP response can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`HTTP_RESPONSE`.

Plugin must return one of the following values:
- `CONTINUE` to forward `HTTP_RESPONSE_BODY` buffer downstream.
- `PAUSE` to pause processing.


### `proxy_on_response_trailers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_trailers`
* returns:
  - `i32 (`[`proxy_action_t`]`) action`

Called when HTTP response trailers are received from upstream.

All `num_trailers` trailers can be retrieved and/or modified using
[`proxy_get_header_map_pairs`] and/or [`proxy_set_header_map_pairs`]
with `map_id` set to `HTTP_RESPONSE_TRAILERS`.

Individual trailers can be retrieved and/or modified using
[`proxy_get_header_map_value`], [`proxy_replace_header_map_value`],
[`proxy_add_header_map_value`] and/or [`proxy_remove_header_map_value`]
with `map_id` set to `HTTP_RESPONSE_TRAILERS`.

Paused request can be resumed using [`proxy_continue_stream`]
or closed using [`proxy_close_stream`] with `stream_type` set to
`HTTP_REQUEST`.

Plugin must return one of the following values:
- `CONTINUE` to forward `HTTP_RESPONSE_TRAILERS` fields downstream.
- `PAUSE` to pause processing.


### `proxy_send_local_response`

* params:
  - `i32 (uint32_t) status_code`
  - `i32 (const char *) status_code_details_data`
  - `i32 (size_t) status_code_details_size`
  - `i32 (const uint8_t *) body_data`
  - `i32 (size_t) body_size`
  - `i32 (const uint8_t *) serialized_headers_data`
  - `i32 (size_t) serialized_headers_size`
  - `i32 (uint32_t) grpc_status`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sends HTTP response with `body` and [serialized] `headers`
without forwarding the original request upstream.

Returned `status` value is:
- `OK` on success.
- `INVALID_MEMORY_ACCESS` when `status_code_details_data`,
  `status_code_details_size`, `body_data`, `body_size`, `headers_data`
  and/or `headers_size` point to invalid memory address.


## HTTP calls

### `proxy_http_call`

* params:
  - `i32 (const char *) upstream_name_data`
  - `i32 (size_t) upstream_name_size`
  - `i32 (const uint8_t *) serialized_headers_data`
  - `i32 (size_t) serialized_headers_size`
  - `i32 (const uint8_t *) body_data`
  - `i32 (size_t) body_size`
  - `i32 (const uint8_t *) serialized_trailers_data`
  - `i32 (size_t) serialized_trailers_size`
  - `i32 (uint32_t) timeout`
  - `i32 (uint32_t *) return_call_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sends HTTP request with [serialized] `headers`, `body`
and [serialized] `trailers` to `upstream_name`.

[`proxy_on_http_call_response`] will be called with `return_call_id`
when the response is received by the host, or after the `timeout`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `upstream`, or when `headers` are missing
  required `:authority`, `:method` and/or `:path` values.
- `INTERNAL_FAILURE` when host failed to send requested HTTP call.
- `INVALID_MEMORY_ACCESS` when `upstream_data`, `upstream_size`,
  `headers_data`, `headers_size` `body_data`, `body_size`,
  `trailers_data`, `trailers_size` and/or `return_call_id` point to
  invalid memory address.


### `proxy_on_http_call_response`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) call_id`
  - `i32 (size_t) num_headers`
  - `i32 (size_t) body_size`
  - `i32 (size_t) num_trailers`
* returns:
  - none

Called when HTTP response for `call_id` is received.

All `num_headers` headers can be retrieved using
[`proxy_get_header_map_pairs`]
or individually [`proxy_get_header_map_value`]
with `map_id` set to `HTTP_CALL_RESPONSE_HEADERS`.

Response body (of `body_size`) can be retrieved using
[`proxy_get_buffer_bytes`] with `buffer_id` set to
`HTTP_RESPONSE_BODY`.

All `num_trailers` trailers can be retrieved using
[`proxy_get_header_map_pairs`]
or individually [`proxy_get_header_map_value`]
with `map_id` set to `HTTP_CALL_RESPONSE_TRAILERS`.


## gRPC calls

### `proxy_grpc_call`

* params:
  - `i32 (const char *) upstream_data`
  - `i32 (size_t) upstream_size`
  - `i32 (const char *) service_name_data`
  - `i32 (size_t) service_name_size`
  - `i32 (const char *) method_name_data`
  - `i32 (size_t) method_name_size`
  - `i32 (const uint8_t *) serialized_initial_metadata_data`
  - `i32 (size_t) serialized_initial_metadata_size`
  - `i32 (const uint8_t *) message_data`
  - `i32 (size_t) message_size`
  - `i32 (uint32_t) timeout`
  - `i32 (uint32_t *) return_call_id`
 * returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sends gRPC `message` with [serialized] `initial_metadata`
to gRPC `method_name` on gRPC `service_name` on `upstream`.

[`proxy_on_grpc_receive`] or [`proxy_on_grpc_close`] will be called
with `return_call_id` when the response is received by the host, or
after the `timeout`.

`return_call_id` can also be used to cancel the outstanding request
using [`proxy_grpc_cancel`] or close it using [`proxy_grpc_close`].

Returned `status` value is:
- `OK` on success.
- `PARSE_FAILURE` for unknown `upstream`.
- `INTERNAL_FAILURE` when the host failed to send a gRPC call.
- `INVALID_MEMORY_ACCESS` when `upstream_data`, `upstream_size`,
  `service_name_data`, `service_name_size` `method_name_data`,
  `method_name_size`, `initial_metadata_data`, `initial_metadata_size`,
  `message_data`, `message_size` and/or `return_call_id` point to
  invalid memory address.


### `proxy_grpc_stream`

* params:
  - `i32 (const char *) upstream_data`
  - `i32 (size_t) upstream_size`
  - `i32 (const char *) service_name_data`
  - `i32 (size_t) service_name_size`
  - `i32 (const char *) method_name_data`
  - `i32 (size_t) method_name_size`
  - `i32 (const uint8_t *) serialized_initial_metadata_data`
  - `i32 (size_t) serialized_initial_metadata_size`
  - `i32 (uint32_t *) return_stream_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Opens gRPC stream with [serialized] `initial_metadata`
to gRPC `method_name` on gRPC `service_name` on `upstream`.

gRPC messages can be sent on this stream using [`proxy_grpc_send`]
with `return_stream_id`.

[`proxy_on_grpc_receive`] or [`proxy_on_grpc_close`] will be called
with `return_stream_id` when the response is received by the host,
or after the `timeout`.

`return_stream_id` can also be used to cancel outstanding gRPC messages
using [`proxy_grpc_cancel`] or close this gRPC stream using
[`proxy_grpc_close`].

Returned `status` value is:
- `OK` on success.
- `PARSE_FAILURE` for unknown `upstream`.
- `INTERNAL_FAILURE` when the host failed to open gRPC stream.
- `INVALID_MEMORY_ACCESS` when `upstream_data`, `upstream_size`,
  `service_name_data`, `service_name_size` `method_name_data`,
  `method_name_size`, `initial_metadata_data`, `initial_metadata_size`
  and/or `return_stream_id` point to invalid memory address.


### `proxy_grpc_send`

* params:
  - `i32 (uint32_t) stream_id`
  - `i32 (const uint8_t *) message_data`
  - `i32 (size_t) message_size`
  - `i32 (bool) end_stream`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sends gRPC `message` on the gRPC stream `stream_id` previously
established using [`proxy_grpc_stream`].

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for invalid `stream_id`.
- `NOT_FOUND` for unknown `stream_id`.
- `INVALID_MEMORY_ACCESS` when `message_data` and/or `message_size`
  point to invalid memory address.


### `proxy_grpc_cancel`

* params:
  - `i32 (uint32_t) call_or_stream_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Cancels `call_or_stream_id` previously started using
[`proxy_grpc_call`] or [`proxy_grpc_stream`].

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for invalid `call_or_stream_id`.
- `NOT_FOUND` for unknown `call_or_stream_id`.


### `proxy_grpc_close`

* params:
  - `i32 (uint32_t) call_or_stream_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Closes `call_or_stream_id` previously started using
[`proxy_grpc_call`] or [`proxy_grpc_stream`].

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for invalid `call_or_stream_id`.
- `NOT_FOUND` for unknown `call_or_stream_id`.


### `proxy_get_status`

* params:
  - `i32 (uint32_t *) return_status_code`
  - `i32 (const char **) return_status_message_data`
  - `i32 (size_t *) return_status_message_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves gRPC `return_status_code` and `return_status_message`
when called from the [`proxy_on_grpc_close`] callback.

Returned `status` value is:
- `OK` on success.
- `INVALID_MEMORY_ACCESS` when `return_status_code`,
  `return_status_message_data` and/or `return_status_message_size`
  point to invalid memory address.


### `proxy_on_grpc_receive_initial_metadata`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) call_id`
  - `i32 (size_t) num_elements`
* returns:
  - none

Called when initial gRPC metadata for `call_id` is received.

All `num_elements` elements can be retrieved using
[`proxy_get_header_map_pairs`] or individually
[`proxy_get_header_map_value`] with `map_id` set to
`GRPC_CALL_INITIAL_METADATA`.


### `proxy_on_grpc_receive`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) call_id`
  - `i32 (size_t) message_size`
* returns:
  - none

Called when the response gRPC message for `call_id` is received.

Message (of `message_size`) can be retrieved using
[`proxy_get_buffer_bytes`] with `buffer_id` set to `GRPC_CALL_MESSAGE`.


### `proxy_on_grpc_receive_trailing_metadata`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) call_id`
  - `i32 (size_t) num_elements`
* returns:
  - none

Called when trailing gRPC metadata for `call_id` is received.

All `num_elements` elements can be retrieved using
[`proxy_get_header_map_pairs`] or individually
[`proxy_get_header_map_value`] with `map_id` set to
`GRPC_CALL_TRAILING_METADATA`.


### `proxy_on_grpc_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) call_id`
  - `i32 (uint32_t) status_code`
* returns:
  - none

Called when gRPC call or stream `call_id` is closed.

gRPC status message can be retrieved using [`proxy_get_status`].


## Shared Key-Value Store

### `proxy_set_shared_data`

* params:
  - `i32 (const char *) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const uint8_t *) value_data`
  - `i32 (size_t) value_size`
  - `i32 (uint32_t) cas`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sets shared data identified by the `key` to the `value`.

If the compare-and-swap value (`cas`) is set to a non-zero value,
then it must match the host's compare-and-swap value in order for
the update to succeed.

Returned `status` value is:
- `OK` on success.
- `CAS_MISMATCH` when `cas` doesn't match host's compare-and-swap
  value.
- `INVALID_MEMORY_ACCESS` when `key_data`, `key_size`, `value_data`,
  `value_size` and/or `cas` point to invalid memory address.


### `proxy_get_shared_data`

* params:
  - `i32 (const char *) key_data`
  - `i32 (size_t) key_size`
  - `i32 (uint8_t **) return_value_data`
  - `i32 (size_t *) return_value_size`
  - `i32 (uint32_t *) return_cas`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Returns shared value (`return_value`) identified by the `key`.

The compare-and-swap value (`return_cas`) can be used for atomically
updating this value using [`proxy_set_shared_data`].

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested `key` was not found.
- `INVALID_MEMORY_ACCESS` when `key_data`, `key_size`,
  `return_value_data`, `return_value_size` and/or `return_cas`
  point to invalid memory address.


## Shared Queues

### `proxy_register_shared_queue`

* params:
  - `i32 (const char *) name_data`
  - `i32 (size_t) name_size`
  - `i32 (uint32_t *) return_queue_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Registers shared queue under `name`.

If the `name` queue already exists, then it's going to be opened
instead of creating a new empty queue.

Items can be enqueued/dequeued on the created/opened queue using
[`proxy_enqueue_shared_queue`] and/or [`proxy_dequeue_shared_queue`]
with `return_queue_id`.

Returned `status` value is:
- `OK` on success.
- `INVALID_MEMORY_ACCESS` when `name_data`, `name_size`
  and/or `return_queue_id` point to invalid memory address.


### `proxy_resolve_shared_queue`

* params:
  - `i32 (const char *) vm_id_data`
  - `i32 (size_t) vm_id_size`
  - `i32 (const char *) name_data`
  - `i32 (size_t) name_size`
  - `i32 (uint32_t *) return_queue_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Resolves existing shared queue using `vm_id` and `name`.

Items can be enqueued/dequeued on the opened queue using
[`proxy_enqueue_shared_queue`] and/or [`proxy_dequeue_shared_queue`]
with `return_queue_id`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested queue was not found.
- `INVALID_MEMORY_ACCESS` when `vm_id_data`, `vm_id_size`, `name_data`,
  `name_size` and/or `return_queue_id` point to invalid memory address.


### `proxy_enqueue_shared_queue`

* params:
  - `i32 (uint32_t) queue_id`
  - `i32 (const uint8_t *) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Adds `value` to the end of the queue `queue_id`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested `queue_id` was not found.
- `INVALID_MEMORY_ACCESS` when `value_data` and/or `value_size` point
  to invalid memory address.


### `proxy_dequeue_shared_queue`

* params:
  - `i32 (uint32_t) queue_id`
  - `i32 (uint8_t **) return_value_data`
  - `i32 (size_t *) return_value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves `return_value` from the front of the queue `queue_id`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested `queue_id` was not found.
- `EMPTY` when there is nothing to dequeue from the requested queue.
- `INVALID_MEMORY_ACCESS` when `return_value_data`
  and/or `return_value_size` point to invalid memory address.


### `proxy_on_queue_ready`
* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) queue_id`
* returns:
  - none

Called when a new item is enqueued on the queue `queue_id`.


## Metrics

### `proxy_define_metric`

* params:
  - `i32 (`[`proxy_metric_type_t`]`) metric_type`
  - `i32 (const char *) name_data`
  - `i32 (size_t) name_size`
  - `i32 (uint32_t *) return_metric_id`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Defines a new metric of `metric_type` with a `name`.

Its value can be modified and/or retrieved using
[`proxy_record_metric`], [`proxy_increment_metric`]
and/or [`proxy_get_metric`] with `return_metric_id`.

Returned `status` value is:
- `OK` on success.
- `BAD_ARGUMENT` for unknown `metric_type`.
- `INVALID_MEMORY_ACCESS` when `name_data`, `name_size`
  and/or `return_metric_id` point to invalid memory address.


### `proxy_record_metric`

* params:
  - `i32 (uint32_t) metric_id`
  - `i64 (uint64_t) value`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Sets metric `metric_id` to the `value`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested `metric_id` was not found.


### `proxy_increment_metric`

* params:
  - `i32 (uint32_t) metric_id`
  - `i64 (int64_t) offset`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Changes metric `metric_id` by the `offset`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested `metric_id` was not found.
- `BAD_ARGUMENT` when requested `offset` cannot be applied to
  `metric_id` (e.g. trying to decrement counter).


### `proxy_get_metric`

* params:
  - `i32 (uint32_t) metric_id`
  - `i32 (uint64_t *) return_value`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Retrieves `return_value` of the metric `metric_id`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested `metric_id` was not found.
- `INVALID_MEMORY_ACCESS` when `return_value` points to invalid memory
  address.


## Properties

> **Warning**
> Properties are implementation-dependent and not stable.

### `proxy_get_property`

* params:
  - `i32 (const uint8_t *) path_data`
  - `i32 (size_t) path_size`
  - `i32 (uint8_t **) return_value_data`
  - `i32 (size_t *) return_value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when there was no property found at the requested `path`.
- `SERIALIZATION_FAILURE` TODO
- `INVALID_MEMORY_ACCESS` when `path_data`, `path_size`,
  `return_value_data` and/or `return_value_size` point to invalid
  memory address.


### `proxy_set_property`

* params:
  - `i32 (const uint8_t *) path_data`
  - `i32 (size_t) path_size`
  - `i32 (const uint8_t *) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when there was no property found at the requested `path`.
- `INVALID_MEMORY_ACCESS` when `path_data`, `path_size`, `value_data`
  and/or `value_size` point to invalid memory address.


## Foreign function interface (FFI)

### `proxy_call_foreign_function`

* params:
  - `i32 (const char *) name_data`
  - `i32 (size_t) name_size`
  - `i32 (const uint8_t *) arguments_data`
  - `i32 (size_t) arguments_size`
  - `i32 (uint8_t **) return_results_data`
  - `i32 (size_t *) return_results_size`
* returns:
  - `i32 (`[`proxy_status_t`]`) status`

Calls registered foreign function `name` with `arguments`.

Its optional return values are written to `return_results`.

Returned `status` value is:
- `OK` on success.
- `NOT_FOUND` when the requested function was not found.
- `INVALID_MEMORY_ACCESS` when `name_data`, `name_size`,
  `arguments_data`, `arguments_size`, `return_results_data`
  and/or `return_results_size` point to invalid memory address.


### `proxy_on_foreign_function`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) function_id`
  - `i32 (size_t) arguments_size`
* returns:
  - none

Called when a registered foreign callback `function_id` is called.

Its arguments (of `arguments_size`) can be retrieved using
[`proxy_get_buffer_bytes`] with `buffer_id` set to
`FOREIGN_FUNCTION_ARGUMENTS`.


## Unimplemented WASI functions

Various unimplemented WASI functions that are expected to be present
when the Wasm module is compiled for the `wasm32-wasi` target.


### `wasi_snapshot_preview1.args_sizes_get`

* params:
  - `i32 (size_t *) return_argc`
  - `i32 (size_t *) return_argv_buffer_size`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

Hosts must write `0` to both output arguments to prevent
[`wasi_snapshot_preview1.args_get`] from being called.

Returned `errno` value is:
- `SUCCESS` on success.
- `FAULT` when `return_argc` and/or `return_argv_buffer_size` point to
  invalid memory address.


### `wasi_snapshot_preview1.args_get`

* params:
  - `i32 (uint8_t **) return_argv`
  - `i32 (uint8_t *) return_argv_size`
* returns:
  - `i32 (`[`wasi_errno_t`]`) errno`

This function should not be called if zero sizes were returned
in [`wasi_snapshot_preview1.args_sizes_get`].

Returned `errno` value is:
- `SUCCESS` on success.


### `wasi_snapshot_preview1.proc_exit`

* params:
  - `i32 (uint32_t) exit_code`
* returns:
  - none

This function is never called.


# Types

### `proxy_log_level_t`

- `TRACE` = `0`
- `DEBUG` = `1`
- `INFO` = `2`
- `WARN` = `3`
- `ERROR` = `4`
- `CRITICAL` = `5`


### `proxy_status_t`

- `OK` = `0`
- `NOT_FOUND` = `1`
- `BAD_ARGUMENT` = `2`
- `SERIALIZATION_FAILURE` = `3`
- `PARSE_FAILURE` = `4`
- `INVALID_MEMORY_ACCESS` = `6`
- `EMPTY` = `7`
- `CAS_MISMATCH` = `8`
- `INTERNAL_FAILURE` = `10`
- `UNIMPLEMENTED` = `12`


### `proxy_action_t`

- `CONTINUE` = `0`
- `PAUSE` = `1`


### `proxy_buffer_type_t`

- `HTTP_REQUEST_BODY` = `0`
- `HTTP_RESPONSE_BODY` = `1`
- `DOWNSTREAM_DATA` = `2`
- `UPSTREAM_DATA` = `3`
- `HTTP_CALL_RESPONSE_BODY` = `4`
- `GRPC_CALL_MESSAGE` = `5`
- `VM_CONFIGURATION` = `6`
- `PLUGIN_CONFIGURATION` = `7`
- `FOREIGN_FUNCTION_ARGUMENTS` = `8`


### `proxy_header_map_type_t`

- `HTTP_REQUEST_HEADERS` = `0`
- `HTTP_REQUEST_TRAILERS` = `1`
- `HTTP_RESPONSE_HEADERS` = `2`
- `HTTP_RESPONSE_TRAILERS` = `3`
- `GRPC_CALL_INITIAL_METADATA` = `4`
- `GRPC_CALL_TRAILING_METADATA` = `5`
- `HTTP_CALL_RESPONSE_HEADERS` = `6`
- `HTTP_CALL_RESPONSE_TRAILERS` = `7`


### `proxy_peer_type_t`

- `UNKNOWN` = `0`
- `LOCAL` = `1`
- `REMOTE` =`2`


### `proxy_stream_type_t`

- `HTTP_REQUEST` = `0`
- `HTTP_RESPONSE` = `1`
- `DOWNSTREAM` = `2`
- `UPSTREAM` = `3`


### `proxy_metric_type_t`

- `COUNTER` = `0`
- `GAUGE` = `1`
- `HISTOGRAM` = `2`


### `wasi_errno_t`

- `SUCCESS` = `0`
- `BADF` = `8`
- `FAULT` = `21`
- `INVAL` = `28`
- `NOTSUP` = `58`


### `wasi_fd_id_t`

- `STDOUT` = `1`
- `STDERR` = `2`


### `wasi_clock_id_t`

- `REALTIME` = `0`
- `MONOTONIC` = `1`


# Serialization

## HTTP fields and gRPC metadata

TODO


[integration]: #Integration
[memory management]: #Memory-management
[serialized]: #Serialization

[`proxy_abi_version_0_2_1`]: #proxy_abi_version_0_2_1
[`_initialize`]: #_initialize
[`main`]: #main
[`_start`]: #_start
[`proxy_on_memory_allocate`]: #proxy_on_memory_allocate
[`malloc`]: #malloc
[`proxy_on_context_create`]: #proxy_on_context_create
[`proxy_on_done`]: #proxy_on_done
[`proxy_on_log`]: #proxy_on_log
[`proxy_on_delete`]: #proxy_on_delete
[`proxy_done`]: #proxy_done
[`proxy_set_effective_context`]: #proxy_set_effective_context
[`proxy_on_vm_start`]: #proxy_on_vm_start
[`proxy_on_configure`]: #proxy_on_configure
[`proxy_log`]: #proxy_log
[`proxy_get_log_level`]: #proxy_get_log_level
[`proxy_get_current_time_nanoseconds`]: #proxy_get_current_time_nanoseconds
[`proxy_set_tick_period_milliseconds`]: #proxy_set_tick_period_milliseconds
[`proxy_on_tick`]: #proxy_on_tick
[`proxy_set_buffer_bytes`]: #proxy_set_buffer_bytes
[`proxy_get_buffer_bytes`]: #proxy_get_buffer_bytes
[`proxy_get_buffer_status`]: #proxy_get_buffer_status
[`proxy_get_header_map_size`]: #proxy_get_header_map_size
[`proxy_get_header_map_pairs`]: #proxy_get_header_map_pairs
[`proxy_set_header_map_pairs`]: #proxy_set_header_map_pairs
[`proxy_get_header_map_value`]: #proxy_get_header_map_value
[`proxy_add_header_map_value`]: #proxy_add_header_map_value
[`proxy_replace_header_map_value`]: #proxy_replace_header_map_value
[`proxy_remove_header_map_value`]: #proxy_remove_header_map_value
[`proxy_continue_stream`]: #proxy_continue_stream
[`proxy_close_stream`]: #proxy_close_stream
[`proxy_on_new_connection`]: #proxy_on_new_connection
[`proxy_on_downstream_data`]: #proxy_on_downstream_data
[`proxy_on_downstream_connection_close`]: #proxy_on_downstream_connection_close
[`proxy_on_upstream_data`]: #proxy_on_upstream_data
[`proxy_on_upstream_connection_close`]: #proxy_on_upstream_connection_close
[`proxy_on_request_headers`]: #proxy_on_request_headers
[`proxy_on_request_body`]: #proxy_on_request_body
[`proxy_on_request_trailers`]: #proxy_on_request_trailers
[`proxy_on_response_headers`]: #proxy_on_response_headers
[`proxy_on_response_body`]: #proxy_on_response_body
[`proxy_on_response_trailers`]: #proxy_on_response_trailers
[`proxy_send_local_response`]: #proxy_send_local_response
[`proxy_http_call`]: #proxy_http_call
[`proxy_on_http_call_response`]: #proxy_on_http_call_response
[`proxy_grpc_call`]: #proxy_grpc_call
[`proxy_grpc_stream`]: #proxy_grpc_stream
[`proxy_grpc_send`]: #proxy_grpc_send
[`proxy_grpc_cancel`]: #proxy_grpc_cancel
[`proxy_grpc_close`]: #proxy_grpc_close
[`proxy_get_status`]: #proxy_get_status
[`proxy_on_grpc_receive_initial_metadata`]: #proxy_on_grpc_receive_initial_metadata
[`proxy_on_grpc_receive`]: #proxy_on_grpc_receive
[`proxy_on_grpc_receive_trailing_metadata`]: #proxy_on_grpc_receive_trailing_metadata
[`proxy_on_grpc_close`]: #proxy_on_grpc_close
[`proxy_set_shared_data`]: #proxy_set_shared_data
[`proxy_get_shared_data`]: #proxy_get_shared_data
[`proxy_register_shared_queue`]: #proxy_register_shared_queue
[`proxy_resolve_shared_queue`]: #proxy_resolve_shared_queue
[`proxy_enqueue_shared_queue`]: #proxy_enqueue_shared_queue
[`proxy_dequeue_shared_queue`]: #proxy_dequeue_shared_queue
[`proxy_on_queue_ready`]: #proxy_on_queue_ready
[`proxy_define_metric`]: #proxy_define_metric
[`proxy_record_metric`]: #proxy_record_metric
[`proxy_increment_metric`]: #proxy_increment_metric
[`proxy_get_metric`]: #proxy_get_metric
[`proxy_get_property`]: #proxy_get_property
[`proxy_set_property`]: #proxy_set_property
[`proxy_call_foreign_function`]: #proxy_call_foreign_function
[`proxy_on_foreign_function`]: #proxy_on_foreign_function

[`wasi_snapshot_preview1.fd_write`]: #wasi_snapshot_preview1.fd_write
[`wasi_snapshot_preview1.clock_time_get`]: #wasi_snapshot_preview1.clock_time_get
[`wasi_snapshot_preview1.random_get`]: #wasi_snapshot_preview1.random_get
[`wasi_snapshot_preview1.environ_sizes_get`]: #wasi_snapshot_preview1.environ_sizes_get
[`wasi_snapshot_preview1.environ_get`]: #wasi_snapshot_preview1.environ_get
[`wasi_snapshot_preview1.args_sizes_get`]: #wasi_snapshot_preview1.args_sizes_get
[`wasi_snapshot_preview1.args_get`]: #wasi_snapshot_preview1.args_get
[`wasi_snapshot_preview1.proc_exit`]: #wasi_snapshot_preview1.proc_exit

[`proxy_log_level_t`]: #proxy_log_level_t
[`proxy_status_t`]: #proxy_status_t
[`proxy_action_t`]: #proxy_action_t
[`proxy_buffer_type_t`]: #proxy_buffer_type_t
[`proxy_header_map_type_t`]: #proxy_header_map_type_t
[`proxy_peer_type_t`]: #proxy_peer_type_t
[`proxy_stream_type_t`]: #proxy_stream_type_t
[`proxy_metric_type_t`]: #proxy_metric_type_t

[`wasi_errno_t`]: #wasi_errno_t
[`wasi_fd_id_t`]: #wasi_fd_id_t
[`wasi_clock_id_t`]: #wasi_clock_id_t
