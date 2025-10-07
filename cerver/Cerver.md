# Cerver

`#include <cerver/cerver.h>`

A `Cerver` is a server: its socket, its clients and their connections, its handlers, its threads and its configuration. This page has how to create one, how it receives its connections, every setter of its configuration and its stats. [[Lifecycle]] has the order of the calls, [[Handlers]] the handlers of the packets, [[Auth]] the authentication, [[Sessions]] the sessions, and [[Configuration]] a table of every setter with its default.

## Concepts

### Handler types

A TCP cerver receives its connections in one of two ways, set with `cerver_set_handler_type ()`. It has no default: a TCP cerver with `CERVER_HANDLER_TYPE_NONE` fails to start.

| | `CERVER_HANDLER_TYPE_POLL` | `CERVER_HANDLER_TYPE_THREADS` |
|---|---|---|
| Accepting | The thread of `cerver_start ()`, in the same `poll ()` | The thread of `cerver_start ()`, with a blocking `accept ()` |
| Receiving | A single thread, the same `poll ()` over every connection | Each connection in its own thread: a thread of the thpool while it is connected, or a detachable thread |
| Direct handlers | In the thread of `poll ()`, so they block every connection while they run | In the thread of their connection, in parallel |
| Sending | The sockets block up to the send timeout, so a client that does not read delays every connection | Only the thread of that connection waits |
| Limits | `cerver_set_poll_fds ()`: the initial size of the poll array and the most connections | The threads of the thpool: with all of them busy, the overload policy decides |
| Good for | Many connections with short work, or handlers with their own queues | Fewer connections that live long and do work in their handlers |

A UDP cerver ignores the handler type: a single thread receives every datagram (see [[UDP]]).

### The thpool and the overload policy

Only a cerver with `CERVER_HANDLER_TYPE_THREADS` creates a thpool, of `cerver_set_thpool_n_threads ()` threads (4 by default). Each connection takes a thread for as long as it is connected, so the cerver receives at most that many connections at the same time. The connections that arrive when every thread is busy wait in the queue of the thpool (`cerver_set_thpool_queue_size ()`), and when that queue is full, the **overload policy** decides:

| Policy | What happens with a new connection |
|---|---|
| `CERVER_OVERLOAD_POLICY_WAIT` (the default) | The cerver stops accepting until the thpool has space; the new connections wait in the backlog of `listen ()` |
| `CERVER_OVERLOAD_POLICY_REJECT` | The new connection is closed right away, and counted in `total_rejected_connections` |

With `cerver_set_handle_detachable_threads (cerver, true)`, each connection gets a new detachable thread instead, without a limit; a cerver without a thpool also uses them.

### Cerver data

`cerver_set_cerver_data ()` keeps a pointer of the application in the cerver, released in its teardown with its delete function, or with `free ()` without one. A game or a files cerver already uses it for its `GameCerver` or `FileCerver`, so only a custom cerver sets its own.

## Usage

A cerver that receives each connection in a thread of a thpool of 8 threads, rejects the connections that do not fit, closes the clients that are inactive for a minute, and runs an update every second:

``` c
#define MY_THREADS				8
#define MY_INACTIVE_SECONDS		60
#define MY_INACTIVE_INTERVAL	30
#define MY_UPDATE_SECONDS		1

static void my_update (void *cerver_update_ptr) {

	CerverUpdate *update = (CerverUpdate *) cerver_update_ptr;

	cerver_stats_print (update->cerver, false, false);

}

Cerver *cerver = cerver_create (
	CERVER_TYPE_CUSTOM, "my-cerver",
	CERVER_DEFAULT_PORT, PROTOCOL_TCP, false, CERVER_DEFAULT_CONNECTION_QUEUE
);

cerver_set_handler_type (cerver, CERVER_HANDLER_TYPE_THREADS);
cerver_set_thpool_n_threads (cerver, MY_THREADS);
cerver_set_overload_policy (cerver, CERVER_OVERLOAD_POLICY_REJECT);

(void) cerver_set_bind_address (cerver, "::");
cerver_set_ipv6_only (cerver, false);

cerver_set_inactive_clients (cerver, MY_INACTIVE_SECONDS, MY_INACTIVE_INTERVAL);
cerver_set_update_interval (cerver, my_update, NULL, NULL, MY_UPDATE_SECONDS);
```

