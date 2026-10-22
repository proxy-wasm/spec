# Host features

| Identifier | Name                           | Value                                                                                         |
|:----------:|:-------------------------------|:----------------------------------------------------------------------------------------------|
|   0x0001   | `HAS_CORE`                     | When non-zero, all baseline functions considered essential are supported.                     |
|   0x0002   | `HAS_LOGGING`                  | When non-zero, all baseline functions for logging are supported.                              |
|   0x0003   | `HAS_HTTP_HEADERS`             | When non-zero, only baseline functions related to HTTP headers are supported.                 |
|   0x0004   | `HAS_HTTP_WITH_BODY`           | When non-zero, all baseline functions for HTTP are supported.                                 |
|   0x0005   | `HAS_HTTP_CALLOUTS_BUFFERED`   | When non-zero, all baseline functions for buffered HTTP callouts are supported.               |
|   0x0006   | `HAS_HTTP_CALLOUTS_STREAMING`  | When non-zero, all baseline functions for streaming HTTP callouts are supported.              |
|   0x0007   | `HAS_GRPC`                     | When non-zero, all baseline functions for gRPC are supported.                                 |
|   0x0008   | `HAS_GRPC_CALLOUTS_BUFFERED`   | When non-zero, all baseline functions for buffered gRPC callouts are supported.               |
|   0x0009   | `HAS_GRPC_CALLOUTS_STREAMING`  | When non-zero, all baseline functions for streaming gRPC callouts are supported.              |
|   0x000A   | `HAS_TCP_FILTER`               | When non-zero, only baseline functions related to new TCP connections are supported.          |
|   0x000B   | `HAS_TCP_WITH_PAYLOAD`         | When non-zero, all baseline functions for TCP are supported.                                  |
|   0x000C   | `HAS_KEY_VALUE_STORES`         | When non-zero, all baseline functions for key-value stores are supported.                     |
|   0x000D   | `HAS_SHARED_QUEUES`            | When non-zero, all baseline functions for shared queues are supported.                        |
|   0x000E   | `HAS_TIMERS`                   | When non-zero, all baseline functions for timers are supported.                               |
|   0x000F   | `HAS_METRICS`                  | When non-zero, all baseline functions for metrics are supported.                              |
|   0x0010   | `HAS_PROPERTIES`               | When non-zero, all baseline functions for properties are supported.                           |
|   0x0011   | `HAS_CUSTOM_FUNCTIONS`         | When non-zero, all baseline functions for custom functions are supported.                     |
|   0x0F01   | `HAS_WASI_PREVIEW1_CORE`       | When non-zero, all baseline functions considered essential from WASI Preview1 are supported.  |


Identifiers below 0x2000 are reserved for standardized features and options.

Numbers above that range are considered private and can be used without assignment in the registry.

Random number in the private range should be used for private extensions and features under active development.

Once the feature is finalized and implemented in a subset of hosts and SDKs, it will be assigned identifier in the standardized range.
