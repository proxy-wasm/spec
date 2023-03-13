# Proxy-Wasm vNEXT ABI specification

# Functions implemented in the Wasm module

All functions implemented in the Wasm module, other than the integration and memory management
functions, include context identifier (`context_id`) as the first parameter, which should be used to
distinguish between different contexts.


## Integration

### `_start`

* params:
  - none
* returns:
  - none

Start function which is called when the module is loaded and initialized. This can be used by SDKs
to setup and/or initialize state, but no proxy_ functions can be used at that point yet.


### `proxy_abi_version_X_Y_Z`

* params:
  - none
* returns:
  - none

Exports ABI version (vX.Y.Z) this module was compiled against.

Considered alternatives: Ideally, this would be exported as a global variable, but existing
toolchains have trouble with that (some don’t support global variables at all, some incorrectly
export language-specific and not Wasm-generic variables).


## Memory management

### `proxy_on_memory_allocate`

* params:
  - `i32 (size_t) memory_size`
* returns:
  - `i32 (void*) allocated_ptr`

Allocates memory using in-VM memory allocator and returns it to the host.


## Module lifecycle

### `proxy_on_context_create`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) parent_context_id`
* returns:
  - none

Called when the host environment creates a new root context (if `parent_context_id` is `0`) or a new
per-stream context.


### `proxy_on_done`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (bool) is_done`

Called when the host environment is done processing the context (`context_id`). Return value
indicates when the Wasm VM is done with the processing as well.


### `proxy_on_log`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - none

Called in the logging phase.


### `proxy_on_delete`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - none

Called when the host environment removes the context (`context_id`). This is used to signal that VM
should stop tracking that `context_id` and remove all associated state.


## Configuration

### `proxy_on_vm_start`

* params:
  - `i32 (uint32_t) root_context_id`
  - `i32 (size_t) vm_configuration_size`
* returns:
  - `i32 (bool) success`

Called when the host environment starts the WebAssembly Virtual Machine. Its configuration
(`vm_configuration_size`) might be retrieved using `proxy_get_buffer`.


### `proxy_on_configure`

* params:
  - `i32 (uint32_t) root_context_id`
  - `i32 (size_t) plugin_configuration_size`
* returns:
  - `i32 (bool) success`

Called when the host environment starts the plugin. Its configuration (`plugin_configuration_size`)
might be retrieved using `proxy_get_buffer`.


## Timers

### `proxy_on_tick`

* params:
  - `i32 (uint32_t) root_context_id`
* returns:
  - none

Timer called every tick period. Tick period can be configured using
`proxy_set_tick_period_milliseconds`.


## TCP/UDP/QUIC stream (L4) extensions

Note: `downstream` means the connection between client and proxy, and `upstream` means the
connection between proxy and backend (aka “next hop”).

### `proxy_on_new_connection`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (proxy_action_t) next_action`

Called on a new TCP connection.


### `proxy_on_downstream_data`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) data_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each data chunk received from downstream.


### `proxy_on_downstream_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (proxy_peer_type_t) peer_type`
* returns:
  - none

Called when downstream connection is closed.


### `proxy_on_upstream_data`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) data_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each data chunk received from upstream.


### `proxy_on_upstream_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (proxy_peer_type_t) peer_type`
* returns:
  - none

Called when upstream connection is closed.


## HTTP (L7) extensions

### `proxy_on_http_request_headers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_headers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP request headers are received from the client. Headers can be retrieved using
`proxy_get_map` and/or `proxy_get_map_value`.


### `proxy_on_http_request_body`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) body_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each chunk of HTTP request body received from the client. Request body can be retrieved
using `proxy_get_buffer`.


### `proxy_on_http_request_trailers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_trailers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP request trailers are received from the client. Trailers can be retrieved using
`proxy_get_map` and/or `proxy_get_map_value`.


### `proxy_on_http_request_metadata`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_elements`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each HTTP/2 METADATA frame received from the client.  Metadata can be retrieved using
`proxy_get_map` and/or `proxy_get_map_value`.


### `proxy_on_http_response_headers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_headers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP response headers are received from the upstream. Headers can be retrieved using
`proxy_get_map` and/or `proxy_get_map_value`.


### `proxy_on_http_response_body`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) body_size`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each chunk of HTTP response body received from the client. Response body can be retrieved
using `proxy_get_buffer`.


### `proxy_on_http_response_trailers`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_trailers`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called when HTTP response trailers are received from the upstream. Trailers can be retrieved using
`proxy_get_map` and/or `proxy_get_map_value`.


