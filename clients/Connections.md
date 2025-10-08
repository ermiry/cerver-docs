# Connections

`#include <cerver/connection.h>`

A `Connection` is a socket of a [[Clients]]: on a cerver, an accepted connection of a remote client; in an application, a connection to a cerver, with the values to connect, the thread that receives its packets, its authentication, an optional send queue and its reconnection. This page has the configuration of the connections that an application creates with `client_connection_create ()`.

## Concepts

### Connecting

`client_connect* ()` resolves the host of the connection, that can be a name or a literal address, to every address of both families, alternating them and starting with the preferred one (`use_ipv6`). Each round tries every address with a non blocking `connect ()` that can take `connection_set_connect_timeout ()` milliseconds (4096 by default), with a new socket each time. Between the rounds the connection waits a backoff that starts at `CONNECTION_FIRST_RETRY_SECONDS` (2) and doubles, until it would pass `connection_set_max_sleep ()` seconds (60 by default): then the connection fails, with `CLIENT_EVENT_CONNECTION_FAILED`. The wait is cancelled when the connection or its client is torn down.

### Receiving

A started connection has an **update thread**: it receives with blocking `recv ()` calls in a buffer of `connection_set_receive_buffer_size ()` bytes (4096 by default), joins the packets that come in pieces, and runs the handlers of the client. With `connection_set_receive (connection, false)`, the connection does not get a thread, and the application receives by itself. `connection_set_custom_receive ()` replaces how the bytes are handled, for another protocol.

### Sending

`packet_send_to ()` sends right away, in the calling thread, locking the write mutex of the socket. With `connection_set_send_queue ()`, `connection_send_packet ()` queues the packet instead, and a dedicated thread sends it and deletes it: the caller never waits for the socket. A packet given to `connection_send_packet ()` belongs to the connection from then on.

A send to a socket that is full waits up to `connection_set_send_timeout ()` seconds (`CERVER_DEFAULT_SEND_TIMEOUT`, 32) before it fails, so a cerver that does not read can not block the client forever.

### Reconnecting

With `connection_set_attempt_reconnect (connection, wait_time)`, a connection that the cerver closes, or that is lost, does not end: its update thread closes the socket, waits `wait_time` seconds, connects with a new socket, and keeps receiving, triggering `CLIENT_EVENT_CONNECTED` again. It keeps trying until it connects or the application ends it.

### Ending from a handler

A handler can not end the connection that it runs in. `connection_drop_request ()` shuts the socket down instead: the packets that were already sent still arrive, and the update thread ends and removes the connection when the handler returns, like when the client closes it.

### States

`connection_get_state ()` returns where the connection is: `CONNECTION_STATE_CONNECTING`, `READY` (connected), `IDLE`, `WORKING`, `BUSY`, `UNAVAILABLE`, `DISCONNECTING` or `DISCONNECTED`.

## Usage

``` c
Connection *connection = client_connection_create (client, "cerver.example", 7000, PROTOCOL_TCP, false);

connection_set_name (connection, "main");
connection_set_max_sleep (connection, 30);
connection_set_connect_timeout (connection, 2000);
connection_set_receive_buffer_size (connection, 8192);

// the packets are sent by a thread of the connection
connection_set_send_queue (connection, 0);

// reconnect 4 seconds after the cerver goes away
connection_set_attempt_reconnect (connection, 4);

(void) client_connect_and_start (client, connection);
```

## Reference

### Configuration

All of them are set before the connection connects.

#### `connection_set_name ()`

``` c
void connection_set_name (Connection *connection, const char *name);
```

Sets the name of the connection, up to `CONNECTION_NAME_SIZE - 1` characters, used in the logs and in the names of its threads.

#### `connection_set_values ()`

``` c
void connection_set_values (
	Connection *connection,
	const char *ip_address, u16 port, Protocol protocol, bool use_ipv6
);
```

Replaces the host, the port, the protocol and the preferred family of the connection.

#### `connection_set_max_sleep ()`

``` c
void connection_set_max_sleep (Connection *connection, u32 max_sleep);
```

Sets the seconds of the backoff after which a connection stops trying (see above). The default is `CONNECTION_DEFAULT_MAX_SLEEP`, 60.

#### `connection_set_connect_timeout ()`

``` c
void connection_set_connect_timeout (Connection *connection, u32 timeout);
```

Sets the milliseconds that a `connect ()` to each address can take. The default is `CONNECTION_DEFAULT_CONNECT_TIMEOUT`, 4096.

#### `connection_set_receive ()`

``` c
void connection_set_receive (Connection *connection, bool receive);
```

With `false`, a started connection does not get an update thread. The default is `true`.

#### `connection_set_receive_flags ()`

``` c
void connection_set_receive_flags (Connection *connection, int flags);
```

Sets the flags of each `recv ()` of the update thread. The default is 0.

#### `connection_set_receive_buffer_size ()`

``` c
void connection_set_receive_buffer_size (Connection *connection, u32 size);
```

Sets the bytes of each `recv ()`. The default is `CONNECTION_DEFAULT_RECEIVE_BUFFER_SIZE`, 4096; 0 is ignored.

#### `connection_set_update_timeout ()`

``` c
void connection_set_update_timeout (Connection *connection, u32 timeout);
```

Sets the seconds that a `recv ()` of the update thread blocks before it checks if the connection still runs. The default is `CONNECTION_DEFAULT_UPDATE_TIMEOUT`, 2.

#### `connection_set_send_timeout ()`

