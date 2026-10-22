# Properties

#### Proxy-Wasm properties

| Identifier | Name                                  |   Type   | Description                                                               |
|:----------:|:--------------------------------------|:--------:|:--------------------------------------------------------------------------|
|   0x0001   | `PLUGIN_NAME`                         | `string` | Plugin name                                                               |
|   0x0002   | `PLUGIN_ROOT_ID`                      | `string` | Plugin root ID                                                            |
|   0x0003   | `PLUGIN_VM_ID`                        | `string` | Plugin VM ID                                                              |


#### Downstream connection properties

| Identifier | Name                                  |   Type   | Description                                                               |
|:----------:|:--------------------------------------|:--------:|:--------------------------------------------------------------------------|
|   0x0101   | `DOWNSTREAM_CONNECTION_ID`            | `uint`   | Connection ID                                                             |
|   0x0102   | `DOWNSTREAM_REMOTE_IP`                | `string` | Remote address                                                            |
|   0x0103   | `DOWNSTREAM_REMOTE_PORT`              | `int`    | Remote port                                                               |
|   0x0104   | `DOWNSTREAM_LOCAL_IP`                 | `string` | Local address                                                             |
|   0x0105   | `DOWNSTREAM_LOCAL_PORT`               | `int`    | Local port                                                                |
|   0x0106   | `DOWNSTREAM_TLS_VERSION`              | `string` | Negotiated TLS version                                                    |
|   0x0107   | `DOWNSTREAM_TLS_REQUESTED_SNI`        | `string` | Requested TLS Server Name Indication (SNI)                                |
|   0x0108   | `DOWNSTREAM_TLS_PEER_CERT_VALIDATED`  | `bool`   | Peer's TLS certificate validation status                                  |
|   0x0109   | `DOWNSTREAM_TLS_PEER_CERT_SUBJECT`    | `string` | Peer's TLS certificate Subject                                            |
|   0x010A   | `DOWNSTREAM_TLS_PEER_CERT_DNS_SAN`    | `string` | First DNS entry in peer's TLS certificate Subject Alternative Name (SAN)  |
|   0x010B   | `DOWNSTREAM_TLS_PEER_CERT_URI_SAN`    | `string` | First URI entry in peer's TLS certificate Subject Alternative Name (SAN)  |
|   0x010C   | `DOWNSTREAM_TLS_PEER_CERT_SHA256`     | `string` | SHA256 digest of peer's TLS certificate                                   |
|   0x010D   | `DOWNSTREAM_TLS_LOCAL_CERT_SUBJECT`   | `string` | Local TLS certificate Subject                                             |
|   0x010E   | `DOWNSTREAM_TLS_LOCAL_CERT_DNS_SAN`   | `string` | First DNS entry in local TLS certificate Subject Alternative Name (SAN)   |
|   0x010F   | `DOWNSTREAM_TLS_LOCAL_CERT_URI_SAN`   | `string` | First URI entry in local TLS certificate Subject Alternative Name (SAN)   |
|   0x0110   | `DOWNSTREAM_TLS_LOCAL_CERT_SHA256`    | `string` | SHA256 digest of local TLS certificate                                    |


#### Upstream connection properties

| Identifier | Name                                  |   Type   | Description                                                               |
|:----------:|:--------------------------------------|:--------:|:--------------------------------------------------------------------------|
|   0x0201   | `UPSTREAM_CONNECTION_ID`              | `uint`   | Connection ID                                                             |
|   0x0202   | `UPSTREAM_REMOTE_IP`                  | `string` | Remote address                                                            |
|   0x0203   | `UPSTREAM_REMOTE_PORT`                | `int`    | Remote port                                                               |
|   0x0204   | `UPSTREAM_LOCAL_IP`                   | `string` | Local address                                                             |
|   0x0205   | `UPSTREAM_LOCAL_PORT`                 | `int`    | Local port                                                                |
|   0x0206   | `UPSTREAM_TLS_VERSION`                | `string` | Negotiated TLS version                                                    |
|   0x0207   | `UPSTREAM_TLS_REQUESTED_SNI`          | `string` | Requested TLS Server Name Indication (SNI)                                |
|   0x0208   | `UPSTREAM_TLS_PEER_CERT_VALIDATED`    | `bool`   | Peer's TLS certificate validation status                                  |
|   0x0209   | `UPSTREAM_TLS_PEER_CERT_SUBJECT`      | `string` | Peer's TLS certificate Subject                                            |
|   0x020A   | `UPSTREAM_TLS_PEER_CERT_DNS_SAN`      | `string` | First DNS entry in peer's TLS certificate Subject Alternative Name (SAN)  |
|   0x020B   | `UPSTREAM_TLS_PEER_CERT_URI_SAN`      | `string` | First URI entry in peer's TLS certificate Subject Alternative Name (SAN)  |
|   0x020C   | `UPSTREAM_TLS_PEER_CERT_SHA256`       | `string` | SHA256 digest of peer's TLS certificate                                   |
|   0x020D   | `UPSTREAM_TLS_LOCAL_CERT_SUBJECT`     | `string` | Local TLS certificate Subject                                             |
|   0x020E   | `UPSTREAM_TLS_LOCAL_CERT_DNS_SAN`     | `string` | First DNS entry in local TLS certificate Subject Alternative Name (SAN)   |
|   0x020F   | `UPSTREAM_TLS_LOCAL_CERT_URI_SAN`     | `string` | First URI entry in local TLS certificate Subject Alternative Name (SAN)   |
|   0x0210   | `UPSTREAM_TLS_LOCAL_CERT_SHA256`      | `string` | SHA256 digest of local TLS certificate                                    |


#### HTTP properties

| Identifier | Name                                  |   Type   | Description                                                               |
|:----------:|:--------------------------------------|:--------:|:--------------------------------------------------------------------------|
|   0x0301   | `HTTP_REQUEST_PROTOCOL`               | `string` | HTTP protocol version (`HTTP/1.0`, `HTTP/1.1`, `HTTP/2`, `HTTP/3`)        |
|   0x0302   | `HTTP_REQUEST_START_TIME`             | `??????` | Time of the first byte received                                           |
|   0x0303   | `HTTP_REQUEST_DURATION`               | `??????` | Total duration of HTTP request                                            |
|   0x0304   | `HTTP_REQUEST_CONTENT_SIZE`           | `int`    | Size of HTTP request body                                                 |
|   0x0305   | `HTTP_REQUEST_TOTAL_SIZE`             | `int`    | Total size of HTTP request (including HTTP headers and trailers)          |
|   0x0306   | `HTTP_RESPONSE_CONTENT_SIZE`          | `int`    | Size of HTTP response body                                                |
|   0x0307   | `HTTP_RESPONSE_TOTAL_SIZE`            | `int`    | Total size of HTTP response (including HTTP headers and trailers)         |


Identifiers below 0x2000 are reserved for standardized properties.

Numbers above that range are considered private and can be used without assignment in the registry.

Random number in the private range should be used for private extensions and properties under active development.

Once the property is finalized and implemented in a subset of hosts and SDKs, it will be assigned identifier in the standardized range.
