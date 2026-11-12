# Clients

`#include <cerver/client.h>`

A `Client` has two roles with the same structure:

- **In an application that connects to cervers**, it is the object that owns the connections, their handlers, their events and errors, and their files. A client can be connected to several cervers at once, one connection each, or have several connections to the same cerver with [[Sessions]].
- **In a cerver**, it is a remote peer that connected: the cerver creates one per new client, with its connections, its session id and its stats. The handlers get it in `packet->client`.

This page has both. The events and errors of a client are in [[Events]] and [[Errors]], its files in [[Files]], and each connection in [[Connections]].

## Concepts

### Connecting

| Function | Blocks | Starts receiving | Reads the info of the cerver first |
|---|---|---|---|
| `client_connect ()` | Yes | No | No |
| `client_connect_to_cerver ()` | Yes | No | Yes |
| `client_connect_async ()` | No, a thread | No | No |
| `client_connect_to_cerver_async ()` | No, a thread | No | Yes |
| `client_connect_and_start ()` | Yes | Yes | Through its receive thread |
| `client_connect_and_start_async ()` | No, a thread | Yes | Through its receive thread |

**Starting** a connection (`client_connection_start ()`, or the `_and_start` functions) creates its update thread, that receives its packets and runs the handlers of the client. Without it, the application receives by itself, with `client_connection_get_next_packet ()` or a request.

A connection keeps trying to connect during its max sleep (`connection_set_max_sleep ()`), with a backoff between the rounds, trying each address of the host. `client_teardown ()` cancels the backoff.

### Requests

`client_request_to_cerver ()` sends a packet and receives the response in the calling thread, with the handlers of the client; it is for a connection that was not started. `client_request_to_cerver_async ()` does it in a new thread and triggers `CLIENT_EVENT_CONNECTION_DATA` when the response was handled. Both expect a response of a single packet, and neither deletes the request.

### Ending

| Function | What it does |
|---|---|
| `client_connection_stop ()` | Closes the socket and marks the connection inactive; the cerver is not told |
| `client_connection_close ()` | Ends the connection and closes its socket, but keeps it |
| `client_connection_end ()` | Ends the connection and deletes it |
| `client_disconnect ()` | Closes every connection of the client |
| `client_teardown ()` | Ends everything and releases the client and its handlers |

`client_teardown ()` waits for the update threads, the async connections and requests, and the events in their own threads. It can not run in the update thread of one of its connections, like from a direct handler: there, use `client_teardown_async ()`.

## Usage

``` c
Client *client = client_create ();
client_set_name (client, "my-client");

Handler *handler = handler_create (my_client_handler);
handler_set_direct_handle (handler, true);
client_set_app_handlers (client, handler, NULL);

Connection *connection = client_connection_create (client, "127.0.0.1", 7000, PROTOCOL_TCP, false);
connection_set_max_sleep (connection, 30);

if (!client_connect_and_start (client, connection)) {
	Packet *packet = packet_create (PACKET_TYPE_APP, MY_REQUEST, &my_message, sizeof (my_message));
	if (packet && !packet_generate (packet)) {
		size_t sent = 0;
		(void) packet_send_to (packet, &sent, false, NULL, client, connection, NULL);
	}

	packet_delete (packet);
}

(void) client_connection_end (client, connection);
(void) client_teardown (client);
```

## Reference

### Create and configure

#### `client_create ()`

``` c
Client *client_create (void);
```

Creates a client with every default. Clients are recycled: a deleted client is kept to be reused, unless the library was built with a sanitizer.

- **Returns** the new client, or `NULL` on error.

#### `client_set_name ()`

``` c
void client_set_name (Client *client, const char *name);
```

Sets the name of the client, up to `CLIENT_NAME_SIZE - 1` characters, used in its logs.

#### `client_set_data ()`, `client_get_data ()`

``` c
void client_set_data (Client *client, void *data, Action delete_data);
void *client_get_data (Client *client);
```

Keep data of the application in the client, released with `delete_data` (or `free ()`) when it is replaced and when the client is deleted. On a cerver, the authentication can set it (see [[Auth]]).

#### `client_set_app_handlers ()`, `client_set_custom_handler ()`

``` c
void client_set_app_handlers (Client *client, Handler *app_handler, Handler *app_error_handler);
void client_set_custom_handler (Client *client, Handler *custom_handler);
```

Set the handlers of the `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` and `PACKET_TYPE_CUSTOM` packets that the connections of the client receive. The client owns them, and deletes the packets after them (see [[Handlers]]).

#### `client_set_max_received_packet_size ()`

``` c
void client_set_max_received_packet_size (Client *client, size_t max_received_packet_size);
```

Sets the biggest packet that the connections accept; a bigger one is a bad packet. The default is `CLIENT_DEFAULT_MAX_RECEIVED_PACKET_SIZE`, 65507.

