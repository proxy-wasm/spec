# Host features

| Identifier | Name                     | Values                                                                                             | Deprecated |
|:----------:|:-------------------------|:---------------------------------------------------------------------------------------------------|:----------:|
|   0x0001   | `HAS_CORE`               | When present, all baseline functions considered essential are supported.                           |     No     |
|   0x0002   | `HAS_LOGGING`            | When present, all baseline functions for logging are supported.                                    |     No     |
|   0x0003   | `HAS_HTTP_HEADERS`       | When present, only baseline functions related to HTTP headers are supported.                       |     No     |
|   0x0004   | `HAS_HTTP_WITH_BODY`     | When present, all baseline functions for HTTP are supported.                                       |     No     |
|   0x0005   | `HAS_HTTP_CALLS`         | When present, all baseline functions for HTTP calls are supported.                                 |     No     |
|   0x0006   | `HAS_GRPC_CALLS`         | When present, all baseline functions for gRPC calls are supported.                                 |     No     |
|   0x0007   | `HAS_GRPC_STREAMS`       | When present, all baseline functions for gRPC streams are supported.                               |     No     |
|   0x0008   | `HAS_TCP_FILTER`         | When present, only baseline functions related to new TCP connections are supported.                |     No     |
|   0x0009   | `HAS_TCP_WITH_PAYLOAD`   | When present, all baseline functions for TCP are supported.                                        |     No     |
|   0x000A   | `HAS_KEY_VALUE_STORES`   | When present, all baseline functions for key-value stores are supported.                           |     No     |
|   0x000B   | `HAS_SHARED_QUEUES`      | When present, all baseline functions for shared queues are supported.                              |     No     |
|   0x000C   | `HAS_TIMERS`             | When present, all baseline functions for timers are supported.                                     |     No     |
|   0x000D   | `HAS_METRICS`            | When present, all baseline functions for metrics are supported.                                    |     No     |
|   0x000E   | `HAS_PROPERTIES`         | When present, all baseline functions for properties are supported.                                 |     No     |
|   0x000F   | `HAS_CUSTOM_FUNCTIONS`   | When present, all baseline functions for custom functions are supported.                           |     No     |
|   0x0F01   | `HAS_WASI_PREVIEW1_CORE` | When present, all baseline functions considered essential from WASI Preview1 are supported.        |     No     |


Identifiers below 0x2000 are reserved for standardized features and options.

Numbers above that range are considered private and can be used without assignment in the registry.

Random number in the private range should be used for private extensions and features under active development.

Once the feature is finalized and implemented in a subset of hosts and SDKs, it will be assigned an ID in the standardized range.
