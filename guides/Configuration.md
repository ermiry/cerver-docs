# Configuration

Every setter of cerver with its default, in one place. Each one is described in the page of its module; all of them are called before the cerver or the connection starts, unless the page says otherwise.

## A cerver

| Setter | Default | Page |
|---|---|---|
| `cerver_set_handler_type ()` | `CERVER_HANDLER_TYPE_NONE`, required for TCP | [[Cerver]] |
| `cerver_set_bind_address ()` | Every address of the family | [[Cerver]] |
| `cerver_set_ipv6_only ()` | `false` (dual stack) | [[Cerver]] |
| `cerver_set_connection_queue ()` | `SOMAXCONN` | [[Cerver]] |
| `cerver_set_reusable_address_flags ()` | `false` | [[Cerver]] |
| `cerver_set_reusable_port ()` | `false` | [[Cerver]] |
| `cerver_set_ignore_sigpipe ()` | `true` | [[Cerver]] |
| `cerver_set_receive_buffer_size ()` | 4096 bytes | [[Cerver]] |
| `cerver_set_max_received_packet_size ()` | 65507 bytes | [[Cerver]] |
| `cerver_set_send_timeout ()` | 32 seconds | [[Cerver]] |
| `cerver_set_thpool_n_threads ()` | 4 | [[Cerver]] |
| `cerver_set_thpool_queue_size ()` | threads × 2 | [[Cerver]] |
| `cerver_set_overload_policy ()` | `CERVER_OVERLOAD_POLICY_WAIT` | [[Cerver]] |
| `cerver_set_handle_detachable_threads ()` | `false` | [[Cerver]] |
| `cerver_set_poll_time_out ()` | 2000 ms | [[Cerver]] |
| `cerver_set_poll_fds ()` | 128, without a max | [[Cerver]] |
| `cerver_set_drain_policy ()` | `CERVER_DRAIN_POLICY_RUN` | [[Cerver]] |
| `cerver_set_drain_timeout ()` | 32 seconds | [[Cerver]] |
| `cerver_set_check_packets ()` | `false` | [[Cerver]] |
| `cerver_set_max_bad_packets ()` | 16 | [[Cerver]] |
| `cerver_set_inactive_clients ()` | Off; 60 seconds, checked every 30 | [[Cerver]] |
| `cerver_set_update ()`, `cerver_set_update_interval ()` | Off | [[Cerver]] |
| `cerver_set_alias ()` | `"my-cerver"` | [[Cerver]] |
| `cerver_set_welcome_msg ()` | None | [[Cerver]] |
| `cerver_stats_set_threshold_time ()` | A day | [[Cerver]] |
| `cerver_set_app_handlers ()`, `cerver_set_custom_handler ()` | None | [[Handlers]] |
| `cerver_set_app_handler_delete ()` and the others | `true` | [[Handlers]] |
| `cerver_set_multiple_handlers ()` | Off | [[Handlers]] |
| `cerver_set_request_handler ()` | None | [[Handlers]] |
| `cerver_set_handle_recieved_buffer ()` | The packets of cerver | [[Receive]] |

## Authentication and sessions

| Setter | Default | Page |
|---|---|---|
| `cerver_set_auth ()` | Off; 2 tries | [[Auth]] |
| `cerver_set_auth_schema ()` | Credentials as bytes | [[Auth]] |
| `cerver_set_on_hold_timeout ()` | 16 seconds | [[Auth]] |
| `cerver_set_on_hold_poll_timeout ()` | 2000 ms | [[Auth]] |
| `cerver_set_on_hold_poll_fds ()` | 64 | [[Auth]] |
| `cerver_set_on_hold_max_bad_packets ()` | 4 | [[Auth]] |
| `cerver_set_on_hold_check_packets ()` | `false` | [[Auth]] |
| `cerver_set_on_hold_receive_buffer_size ()` | 4096 bytes | [[Auth]] |
| `cerver_set_sessions ()` | Off; the default generator with `NULL` | [[Sessions]] |

## UDP

