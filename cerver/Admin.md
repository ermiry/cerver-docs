# Admin

`#include <cerver/admin.h>`, and `cerver_set_admin_enable ()` of `#include <cerver/cerver.h>`.

An admin is a privileged client of a cerver: it connects to the same port, authenticates with its own method, and its packets go to their own handlers, apart from the ones of the clients. It is meant to monitor and control a cerver, like to ask for its stats or to change its configuration while it runs.

## Concepts

### How an admin connects

1. The cerver enables its admins with `cerver_set_admin_enable ()`, that creates its `AdminCerver` (`cerver->admin`). The cerver must require authentication (see [[Auth]]), because an admin arrives like any other connection, on hold.
2. The connection of the admin sets its credentials with `admin_auth` (`connection_set_auth_data (connection, data, size, delete, true)`), so it sends an `AUTH_PACKET_TYPE_ADMIN_AUTH`.
3. The authenticate method of the admins (`admin_cerver_set_authenticate ()`) checks them, with the same `AuthMethod` as the clients. On success, the connection becomes an `Admin`, with its own `Client`, and `CERVER_EVENT_ADMIN_CONNECTED` is triggered; a failed one triggers `CERVER_EVENT_ADMIN_FAILED_AUTH`.
4. The admin cerver receives the admins in its own thread, with `poll ()`, and passes their `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` and `PACKET_TYPE_CUSTOM` packets to its own handlers.

The cerver accepts at most `admin_cerver_set_max_admins ()` admins (1 by default), each one with at most `admin_cerver_set_max_admin_connections ()` connections (2). A UDP cerver can not have admins.

### The handlers of the admins

They are [[Handlers]] like the ones of the cerver: a direct handler gets the `Packet *`, and one with a queue gets a `HandlerData *`. The packet comes from the connection of the admin; `admin_get_by_sock_fd ()` returns its `Admin`. The packets are deleted after their handlers, unless the `*_handler_delete ()` setters say otherwise, and the admin cerver owns its handlers.

## Usage

``` c
static void admin_handler (void *packet_ptr) {

	const Packet *packet = (const Packet *) packet_ptr;

	// answer the admin with the stats of the cerver
	cerver_stats_print (packet->cerver, true, true);

}

(void) cerver_set_auth (cerver, MY_AUTH_TRIES, my_authenticate);

if (!cerver_set_admin_enable (cerver)) {
	admin_cerver_set_authenticate (cerver->admin, my_admin_authenticate);

	Handler *handler = handler_create (admin_handler);
	handler_set_direct_handle (handler, true);
	admin_cerver_set_app_handlers (cerver->admin, handler, NULL);
}
```

## Reference

### Configuration

All of them are set before `cerver_start ()`.

#### `admin_cerver_set_authenticate ()`

``` c
void admin_cerver_set_authenticate (AdminCerver *admin_cerver, delegate authenticate);
```

Sets the method that authenticates the admins; it gets an `AuthMethod *` and returns 0 to accept the admin (see [[Auth]]).

#### `admin_cerver_set_max_admins ()`

``` c
void admin_cerver_set_max_admins (AdminCerver *admin_cerver, u8 max_admins);
```

Sets how many admins can be connected at the same time. The default is `ADMIN_CERVER_DEFAULT_MAX_ADMINS`, 1.

#### `admin_cerver_set_max_admin_connections ()`

``` c
void admin_cerver_set_max_admin_connections (AdminCerver *admin_cerver, u8 max_admin_connections);
```

Sets how many connections each admin can have. The default is `ADMIN_CERVER_DEFAULT_MAX_ADMIN_CONNECTIONS`, 2.

#### `admin_cerver_set_bad_packets_limit ()`

``` c
void admin_cerver_set_bad_packets_limit (AdminCerver *admin_cerver, i32 n_bad_packets_limit);
```

Sets how many bad packets an admin connection can send before it is dropped; each one gets a `CERVER_ERROR_PACKET_ERROR`. 0 or less uses `ADMIN_CERVER_DEFAULT_MAX_BAD_PACKETS`, 4.

#### `admin_cerver_set_receive_buffer_size ()`

``` c
void admin_cerver_set_receive_buffer_size (AdminCerver *admin_cerver, const size_t buffer_size);
```