## Reference

### Global

#### `cerver_init ()`, `cerver_end ()`

``` c
void cerver_init (void);
void cerver_end (void);
```

`cerver_init ()` initializes the global values of cerver, the log, once at the start of the program. `cerver_end ()` releases them once at its end, after every cerver and client.

### Create, start and stop

#### `cerver_create ()`

``` c
Cerver *cerver_create (
	const CerverType type, const char *name,
	const u16 port, const Protocol protocol, bool use_ipv6,
	const u16 connection_queue
);
```

Creates a cerver with every default; nothing is opened until `cerver_start ()`.

- **`type`**: `CERVER_TYPE_CUSTOM`, `CERVER_TYPE_GAME` or `CERVER_TYPE_FILES` (see [[Concepts]]).
- **`name`**: up to `CERVER_INFO_NAME_SIZE - 1` (127) characters, sent to the clients in the info of the cerver.
- **`protocol`**: `PROTOCOL_TCP` or `PROTOCOL_UDP`.
- **`use_ipv6`**: binds every IPv6 address instead of every IPv4 one; `cerver_set_bind_address ()` replaces it.
- **`connection_queue`**: the backlog of `listen ()`, `CERVER_DEFAULT_CONNECTION_QUEUE` is `SOMAXCONN`.
- **Returns** the new cerver, or `NULL` on error.

#### `cerver_start ()`

``` c
u8 cerver_start (Cerver *cerver);
```

Starts the cerver and blocks while it runs (see [[Lifecycle]]). A stop requested before it cancels the start.

- **Returns** 0 after the cerver was stopped, 1 if it failed to start or it was already running. Either way, the cerver is released with `cerver_teardown ()`.

#### `cerver_is_running ()`

``` c
bool cerver_is_running (const Cerver *cerver);
```

Returns `true` while the cerver is accepting connections. It can be called from any thread.

#### `cerver_shutdown ()`

``` c
u8 cerver_shutdown (Cerver *cerver);
```

Stops accepting connections and wakes up `cerver_start ()`, that returns. It can be called from a signal handler, from another thread and more than once. The connections in progress are drained by `cerver_teardown ()`.

- **Returns** 0 on success, 1 for a `NULL` cerver.

#### `cerver_teardown ()`

``` c
u8 cerver_teardown (Cerver *cerver);
```

Stops the cerver if it is running, drains its connections with the drain policy and timeout, stops its handlers and threads and releases it. It also releases a cerver that never started or that failed to start. If some connections did not finish after the drain and its grace, the memory that they use is not released, to avoid a use after free. Never call it from a handler or an event of the same cerver.

- **Returns** 0 on success, 1 for a `NULL` cerver.

### Info

#### `cerver_set_alias ()`

``` c
void cerver_set_alias (Cerver *cerver, const char *alias);
```

Sets a short name of the cerver, up to `CERVER_INFO_ALIAS_SIZE - 1` (31) characters, used in the names of its threads. The default is `CERVER_DEFAULT_ALIAS`, `"my-cerver"`.

#### `cerver_set_welcome_msg ()`

``` c
u8 cerver_set_welcome_msg (Cerver *cerver, const char *msg);
```

Sets the message that the cerver sends to each client in its info when it connects, up to `CERVER_INFO_WELCOME_SIZE - 1` (511) characters; a longer one is cut.

- **Returns** 0 on success, 1 on error, like a `NULL` message.

#### `cerver_info_send_info_packet ()`

``` c
u8 cerver_info_send_info_packet (Cerver *cerver, Client *client, Connection *connection);
```

Sends the info of the cerver (a `CerverSummary`, see [[Protocol]]) to a connection. Cerver sends it when a connection is registered; an application only calls it to send it again.

- **Returns** 0 on success, 1 on error.