### `proxy_on_http_response_metadata`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (size_t) num_elements`
  - `i32 (bool) end_of_stream`
* returns:
  - `i32 (proxy_action_t) next_action`

Called for each HTTP/2 METADATA frame received from the upstream.  Metadata can be retrieved using
`proxy_get_map` and/or `proxy_get_map_value`.


## HTTP calls

### `proxy_on_http_call_response`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) num_headers`
  - `i32 (size_t) body_size`
  - `i32 (size_t) num_trailers`
* returns:
  - none

Called when the response to the HTTP call (`callout_id`) is received.


## gRPC callouts

### `proxy_on_grpc_call_response_header_metadata`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) num_elemets`
* returns:
  - none

Called when header metadata in the response to the gRPC call (`callout_id`) is received.


### `proxy_on_grpc_call_response_message`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) message_size`
* returns:
  - none

Called when the response to the gRPC call (`callout_id`) is received.


### `proxy_on_grpc_call_response_trailer_metadata`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) callout_id`
  - `i32 (size_t) num_elements`
* returns:
  - none

Called when trailer metadata in the response to the gRPC call (`callout_id`) is received.


### `proxy_on_grpc_call_close`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) callout_id`
  - `i32 (uint32_t) status_code`
* returns:
  - none

Called when the request to the gRPC call (`callout_id`) fails or if the gRPC connection is closed.


## Shared queue

### `proxy_on_queue_ready`

* params:
  - `i32 (uint32_t) context_id`
  - `i32 (uint32_t) queue_id`
* returns:
  - none

Called when there is data available in the queue (`queue_id`).


# Functions implemented in the host environment

All functions implemented in the host environment return `proxy_result_t`, which indicates the
status of the call (successful, invalid memory access, etc.), and the return values are written into
memory pointers passed in as arguments (indicated by the `return_` prefix in the specification).


## Logging

### `proxy_log`

* params:
  - `i32 (proxy_log_level_t) log_level`
  - `i32 (const char*) message_data`
  - `i32 (size_t) message_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Log message (`message_data`, `message_size`) at the given `log_level`.

### `proxy_log_destination`

* params:
  - `i32 (const char*) log_destination_data`
  - `i32 (size_t) log_destination_size`
  - `i32 (proxy_log_level_t) log_level`
  - `i32 (const char*) message_data`
  - `i32 (size_t) message_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Log message (`message_data`, `message_size`) at the given `log_level` to the log destination (`log_destination_data`, `log_destination_size`).

### `proxy_get_current_time`

* params:
  - `i32 (uint64_t*) return_current_time_nanoseconds`
* returns:
  - `i32 (proxy_result_t) call_result`

Get current time (`return_current_time`).


## Module lifecycle

### `proxy_set_effective_context`

* params:
  - `i32 (uint32_t) context_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Change the effective context to `context_id`. This function is usually used to change the context
after receiving `proxy_on_http_call_response`, `proxy_on_grpc_call_response` or
`proxy_on_queue_ready`.


### `proxy_done`

* params:
  - none
* returns:
  - `i32 (proxy_result_t) call_result`

Indicate to the host environment that Wasm VM side is done processing current context. This can be
used after returning `false` in `proxy_on_done`.


## Timers

### `proxy_set_tick_period`

* params:
  - `i32 (uint32_t) tick_period`
* returns:
  - `i32 (proxy_result_t) call_result`

Set timer period (`tick_period`). Once set, the host environment will call `proxy_on_tick` every
`tick_period` milliseconds.


## Buffers, maps, and properties

### `proxy_get_buffer`

* params:
  - `i32 (proxy_buffer_type_t) buffer_type`
  - `i32 (offset_t) offset`
  - `i32 (size_t) max_size`
  - `i32 (const char**) return_buffer_data`
  - `i32 (size_t*) return_buffer_size`
  - `i32 (uint32_t*) return_flags`
* returns:
  - `i32 (proxy_result_t) call_result`

Get up to max_size bytes from the `buffer_type`, starting from `offset`. Bytes are written into
buffer slice (`return_buffer_data`, `return_buffer_size`), and buffer flags are written into
`return_flags`.


### `proxy_set_buffer`

* params:
  - `i32 (proxy_buffer_type_t) buffer_type`
  - `i32 (offset_t) offset`
  - `i32 (size_t) size`
  - `i32 (const char*) buffer_data`
  - `i32 (size_t) buffer_size`
  - `i32 (uint32_t) flags`
* returns:
  - `i32 (proxy_result_t) call_result`

Set content of the buffer `buffer_type` to the bytes (`buffer_data`, `buffer_size`), replacing
`size` bytes, starting at `offset` in the existing buffer.


### `proxy_get_map`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char**) return_map_data`
  - `i32 (size_t*) return_map_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Get all key-value pairs from a given map (`map_type`).


### `proxy_set_map`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) map_data`
  - `i32 (size_t) map_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Set all key-value pairs in a given map (`map_type`).


### `proxy_get_map_value`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char**) return_value_data`
  - `i32 (size_t*) return_value_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Get content of key (`key_data`, `key_size`) from a given map (`map_type`).


### `proxy_set_map_value`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Set or replace the content of key (`key_data`, `key_size`) to the value (`value_data`, `value_size`)
in a given map (`map_type`).

### `proxy_add_map_value`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) value_data`
  - `i32 (size_t) value_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Add key (`key_data`, `key_size`) with the value (`value_data`, `value_size`) to a given map
(`map_type`).


### `proxy_remove_map_value`

* params:
  - `i32 (proxy_map_type_t) map_type`
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Remove key (`key_data`, `key_size`) from a given map (`map_type`).


### `proxy_get_property`

* params:
  - `i32 (const char*) property_path_data`
  - `i32 (size_t) property_path_size`
  - `i32 (const char**) return_property_value_data`
  - `i32 (size_t*) return_property_value_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Get property (`property_path_data`, `property_path_size`).