Sets the bytes of each receive of an admin connection. The default is `ADMIN_CERVER_DEFAULT_RECEIVE_BUFFER_SIZE`, 4096.

#### `admin_cerver_set_max_fds ()`

``` c
void admin_cerver_set_max_fds (AdminCerver *admin_cerver, u32 max_n_fds);
```

Sets the size of the poll array of the admins: the most admin connections at the same time. The default is `ADMIN_CERVER_DEFAULT_POLL_FDS`, 4. 0, and a call after the cerver started, are ignored.

#### `admin_cerver_set_poll_timeout ()`

``` c
void admin_cerver_set_poll_timeout (AdminCerver *admin_cerver, u32 poll_timeout);
```

Sets the timeout in milliseconds of each `poll ()` of the admins. The default is `ADMIN_CERVER_DEFAULT_POLL_TIMEOUT`, 2000.

#### `admin_cerver_set_check_packets ()`

``` c
void admin_cerver_set_check_packets (AdminCerver *admin_cerver, bool check_packets);
```

With `true`, the packets of the admins are checked with `packet_check ()`. The default is `false`.

### Handlers

#### `admin_cerver_set_app_handlers ()`

``` c
void admin_cerver_set_app_handlers (
	AdminCerver *admin_cerver,
	Handler *app_handler, Handler *app_error_handler
);
```

Sets the handlers of the `PACKET_TYPE_APP` and `PACKET_TYPE_APP_ERROR` packets of the admins. The admin cerver owns them.

#### `admin_cerver_set_custom_handler ()`

``` c
void admin_cerver_set_custom_handler (AdminCerver *admin_cerver, Handler *custom_handler);
```

Sets the handler of the `PACKET_TYPE_CUSTOM` packets of the admins.

#### `admin_cerver_set_app_handler_delete ()`, `admin_cerver_set_app_error_handler_delete ()`, `admin_cerver_set_custom_handler_delete ()`

``` c
void admin_cerver_set_app_handler_delete (AdminCerver *admin_cerver, bool delete_packet);
void admin_cerver_set_app_error_handler_delete (AdminCerver *admin_cerver, bool delete_packet);
void admin_cerver_set_custom_handler_delete (AdminCerver *admin_cerver, bool delete_packet);
```

With `false`, the packets of that type are not deleted after their handler, and the application deletes them. The default is `true`.

#### `admin_cerver_get_n_handlers_alive ()`, `admin_cerver_get_n_handlers_working ()`

``` c
unsigned int admin_cerver_get_n_handlers_alive (AdminCerver *admin_cerver);
unsigned int admin_cerver_get_n_handlers_working (AdminCerver *admin_cerver);
```

The handler threads of the admins that are running, and the handlers that are handling a packet.

### Updates

#### `admin_cerver_set_update ()`, `admin_cerver_set_update_interval ()`

``` c
void admin_cerver_set_update (
	AdminCerver *admin_cerver,
	Action update, void *update_args, void (*delete_update_args)(void *),
	const u8 fps
);

void admin_cerver_set_update_interval (
	AdminCerver *admin_cerver,
	Action update, void *update_args, void (*delete_update_args)(void *),
	const u32 interval
);
```

Run `update` `fps` times per second, or every `interval` seconds, in a thread of their own while the cerver runs, like the updates of a cerver: the action gets a `CerverUpdate *` with the cerver and the arguments.

### Admins

#### `admin_cerver_get_current_admins ()`

``` c
u8 admin_cerver_get_current_admins (AdminCerver *admin_cerver);
```

Returns how many admins are connected.

#### `admin_cerver_broadcast_to_admins ()`

``` c
u8 admin_cerver_broadcast_to_admins (AdminCerver *admin_cerver, Packet *packet);
```

Sends a generated packet to the first connection of every admin. The caller still owns the packet.

- **Returns** 0 on success, 1 on error.

#### `admin_cerver_broadcast_to_admins_split ()`, `admin_cerver_broadcast_to_admins_pieces ()`

``` c
u8 admin_cerver_broadcast_to_admins_split (AdminCerver *admin_cerver, Packet *packet);

u8 admin_cerver_broadcast_to_admins_pieces (
	AdminCerver *admin_cerver, Packet *packet,
	void **pieces, size_t *sizes, u32 n_pieces
);
```

The same, sending the header and the data apart (`packet_send_to_split ()`), or the header and then each piece (`packet_send_pieces ()`, see [[Packets]]).