#### `cerver_type_to_string ()`, `cerver_handler_type_to_string ()`, `cerver_handler_type_description ()`, `cerver_overload_policy_to_string ()`, `cerver_overload_policy_description ()`, `cerver_drain_policy_to_string ()`, `cerver_drain_policy_description ()`

``` c
const char *cerver_type_to_string (CerverType type);
const char *cerver_handler_type_to_string (CerverHandlerType type);
const char *cerver_handler_type_description (CerverHandlerType type);
const char *cerver_overload_policy_to_string (CerverOverloadPolicy policy);
const char *cerver_overload_policy_description (CerverOverloadPolicy policy);
const char *cerver_drain_policy_to_string (CerverDrainPolicy policy);
const char *cerver_drain_policy_description (CerverDrainPolicy policy);
```

The name and the description of each value, like `"Poll"` and `"Handle connections using a single thread & poll ()"`. A value out of range gets the ones of `NONE`.

### Network

#### `cerver_set_network_values ()`

``` c
void cerver_set_network_values (
	Cerver *cerver,
	const u16 port, const Protocol protocol,
	bool use_ipv6, const u16 connection_queue
);
```

Replaces the network values of `cerver_create ()`, before the cerver starts.

#### `cerver_set_connection_queue ()`

``` c
void cerver_set_connection_queue (Cerver *cerver, const u16 connection_queue);
```

Sets the backlog of `listen ()`.

#### `cerver_set_bind_address ()`

``` c
unsigned int cerver_set_bind_address (Cerver *cerver, const char *address);
```

Sets the literal IPv4 or IPv6 address that the cerver binds, like `"127.0.0.1"`, `"::1"` or `"::"`, with a zone for a link local IPv6 address (`"fe80::1%eth0"`). Its family sets whether the cerver uses IPv6. `NULL` binds every address of the family again. Call it before `cerver_start ()`.

- **Returns** 0 on success, 1 if it is not a valid ip; a host name is not valid.

#### `cerver_set_ipv6_only ()`

``` c
void cerver_set_ipv6_only (Cerver *cerver, bool ipv6_only);
```

Sets `IPV6_V6ONLY` in the socket of an IPv6 cerver. With `false`, the default, a cerver bound to `"::"` also accepts IPv4 connections, that it sees as `::ffff:a.b.c.d` and normalizes to their IPv4 address.

#### `cerver_set_reusable_address_flags ()`

``` c
void cerver_set_reusable_address_flags (Cerver *cerver, bool value);
```

Sets `SO_REUSEADDR`, so the cerver can bind its port right after another one that used it closed. The default is `false`.

#### `cerver_set_reusable_port ()`

``` c
void cerver_set_reusable_port (Cerver *cerver, bool value);
```

Sets `SO_REUSEPORT`, so several processes listen in the same port and the kernel spreads the connections between them. Only enable it for that: with it, a second cerver binds the same port without an error. The default is `false`.

#### `cerver_set_ignore_sigpipe ()`

``` c
void cerver_set_ignore_sigpipe (Cerver *cerver, bool value);
```

Sets whether `cerver_start ()` ignores `SIGPIPE` when the process has not set a handler for it. Writing to a connection that the client reset raises `SIGPIPE`, that kills the process by default; cerver also sends with `MSG_NOSIGNAL`. The default is `true`.

#### `cerver_set_send_timeout ()`

``` c
void cerver_set_send_timeout (Cerver *cerver, u32 send_timeout);
```

Sets the deadline in seconds of each send to an accepted connection, so a client that never reads can not block a thread forever. 0 disables it. The default is `CERVER_DEFAULT_SEND_TIMEOUT`, 32 seconds.

#### `cerver_set_sockets_pool_init ()`

``` c
void cerver_set_sockets_pool_init (Cerver *cerver, unsigned int n_sockets);
```

Deprecated: it does nothing. Each connection owns its socket since cerver 1.8.

### Receiving

#### `cerver_set_handler_type ()`

``` c
void cerver_set_handler_type (Cerver *cerver, CerverHandlerType handler_type);
```