### `proxy_set_property`

* params:
  - `i32 (const char*) property_path_data`
  - `i32 (size_t) property_path_size`
  - `i32 (const char*) property_value_data`
  - `i32 (size_t) property_value_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Set property (`property_path_data`, `property_path_size`) to given value (`property_value_data`,
`property_value_size`).


## TCP/UDP/QUIC stream (L4) extensions

### `proxy_resume_downstream`

* params:
  - none
* returns:
  - `i32 (proxy_result_t) call_result`

Resume processing of paused downstream.


### `proxy_resume_upstream`

* params:
  - none
* returns:
  - `i32 (proxy_result_t) call_result`

Resume processing of paused upstream.


## HTTP (L7) extensions

### `proxy_resume_http_request`

* params:
  - none
* returns:
  - `i32 (proxy_result_t) call_result`

Resume processing of paused HTTP request.


### `proxy_resume_http_response`

* params:
  - none
* returns:
  - `i32 (proxy_result_t) call_result`

Resume processing of paused HTTP response.


### `proxy_send_http_response`

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

Sends HTTP response without forwarding request to the upstream.


## HTTP calls

### `proxy_dispatch_http_call`

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

Dispatch a HTTP call to upstream (`upstream_name_data`, `upstream_name_size`). Once the response is
returned to the host, `proxy_on_http_call_response` will be called with a unique call identifier
(`return_callout_id`).


## gRPC calls

### `proxy_dispatch_grpc_call`

* params:
  - `i32 (const char*) grpc_service_data`
  - `i32 (size_t) grpc_service_size`
  - `i32 (const char*) service_name_data`
  - `i32 (size_t) service_name_size`
  - `i32 (const char*) method_data`
  - `i32 (size_t) method_size`
  - `i32 (const char*) grpc_message_data`
  - `i32 (size_t) grpc_message_size`
  - `i32 (uint32_t) timeout_milliseconds`
  - `i32 (uint32_t*) return_callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Dispatch a gRPC call to a service (`service_name_data`, `service_name_size`). Once the response is
returned to the host, `proxy_on_grpc_call_response` and/or `proxy_on_grpc_call_close` will be called
with a unique call identifier (`return_callout_id`). The call identifier can also be used to cancel
outstanding requests using `proxy_cancel_grpc_call` or close outstanding request using or
`proxy_close_grpc_call`.


### `proxy_open_grpc_stream`

* params:
  - `i32 (const char*) grpc_service_data`
  - `i32 (size_t) grpc_service_size`
  - `i32 (const char*) service_name_data`
  - `i32 (size_t) service_name_size`
  - `i32 (const char*) method_data`
  - `i32 (size_t) method_size`
  - `i32 (uint32_t*) return_callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Open a connection to the gRPC service (`service_name_data`, `service_name_size`).


### `proxy_send_grpc_call_message`

* params:
  - `i32 (uint32_t) callout_id`
  - `i32 (const char*) grpc_message_data`
  - `i32 (size_t) grpc_message_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Send gRPC message (`grpc_message_data`, `grpc_message_size`) on the existing gRPC stream
(`callout_id`).


### `proxy_cancel_grpc_call`