#### `client_set_check_packets ()`

``` c
void client_set_check_packets (Client *client, bool check_packets);
```

With `true`, every packet is checked with `packet_check ()` (see [[Protocol]]). The default is `false`.

#### `client_set_max_bad_packets ()`

``` c
void client_set_max_bad_packets (Client *client, u8 max_bad_packets);
```

Sets how many bad packets a connection can receive before the client ends it; 0 disables the limit. The default is `CLIENT_DEFAULT_MAX_BAD_PACKETS`, 16.

### Connections

#### `client_connection_create ()`

``` c
Connection *client_connection_create (
	Client *client,
	const char *ip_address, u16 port,
	Protocol protocol, bool use_ipv6
);
```

Creates a connection to a host and registers it to the client. `ip_address` is a literal IPv4 or IPv6 address or a host name, resolved when it connects; `use_ipv6` is the family that is tried first.

- **Returns** the connection, owned by the client, or `NULL` on error.

#### `client_connection_register ()`, `client_connection_unregister ()`

``` c
int client_connection_register (Client *client, Connection *connection);
int client_connection_unregister (Client *client, Connection *connection);
```

Add an existing connection to the client, or take it out without deleting it.

- **Return** 0 on success, 1 on error.

#### `client_connection_add ()`, `client_connection_remove ()`

``` c
u8 client_connection_add (Client *client, Connection *connection);
u8 client_connection_remove (Client *client, Connection *connection);
```

Add a connection to the list of the client, or remove it, without touching any other structure.

- **Return** 0 on success, 1 on error.

#### `client_connect ()`, `client_connect_to_cerver ()`, `client_connect_async ()`, `client_connect_to_cerver_async ()`

``` c
unsigned int client_connect (Client *client, Connection *connection);
unsigned int client_connect_to_cerver (Client *client, Connection *connection);
unsigned int client_connect_async (Client *client, Connection *connection);
unsigned int client_connect_to_cerver_async (Client *client, Connection *connection);
```

Connect a connection, without starting it (see the table above). The `_to_cerver` ones read the info of the cerver first.

- **Return** 0 when the connection is connected (or its thread was created), 1 on error.

#### `client_connection_start ()`

``` c
int client_connection_start (Client *client, Connection *connection);
```

Starts the update thread of a connection that is connected, and its send queue if it has one.

- **Returns** 0 on success, 1 on error.

#### `client_connect_and_start ()`, `client_connect_and_start_async ()`

``` c
int client_connect_and_start (Client *client, Connection *connection);
u8 client_connect_and_start_async (Client *client, Connection *connection);
```

Connect and start a connection, in the calling thread or in a new one.

- **Return** 0 when the connection is started (or its thread was created), 1 on error.

#### `client_connection_get_next_packet ()`

``` c
unsigned int client_connection_get_next_packet (Client *client, Connection *connection);
```

Receives a complete packet from a connection that was not started, and handles it with the handlers of the client.

- **Returns** 0 on success, 1 on error.

#### `client_receive ()`

``` c
unsigned int client_receive (Client *client, Connection *connection);
```

Receives what is in the socket of a connection, into a new buffer, and handles its packets.

- **Returns** 0 on success, 1 on error, most likely because the connection ended.

### Requests

#### `client_request_to_cerver ()`

``` c
unsigned int client_request_to_cerver (Client *client, Connection *connection, Packet *request);
```

Sends a request and waits for its response, of a single packet, that the handlers of the client handle. The request is not deleted.

- **Returns** 0 when the response was handled, 1 on error.

#### `client_request_to_cerver_async ()`

``` c
unsigned int client_request_to_cerver_async (Client *client, Connection *connection, Packet *request);
```

The same in a new thread; `CLIENT_EVENT_CONNECTION_DATA` is triggered when the response was handled.

- **Returns** 0 if the thread was created, 1 on error.

#### `client_requests_wait ()`

``` c
void client_requests_wait (Client *client);
```

Waits until every async request and connection of the client has ended. It can not be called from the handlers of the client, that run in those threads.

### End

#### `client_connection_stop ()`, `client_connection_close ()`, `client_connection_end ()`

``` c
int client_connection_stop (Client *client, Connection *connection);
int client_connection_close (Client *client, Connection *connection);
int client_connection_end (Client *client, Connection *connection);
```

Stop, close or end a connection (see the table above).

- **Return** 0 on success, 1 on error.

#### `client_disconnect ()`

``` c
u8 client_disconnect (Client *client);
```

Closes every connection of the client.

- **Returns** 0 on success, 1 on error.

#### `client_got_disconnected ()`

``` c
void client_got_disconnected (Client *client);
```

Closes every connection and marks the client as not running, like when a cerver sends it a `CLIENT_PACKET_TYPE_DISCONNECT`.