Sets how a TCP cerver receives its connections, `CERVER_HANDLER_TYPE_POLL` or `CERVER_HANDLER_TYPE_THREADS` (see the concepts above). It is required: the default, `CERVER_HANDLER_TYPE_NONE`, makes `cerver_start ()` fail.

#### `cerver_set_handle_detachable_threads ()`

``` c
void cerver_set_handle_detachable_threads (Cerver *cerver, bool active);
```

With `CERVER_HANDLER_TYPE_THREADS`, gives each connection a new detachable thread instead of a thread of the thpool. The teardown waits for them while it drains. The default is `false`.

#### `cerver_set_thpool_n_threads ()`

``` c
void cerver_set_thpool_n_threads (Cerver *cerver, u16 n_threads);
```

Sets the threads of the thpool, that only `CERVER_HANDLER_TYPE_THREADS` creates: the most connections that the cerver receives at the same time. The default is `CERVER_DEFAULT_POOL_THREADS`, 4. The handlers of the cerver must be thread safe, as they run in many threads.

#### `cerver_set_thpool_queue_size ()`

``` c
void cerver_set_thpool_queue_size (Cerver *cerver, unsigned int queue_size);
```

Sets how many accepted connections can wait in the queue of the thpool, rounded up to a power of 2. 0, the default, uses the threads times `THPOOL_DEFAULT_QUEUE_SIZE_FACTOR` (2).

#### `cerver_set_overload_policy ()`

``` c
void cerver_set_overload_policy (Cerver *cerver, CerverOverloadPolicy policy);
```

Sets what happens with a new connection when the queue of the thpool is full: `CERVER_OVERLOAD_POLICY_WAIT` (the default) or `CERVER_OVERLOAD_POLICY_REJECT`. An invalid value is ignored.

#### `cerver_set_poll_time_out ()`

``` c
void cerver_set_poll_time_out (Cerver *cerver, const u32 poll_timeout);
```

Sets the timeout in milliseconds of each `poll ()` of `CERVER_HANDLER_TYPE_POLL`. The default is `CERVER_DEFAULT_POLL_TIMEOUT`, 2000.

#### `cerver_set_poll_fds ()`

``` c
u8 cerver_set_poll_fds (Cerver *cerver, const u32 initial, const u32 max_connections);
```

Sets, before the cerver starts, the initial size of the array of `poll ()` (`CERVER_DEFAULT_POLL_FDS`, 128), that grows when it is full, and the most connections that it handles, 0 (the default) without a limit. A connection beyond the most is closed and counted in `total_rejected_connections`.

- **`initial`**: at least `CERVER_POLL_MIN_FDS` (2), the socket of the cerver and a connection.
- **Returns** 0 on success, 1 on error, like a cerver that already started.

#### `cerver_set_receive_buffer_size ()`

``` c
void cerver_set_receive_buffer_size (Cerver *cerver, const size_t size);
```

Sets the bytes of each `recv ()` of a connection. A packet bigger than the buffer arrives in several reads and is joined by cerver. The default is `CERVER_DEFAULT_RECEIVE_BUFFER_SIZE`, 4096; 0 is ignored.

#### `cerver_set_max_received_packet_size ()`

``` c
void cerver_set_max_received_packet_size (Cerver *cerver, size_t max_received_packet_size);
```

Sets the biggest packet, header included, that the cerver accepts. A bigger one closes its connection right away. The default is `CERVER_DEFAULT_MAX_RECEIVED_PACKET_SIZE`, 65507 bytes (`MAX_UDP_PACKET_SIZE`); 0 is ignored.

#### `cerver_set_handle_recieved_buffer ()`

``` c
void cerver_set_handle_recieved_buffer (Cerver *cerver, Action handle_received_buffer);
```

Replaces how the cerver handles the raw bytes that it receives, for an application with its own protocol. The action gets the `ReceiveHandle` of the read (see [[Packets]]); the packets and handlers of cerver are not used.

#### `cerver_set_check_packets ()`

``` c
void cerver_set_check_packets (Cerver *cerver, bool check_packets);
```

With `true`, every packet is checked with `packet_check ()`: its protocol id and its protocol version have to match the local ones (see [[Protocol]]). A packet that fails is a bad packet. The default is `false`.