``` c
void connection_set_send_timeout (Connection *connection, u32 timeout);
```

Sets the seconds that a send waits for space in a full socket before it fails; 0 uses `CERVER_DEFAULT_SEND_TIMEOUT`.

#### `connection_set_send_queue ()`

``` c
void connection_set_send_queue (Connection *connection, int flags);
```

Gives the connection a send queue and a thread that sends its packets, with these flags; the thread starts with the connection and is joined when it is deleted.

#### `connection_send_packet ()`

``` c
void connection_send_packet (Connection *connection, Packet *packet);
```

Queues a generated packet for the send thread, that sends it and deletes it. A packet that can not be queued, like with a connection without a send queue, is deleted.

#### `connection_set_custom_receive ()`

``` c
void connection_set_custom_receive (
	Connection *connection,
	u8 (*custom_receive) (void *custom_data_ptr, char *buffer, const size_t buffer_size),
	void *args, void (*args_delete)(void *)
);
```

Replaces how the update thread handles the bytes that it receives. The method gets a `ConnectionCustomReceiveData *` with the `client`, the `connection` and `args`, and the buffer with what was received; it returns 0 to keep receiving and 1 to end the connection.

#### `connection_set_received_data ()`

``` c
void connection_set_received_data (Connection *connection, void *data, size_t data_size, Action data_delete);
```

Keeps data in the connection, like the response of a request, released with `data_delete` when the connection is deleted. Setting it again does not release the previous data.

#### `connection_set_attempt_reconnect ()`

``` c
void connection_set_attempt_reconnect (Connection *connection, unsigned int wait_time);
```

Makes the connection reconnect when its cerver closes it or it is lost, waiting `wait_time` seconds before each attempt (see above).

### State

#### `connection_get_state ()`

``` c
ConnectionState connection_get_state (Connection *connection);
```

Returns the state of the connection, read with its lock.

#### `connection_state_string ()`, `connection_state_description ()`

``` c
const char *connection_state_string (const ConnectionState state);
const char *connection_state_description (const ConnectionState state);
```

The name and the description of a state.

#### `connection_drop_request ()`

``` c
u8 connection_drop_request (Connection *connection);
```

Asks to end the connection from one of its handlers (see above); the connection must not be used after it.

- **Returns** 0 on success, 1 on error.

#### `connection_get_values ()`

``` c
void connection_get_values (Connection *connection);
```

Fills the ip and the port of the connection from its address, like for an accepted connection.

### Lower level

The functions that `client_connection_create ()` and the cerver already call; an application only needs them to build connections by itself.

#### `connection_new ()`, `connection_create_empty ()`, `connection_create_complete ()`, `connection_create ()`

``` c
Connection *connection_new (void);
Connection *connection_create_empty (void);
Connection *connection_create_complete (void);
Connection *connection_create (const i32 sock_fd, const struct sockaddr_storage *address, const Protocol protocol);
```

Allocate a connection: empty, with its socket and its stats, ready to connect with its synchronization values, or from an accepted socket. Connections are recycled, like the clients.

#### `connection_delete ()`

``` c
void connection_delete (void *connection_ptr);
```

Deletes a connection. It does not close its socket: the end functions of the client do (see [[Clients]]).

#### `connection_comparator ()`

``` c
int connection_comparator (const void *a, const void *b);
```

Compares two connections by their sockets.

#### `connection_stats_new ()`, `connection_stats_print ()`

``` c
ConnectionStats *connection_stats_new (void);
void connection_stats_print (Connection *connection);
```

Allocate and print the stats of a connection: receives, packets and bytes, and the packets per type.

## Types

| Type | What it is |
|---|---|
| `Connection` | A connection: `name`, `ip`, `port`, `protocol`, its `socket`, its `state`, its `stats`, its `cerver_report` (the info of its cerver), its authentication, its send queue and its reconnection |
| `ConnectionState` | The states above |
| `ConnectionStats` | Receives, bytes and packets received and sent, and the packets per type |
| `ConnectionCustomReceiveData` | What a custom receive gets: `client`, `connection`, `args` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CONNECTION_NAME_SIZE`, `CONNECTION_IP_SIZE` | 64 | The buffers of the name and the ip |
| `CONNECTION_DEFAULT_MAX_SLEEP` | 60 | Seconds of the backoff |
| `CONNECTION_FIRST_RETRY_SECONDS` | 2 | The first wait of the backoff |
| `CONNECTION_DEFAULT_CONNECT_TIMEOUT` | 4096 | Milliseconds of each `connect ()` |
| `CONNECTION_CONNECT_POLL_INTERVAL` | 128 | Milliseconds between the checks of a `connect ()` that waits |
| `CONNECTION_DEFAULT_RECEIVE_BUFFER_SIZE` | 4096 | Bytes of each `recv ()` |
| `CONNECTION_DEFAULT_UPDATE_TIMEOUT` | 2 | Seconds of each `recv ()` of the update thread |
| `CONNECTION_DEFAULT_RECONNECT_WAIT_TIME` | 30 | Seconds before a reconnection |
| `CONNECTION_DEFAULT_MAX_AUTH_TRIES` | 2 | The authentication tries |
| `CONNECTION_DEFAULT_BAD_PACKETS` | 4 | |
| `CONNECTION_DATAGRAM_TOKEN_SIZE` | 16 | The token of an authenticated UDP connection |

## See also

- [[Clients]]: connecting, starting and ending the connections
- [[Auth]]: the auth data of a connection
- [[Sockets]]: the socket of a connection and its deadlines