#### `client_teardown ()`, `client_teardown_async ()`

``` c
u8 client_teardown (Client *client);
u8 client_teardown_async (Client *client);
```

End every connection and release the client, in the calling thread or in a new one.

- **Return** 0 on success, 1 on error, like a `client_teardown ()` from an update thread of the client.

#### `client_is_running ()`

``` c
bool client_is_running (const Client *client);
```

Returns `true` while the client has started connections.

### The clients of a cerver

#### `client_get_by_sock_fd ()`

``` c
Client *client_get_by_sock_fd (Cerver *cerver, i32 sock_fd);
```

Returns the client of a cerver with a connection of the socket, or `NULL`.

#### `client_drop ()`

``` c
void client_drop (Cerver *cerver, Client *client);
```

Unregisters a client from a cerver and deletes it with its connections.

#### `client_connection_drop ()`

``` c
u8 client_connection_drop (Cerver *cerver, Client *client, Connection *connection);
```

Closes a connection of a client of a cerver, removes it from the client and deletes it.

- **Returns** 0 on success, 1 on error.

#### `client_remove_connection_by_sock_fd ()`

``` c
ClientConnectionsStatus client_remove_connection_by_sock_fd (Cerver *cerver, Client *client, const i32 sock_fd);
```

Drops the connection of a socket, and the client if it has no connections left.

- **Returns** `CLIENT_CONNECTIONS_STATUS_ONE` if the client still has a connection, `DROPPED` if it was dropped, or `ERROR`.

#### `client_broadcast_to_all_avl ()`

``` c
void client_broadcast_to_all_avl (AVLNode *node, Cerver *cerver, Packet *packet);
```

Sends a generated packet to every connection of every client of a tree, like `cerver->clients->root`. Call it with `cerver->clients->mutex` locked, as the tree changes while the cerver runs.

#### `client_create_with_connection ()`

``` c
Client *client_create_with_connection (Cerver *cerver, const i32 sock_fd, const struct sockaddr_storage *address);
```

Creates a client with a connection of an accepted socket; cerver uses it for each new connection.

#### `client_new ()`, `client_delete ()`, `client_delete_dummy ()`

``` c
Client *client_new (void);
void client_delete (void *ptr);
void client_delete_dummy (void *ptr);
```

Allocate an empty client, delete a client with its connections and its data, and a delete function that does nothing, for the collections that must not delete the clients that they hold.

#### `client_comparator_client_id ()`, `client_comparator_session_id ()`

``` c
int client_comparator_client_id (const void *a, const void *b);
int client_comparator_session_id (const void *a, const void *b);
```

Compare two clients by their `id` or by their session id, for the collections of clients.

### Stats and strings

#### `client_stats_print ()`, `client_file_stats_print ()`

``` c
void client_stats_print (Client *client);
void client_file_stats_print (Client *client);
```

Print the receives, packets and bytes of the client, and its file requests and transfers.

#### `client_connections_status_to_string ()`, `client_connections_status_description ()`, `client_handler_error_to_string ()`, `client_handler_error_description ()`

``` c
const char *client_connections_status_to_string (const ClientConnectionsStatus status);
const char *client_connections_status_description (const ClientConnectionsStatus status);
const char *client_handler_error_to_string (const ClientHandlerError error);
const char *client_handler_error_description (const ClientHandlerError error);
```

The names and descriptions of the two enums.

## Types

| Type | What it is |
|---|---|
| `Client` | A client: `id`, `name`, its `connections` (a `DoubleList`), its session id, its `data`, its handlers, its events and errors, its files and its stats |
| `ClientStats` | Receives, bytes and packets received and sent, and the packets per type |
| `ClientFileStats` | The file requests and transfers |
| `ClientConnectionsStatus` | `NONE`, `ERROR`, `ONE`, `DROPPED` |
| `ClientHandlerError` | `NONE`, `PACKET`, `CLOSED`: the result of handling a packet |
| `ClientConnection` | A client and a connection, with the thread of an async connect |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CLIENT_NAME_SIZE` | 64 | The buffer of the name |
| `CLIENT_DEFAULT_NAME` | `"no-name"` | |
| `CLIENT_DEFAULT_MAX_RECEIVED_PACKET_SIZE` | 65507 | The biggest packet |
| `CLIENT_DEFAULT_MAX_BAD_PACKETS` | 16 | Bad packets before a connection ends |
| `CLIENT_FILES_MAX_PATHS` | 32 | The file paths of a client |
| `CLIENT_NEXT_PACKET_BUFFER_SIZE` | 4096 | The pieces in which `client_connection_get_next_packet ()` reads |

## See also

- [[Connections]]: the configuration of each connection
- [[Events]] and [[Errors]]: what a client can react to
- [[Lifecycle]]: the order of the calls