#### `cerver_set_max_bad_packets ()`

``` c
void cerver_set_max_bad_packets (Cerver *cerver, u8 max_bad_packets);
```

Sets how many bad packets an authenticated connection can send before it is dropped. A bad packet fails `packet_check ()` or has an unknown type, and its sender gets a `CERVER_ERROR_PACKET_ERROR`. 0 disables the limit. The default is `CERVER_DEFAULT_MAX_BAD_PACKETS`, 16.

#### `cerver_set_inactive_clients ()`

``` c
void cerver_set_inactive_clients (
	Cerver *cerver,
	u32 max_inactive_time, u32 check_inactive_interval
);
```

Enables a thread that drops the clients that did not send or receive a packet in `max_inactive_time` seconds (0 for `CERVER_DEFAULT_MAX_INACTIVE_TIME`, 60), checked every `check_inactive_interval` seconds (0 for `CERVER_DEFAULT_CHECK_INACTIVE_INTERVAL`, 30).

### Drain

#### `cerver_set_drain_policy ()`

``` c
void cerver_set_drain_policy (Cerver *cerver, CerverDrainPolicy policy);
```

Sets what the teardown does with the connections that were still queued in the thpool: `CERVER_DRAIN_POLICY_RUN` (the default) handles them until the drain timeout, `CERVER_DRAIN_POLICY_REJECT` rejects them right away. An invalid value is ignored.

#### `cerver_set_drain_timeout ()`

``` c
void cerver_set_drain_timeout (Cerver *cerver, u32 drain_timeout);
```

Sets the seconds that the teardown waits for the connections in progress; 0 waits until every one has finished. The default is `CERVER_DEFAULT_DRAIN_TIMEOUT`, 32.

### Updates

#### `cerver_set_update ()`

``` c
void cerver_set_update (
	Cerver *cerver,
	Action update,
	void *update_args, void (*delete_update_args)(void *),
	const u8 fps
);
```

Runs `update` `fps` times per second in a thread of its own, while the cerver runs. The action gets a `CerverUpdate *` with the `cerver` and the `args`, that cerver creates and deletes. `delete_update_args` releases `update_args` in the teardown.

#### `cerver_set_update_interval ()`

``` c
void cerver_set_update_interval (
	Cerver *cerver,
	Action update,
	void *update_args, void (*delete_update_args)(void *),
	const u32 interval
);
```

Runs `update` every `interval` seconds in a thread of its own, the same way. The two updates can be used at the same time.

#### `cerver_update_delete ()`

``` c
void cerver_update_delete (void *cerver_update_ptr);
```

Deletes a `CerverUpdate`. Cerver deletes the one of each update itself; an application only needs it for one that it created.

### Data

#### `cerver_set_cerver_data ()`

``` c
void cerver_set_cerver_data (Cerver *cerver, void *data, Action delete_data);
```

Keeps `data` in the cerver, in `cerver->cerver_data`, released in its teardown with `delete_data`, or with `free ()` if it is `NULL`. Only a custom cerver sets its own: a game or a files cerver uses it.

### Admins

#### `cerver_set_admin_enable ()`

``` c
u8 cerver_set_admin_enable (Cerver *cerver);
```

Creates the `AdminCerver` of the cerver, that takes the connections of its admins (see [[Admin]]). A UDP cerver with admins fails to start.

- **Returns** 0 on success, 1 on error.

### Stats

The stats of a cerver are in `cerver->stats`, a `CerverStats` updated with atomics from many threads. Read a field with `CERVER_STATS_GET (cerver->stats->total_n_packets_received)`.

#### `cerver_stats_set_threshold_time ()`

``` c
void cerver_stats_set_threshold_time (Cerver *cerver, time_t threshold_time);
```

Sets how often, in seconds, the stats of the packets are reset. The default is a day.

#### `cerver_stats_print ()`

``` c
void cerver_stats_print (Cerver *cerver, bool received, bool sent);
```

Prints the stats of the cerver, and the packets received and sent by type when `received` and `sent` are `true`.