| Setter | Default | Page |
|---|---|---|
| `cerver_set_udp_peer_timeout ()` | 16 seconds | [[UDP]] |
| `cerver_set_udp_max_peers ()` | 1024 | [[UDP]] |

## Admins

| Setter | Default | Page |
|---|---|---|
| `cerver_set_admin_enable ()` | Off | [[Admin]] |
| `admin_cerver_set_max_admins ()` | 1 | [[Admin]] |
| `admin_cerver_set_max_admin_connections ()` | 2 | [[Admin]] |
| `admin_cerver_set_bad_packets_limit ()` | 4 | [[Admin]] |
| `admin_cerver_set_receive_buffer_size ()` | 4096 bytes | [[Admin]] |
| `admin_cerver_set_max_fds ()` | 4 | [[Admin]] |
| `admin_cerver_set_poll_timeout ()` | 2000 ms | [[Admin]] |
| `admin_cerver_set_check_packets ()` | `false` | [[Admin]] |

## A client and its connections

| Setter | Default | Page |
|---|---|---|
| `client_set_max_received_packet_size ()` | 65507 bytes | [[Clients]] |
| `client_set_check_packets ()` | `false` | [[Clients]] |
| `client_set_max_bad_packets ()` | 16 | [[Clients]] |
| `connection_set_max_sleep ()` | 60 seconds | [[Connections]] |
| `connection_set_connect_timeout ()` | 4096 ms | [[Connections]] |
| `connection_set_receive ()` | `true` | [[Connections]] |
| `connection_set_receive_buffer_size ()` | 4096 bytes | [[Connections]] |
| `connection_set_update_timeout ()` | 2 seconds | [[Connections]] |
| `connection_set_send_timeout ()` | 32 seconds | [[Connections]] |
| `connection_set_send_queue ()` | Off | [[Connections]] |
| `connection_set_attempt_reconnect ()` | Off | [[Connections]] |

## Games

| Setter | Default | Page |
|---|---|---|
| `game_type_add_lobby_config ()` | The lobby poll, 4 players | [[Gametypes]] |
| `game_type_set_player_timeout ()` | 0, never | [[Gametypes]] |
| `game_type_set_fps ()` | 20 | [[Gametypes]] |
| `lobby_set_poll_time_out ()` | 2000 ms | [[Lobby]] |

## The log

| Setter | Default | Page |
|---|---|---|
| `cerver_log_set_output_type ()` | `LOG_OUTPUT_TYPE_STD` | [[Logs]] |
| `cerver_log_set_path ()` | `/var/log/cerver` | [[Logs]] |
| `cerver_log_set_time_config ()` | `LOG_TIME_TYPE_NONE` | [[Logs]] |
| `cerver_log_set_update_interval ()` | 1 second | [[Logs]] |
| `cerver_log_set_quiet ()` | `false` | [[Logs]] |

## `config.h`

`#include <cerver/config.h>`, that every header includes, defines `_GNU_SOURCE` and `_XOPEN_SOURCE 700`, and these macros:

| Macro | What it is |
|---|---|
| `CERVER_EXPORT` | A function of the API for the applications |
| `CERVER_PUBLIC` | A function that cerver uses and an application can also use |
| `CERVER_PRIVATE` | An internal function, not part of this reference |
| `CERVER_INLINE` | `inline` in C, nothing in C++ |
| `CERVER_ZERO_INITIALIZER` | `{ 0 }` in C and `{ }` in C++: a struct set to zero without warnings in both |
| `CERVER_PRINTF_FORMAT (format, first)` | Makes the compiler check the arguments against a `printf` format |
| `CERVER_ATTRS (x)` | `__attribute__ (x)` with gcc and clang |
| `DEPRECATED (func)` | Marks a function as deprecated |
| `MIN (a, b)`, `MAX (a, b)` | |
| `ARRAY_SIZE (a)`, `ARRAY_ELEM_AT (a, i, v)` | The elements of an array, and the element `i` or `v` when it is out of range |

## See also

- [[Cerver]], [[Clients]], [[Connections]]: the details of each setter