* params:
  - `i32 (uint32_t) callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Cancel outstanding gRPC request or stream (`callout_id`).


### `proxy_close_grpc_call`

* params:
  - `i32 (uint32_t) callout_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Cancel outstanding gRPC request or close existing gRPC stream (`callout_id`).


## Shared Key-Value store

### `proxy_get_shared_data`

* params:
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char**) return_value_data`
  - `i32 (size_t*) return_value_size`
  - `i32 (uint32_t*) return_cas`
* returns:
  - `i32 (proxy_result_t) call_result`

Get shared data identified by a key (`key_data`, `key_size`). The compare-and-switch value
(`return_cas`) is returned and can be used when updating the value with `proxy_set_shared_data`.


### `proxy_set_shared_data`

* params:
  - `i32 (const char*) key_data`
  - `i32 (size_t) key_size`
  - `i32 (const char*) value_data`
  - `i32 (size_t) value_size`
  - `i32 (uint32_t) cas`
* returns:
  - `i32 (proxy_result_t) call_result`

Set shared data identified by a key (`key_data`, `key_size`) to a value (`value_data`,
`value_size`). If compare-and-switch value (`cas`) is set, then it must match the current value in
order to update to succeed.


## Shared queue

### `proxy_register_shared_queue`

* params:
  - `i32 (const char*) queue_name_data`
  - `i32 (size_t) queue_name_size`
  - `i32 (uint32_t*) return_queue_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Register a shared queue using a given name (`queue_name_data`, `queue_name_size`). It can be
referred to in `proxy_enqueue_shared_queue` and `proxy_dequeue_shared_queue` using returned unique
queue identifier (`queue_id`).


### `proxy_resolve_shared_queue`

* params:
  - `i32 (const char*) queue_name_data`
  - `i32 (size_t) queue_name_size`
  - `i32 (uint32_t*) return_queue_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Resolves existing shared queue using a given name (`queue_name_data`, `queue_name_size`). It can be
referred to in `proxy_enqueue_shared_queue` and `proxy_dequeue_shared_queue` using returned unique
queue identifier (`queue_id`).


### `proxy_dequeue_shared_queue`

* params:
  - `i32 (uint32_t) queue_id`
  - `i32 (const char**) payload_data`
  - `i32 (size_t*) payload_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Get data (`payload_data`, `payload_size`) to the end of the queue (`queue_id`).


### `proxy_enqueue_shared_queue`

* params:
  - `i32 (uint32_t) queue_id`
  - `i32 (const char*) payload_data`
  - `i32 (size_t) payload_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Add data (`payload_data`, `payload_size`) to the front of the queue (`queue_id`).


### `proxy_remove_shared_queue`

* params:
  - `i32 (uint32_t) queue_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Remove queue (`queue_id`).


## Stats/metrics

### `proxy_define_metric`

* params:
  - `i32 (proxy_metric_type_t) metric_type`
  - `i32 (const char*) metric_name_data`
  - `i32 (size_t) metric_name_size`
  - `i32 (uint32_t*) return_metric_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Define a metric using a given name (`metric_name_data`, `metic_name_size`). It can be referred to in
`proxy_get_metric`, `proxy_increment_metric` and `proxy_record_metric` using returned unique metric
identifier (`metric_id`).


### `proxy_get_metric`

* params:
  - `i32 (uint32_t) metric_id`
  - `i32 (uint64_t*) return_value`
* returns:
  - `i32 (proxy_result_t) call_result`

Get the value of the metric (`metric_id`).


### `proxy_record_metric`

* params:
  - `i32 (uint32_t) metric_id`
  - `i64 (uint64_t) value
* returns:
  - `i32 (proxy_result_t) call_result`

Set the value of the metric (`metric_id`) to the value (`value`).


### `proxy_increment_metric`

* params:
  - `i32 (uint32_t) metric_id`
  - `i64 (int64_t) offset
* returns:
  - `i32 (proxy_result_t) call_result`

Increment/decrement value of the metric (`metric_id`) by offset (`offset`).


### `proxy_remove_metric`

* params:
  - `i32 (uint32_t) metric_id`
* returns:
  - `i32 (proxy_result_t) call_result`

Remove the metric (`metric_id`).


## Foreign function interface (FFI):

### `proxy_call_foreign_function`

* params:
  - `i32 (const char*) function_name_data`
  - `i32 (size_t) function_name_size`
  - `i32 (const char *) parameters_data`
  - `i32 (size_t) parameters_size`
  - `i32 (const char**) return_results_data`
  - `i32 (size_t*) return_results_size`
* returns:
  - `i32 (proxy_result_t) call_result`

Call registered foreign function (`function_name_data`, `function_name_size`).