## Types

| Type | What it is |
|---|---|
| `Cerver` | A cerver; read its fields, set them only with the setters |
| `CerverType` | `CERVER_TYPE_NONE`, `CUSTOM`, `GAME`, `WEB`, `FILES`, `BALANCER` |
| `CerverHandlerType` | `CERVER_HANDLER_TYPE_NONE`, `POLL`, `THREADS` |
| `CerverOverloadPolicy` | `CERVER_OVERLOAD_POLICY_NONE`, `WAIT`, `REJECT` |
| `CerverDrainPolicy` | `CERVER_DRAIN_POLICY_NONE`, `RUN`, `REJECT` |
| `CerverInfo` | The `name`, the `alias`, the `welcome` message, the time it started and its `uptime` in seconds |
| `CerverStats` | The packets, receives and bytes from the clients and the on hold connections, the packets and bytes sent, the current and total clients and connections, the rejected connections, and the packets per type received and sent |
| `CerverUpdate` | What an update gets: `cerver` and `args` |
| `CerverSummary` | The info of a cerver that it sends to its clients (see [[Protocol]]) |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CERVER_DEFAULT_PORT` | 7000 | |
| `CERVER_DEFAULT_CONNECTION_QUEUE` | `SOMAXCONN` | The backlog of `listen ()` |
| `CERVER_DEFAULT_RECEIVE_BUFFER_SIZE` | 4096 | Bytes of each `recv ()` |
| `CERVER_DEFAULT_MAX_RECEIVED_PACKET_SIZE` | 65507 | The biggest packet that is accepted |
| `CERVER_DEFAULT_POOL_THREADS` | 4 | Threads of the thpool |
| `CERVER_DEFAULT_SEND_TIMEOUT` | 32 | Seconds of the deadline of each send |
| `CERVER_DEFAULT_POLL_FDS` | 128 | Initial size of the poll array |
| `CERVER_DEFAULT_POLL_TIMEOUT` | 2000 | Milliseconds of each `poll ()` |
| `CERVER_DEFAULT_DRAIN_TIMEOUT` | 32 | Seconds of the drain |
| `CERVER_DRAIN_GRACE_TIMEOUT` | 4 | Seconds more after a drain that timed out |
| `CERVER_OVERLOAD_WAIT_TIMEOUT` | 1024 | Milliseconds that a full thpool waits before checking if the cerver still runs |
| `CERVER_DEFAULT_MAX_BAD_PACKETS` | 16 | Bad packets before a connection is dropped |
| `CERVER_DEFAULT_MAX_INACTIVE_TIME`, `CERVER_DEFAULT_CHECK_INACTIVE_INTERVAL` | 60, 30 | Seconds of the inactive clients |
| `CERVER_DEFAULT_UPDATE_TICKS`, `CERVER_DEFAULT_UPDATE_INTERVAL_SECONDS` | 30, 1 | The defaults of the updates |
| `CERVER_DEFAULT_SOCKET_RECEIVE_TIMEOUT` | 5 | `SO_RCVTIMEO` in seconds of a connection with its own thread |
| `CERVER_INFO_NAME_SIZE`, `CERVER_INFO_ALIAS_SIZE`, `CERVER_INFO_WELCOME_SIZE` | 128, 32, 512 | The buffers of the info, with the NUL |
| `CERVER_BIND_ADDRESS_SIZE` | `NETWORK_ADDRESS_STRING_SIZE` | An IPv6 address with its zone |
| `CERVER_STATS_ADD (field, value)`, `CERVER_STATS_SUBTRACT (field, value)`, `CERVER_STATS_GET (field)` | | Atomic access to a stat |

[[Configuration]] has the defaults of the authentication, the on hold connections and the sessions.

## See also

- [[Lifecycle]]: start, stop, drain and teardown
- [[Handlers]]: the handlers of the packets and their setters
- [[Auth]]: `cerver_set_auth ()` and the on hold connections
- [[Sessions]]: `cerver_set_sessions ()`
- [[UDP]]: a UDP cerver
- [[Configuration]]: every setter with its default