#### `admin_send_packet ()`, `admin_send_packet_split ()`, `admin_send_packet_pieces ()`

``` c
u8 admin_send_packet (Admin *admin, Packet *packet);
u8 admin_send_packet_split (Admin *admin, Packet *packet);
u8 admin_send_packet_pieces (Admin *admin, Packet *packet, void **pieces, size_t *sizes, u32 n_pieces);
```

Send a packet to the first connection of an admin, whole, split, or in pieces.

- **Return** 0 on success, 1 on error.

#### `admin_get_by_sock_fd ()`, `admin_get_by_session_id ()`

``` c
Admin *admin_get_by_sock_fd (AdminCerver *admin_cerver, const i32 sock_fd);
Admin *admin_get_by_session_id (AdminCerver *admin_cerver, const char *session_id);
```

Return the admin with a connection of the socket, or with the session id of its client, or `NULL`.

#### `admin_set_data ()`

``` c
void admin_set_data (Admin *admin, void *data, Action delete_data);
```

Keeps data of the application in the admin, released with `delete_data` when the admin is deleted (`NULL` does not release it).

#### `admin_remove_connection_by_sock_fd ()`

``` c
AdminConnectionsStatus admin_remove_connection_by_sock_fd (
	AdminCerver *admin_cerver, Admin *admin, const i32 sock_fd
);
```

Removes a connection of an admin; an admin without connections left is dropped.

- **Returns** `ADMIN_CONNECTIONS_STATUS_ONE` if the admin still has a connection, `ADMIN_CONNECTIONS_STATUS_DROPPED` if it was dropped, or `ADMIN_CONNECTIONS_STATUS_ERROR`.

#### `admin_comparator_by_id ()`

``` c
int admin_comparator_by_id (const void *a, const void *b);
```

Compares two admins by their `id`, for the lists of admins.

### Stats and strings

#### `admin_cerver_stats_print ()`

``` c
void admin_cerver_stats_print (AdminCerverStats *stats);
```

Prints the stats of the admins: their packets and bytes, received and sent, and their connections (`cerver->admin->stats`).

#### `admin_connections_status_to_string ()`, `admin_connections_status_description ()`, `admin_cerver_handler_error_to_string ()`, `admin_cerver_handler_error_description ()`

``` c
const char *admin_connections_status_to_string (const AdminConnectionsStatus status);
const char *admin_connections_status_description (const AdminConnectionsStatus status);
const char *admin_cerver_handler_error_to_string (const AdminCerverHandlerError error);
const char *admin_cerver_handler_error_description (const AdminCerverHandlerError error);
```

The names and the descriptions of the two enums.

## Types

| Type | What it is |
|---|---|
| `AdminCerver` | The admins of a cerver, in `cerver->admin`; set it with the functions above |
| `Admin` | An admin: `id`, its `client` with its connections, its `data`, its `bad_packets` |
| `AdminCerverStats` | The packets, receives and bytes of the admins, their connections and their totals |
| `AdminConnectionsStatus` | `NONE`, `ERROR`, `ONE`, `DROPPED` |
| `AdminCerverHandlerError` | `NONE`, `PACKET`, `DROPPED` |

## Constants

| Constant | Value |
|---|---|
| `ADMIN_CERVER_DEFAULT_MAX_ADMINS` | 1 |
| `ADMIN_CERVER_DEFAULT_MAX_ADMIN_CONNECTIONS` | 2 |
| `ADMIN_CERVER_DEFAULT_MAX_BAD_PACKETS` | 4 |
| `ADMIN_CERVER_DEFAULT_RECEIVE_BUFFER_SIZE` | 4096 |
| `ADMIN_CERVER_DEFAULT_POLL_FDS` | 4 |
| `ADMIN_CERVER_DEFAULT_POLL_TIMEOUT` | 2000 |
| `ADMIN_CERVER_DEFAULT_CHECK_PACKETS` | `false` |
| `ADMIN_CERVER_DEFAULT_UPDATE_TICKS`, `ADMIN_CERVER_DEFAULT_UPDATE_INTERVAL_SECONDS` | 30, 1 |

## See also

- [[Auth]]: the authentication that the admins go through
- [[Handlers]]: how the handlers of the admins run
- [[Events]]: the events of the admins
