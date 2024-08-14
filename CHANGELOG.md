# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## 0.2.1 - 2020-08-04

### Added

- Added `proxy_get_log_level`.

## 0.2.0 - 2020-08-03

### Added

- Added `proxy_close_stream`.

- Added `proxy_on_foreign_function`.

### Changed

- `proxy_on_request_headers` and `proxy_on_response_headers` added `end_of_stream`
  flag as the 3rd argument.

- `proxy_get_header_map_value` now returns `NOT_FOUND` instead of `OK` with
  an empty value for non-existing keys.

- Replaced `proxy_continue_request` with `proxy_continue_stream(HTTP_REQUEST)`.

- Replaced `proxy_continue_response` with `proxy_continue_stream(HTTP_RESPONSE)`.

- Replaced `proxy_get_configuration` with `proxy_get_buffer_bytes(VM_CONFIGURATION)`
  and `proxy_get_buffer_bytes(PLUGIN_CONFIGURATION)`.

### Removed

- Removed `proxy_clear_route_cache`.

## 0.1.0 - 2020-02-29

### Added

- Initial release.
