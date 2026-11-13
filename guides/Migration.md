# Migration

What changes for an application that moves from cerver 1.7 to 1.8, from 1.8 to 1.8.1, and from 1.8.1 to 1.8.2. The complete list of every beta, with the details of each change, is [`incompatibilities.md`](https://github.com/ermiry/cerver/blob/master/incompatibilities.md) in the repository.

## From 1.7 to 1.8

### The protocol

**A 1.8 does not talk with a 1.7**: clients, cervers, services and balancers are updated together.

- The header is 32 bytes in network byte order, written field by field (`PACKET_HEADER_SIZE`); `sizeof (PacketHeader)` is not its size on the wire anymore. It carries the protocol id and version of the sender, a `format`, `flags` and a `route_id` (see [[Protocol]]).
- The objects of cerver are CBOR objects of the [[Serializer]]: the info of a cerver is a `CerverSummary`, the session token a `SessionToken`, the errors an `ErrorMessage`, the file headers a `FileHeader`, and the game requests and answers the lobby objects. `SCerver`, `SToken`, `SError`, `SLobby`, `SGameSettings` and the `SString*` types were removed.
- The session token has its own auth request types, `AUTH_PACKET_TYPE_CLIENT_TOKEN` and `ADMIN_TOKEN`, set with `connection_set_auth_token ()`.
- A service behind a balancer answers with the `route_id` of the request (`packet_set_route_id ()`).

### Memory

- **Connections, sockets, clients, receives and packets are recycled**: a pointer to one of them is never valid after it was deleted, even when it still points to readable memory. Their stats, locks and conditions live inside them and must not be freed nor replaced.
- A `Packet` and its data are released with `packet_delete ()`; `packet_delete ()` calls the `release` of the packet first, so a packet filled by hand has it as `NULL`.

### Threads

- The handler threads are joinable and the teardowns join them: a handler that never returns blocks the teardown.
- The teardowns wait for the events that run in their own threads and drain the work in progress, with a timeout and a grace (see [[Lifecycle]]).
- Only `CERVER_HANDLER_TYPE_THREADS` creates the thpool; `thpool_add_work ()` fails with a full queue, and the overload policy decides what a full cerver does.
- `Pool`, `JobQueue`, `Worker` and the managers were rewritten: priorities, waits with timeouts, joinable workers, and `WorkerManager` (see [[Jobs]], [[Workers]], [[Managers]]).
- Every thread of cerver blocks `SIGINT` and `SIGTERM`; a signal that interrupts a receive (`EINTR`) does not end its connection.

### Network

- IPv4 and IPv6: `cerver_set_bind_address ()`, dual stack by default (`cerver_set_ipv6_only ()`), zones, and clients that try every address of a host (see [[Network]] and [[Connections]]).
- Every send uses `MSG_NOSIGNAL` and has a deadline: a full socket waits up to the send timeout instead of failing at once.
- UDP works, with peers, authentication with a token in each datagram, sessions and protection against amplification (see [[UDP]]).
- `MAX_UDP_PACKET_SIZE` is 65507, the default biggest packet of TCP too.

### API

- New: handlers by request type, request stats, `cerver_set_poll_fds ()`, the `BALANCER_TYPE_CUSTOM` balancer, `system.h`, `histogram.h`, `serializer.h`, `freelist.h`, `arena.h`.
- Removed or renamed: `cerver_sockets_pool_push`, `_pop` and `receive_handle_reset ()`; `ELEM_AT` is `ARRAY_ELEM_AT`; `isRunning` is `running`; `sock_set_reusable ()` no longer sets `SO_REUSEPORT`; `CERVER_DEFAULT_SOCKET_RECV_TIMEOUT` is `CERVER_DEFAULT_SOCKET_RECEIVE_TIMEOUT`.
- The functions with a format (`cerver_log* ()`, `str_create ()`, `c_string_create ()`, `thread_set_name ()`) are checked by the compiler: a `%ld` with a `u64` does not build with `WERROR=1`.
- Game: the lobbies have references and a recursive lock, `lobby_poll`, the update every frame, and only the owner inits, starts and destroys a game (see [[Game]]).

## From 1.8 to 1.8.1

The protocol does not change: a 1.8.1 talks with a 1.8.

- **The default session ids** are 64 hex characters of random bytes of the kernel; they were the hash of the time, and could be guessed.
- **A cerver that failed to start** is released with `cerver_teardown ()`; the examples no longer call the private `cerver_delete ()`.
- **The data of an authentication** (`AuthData.data`) becomes the data of the new client, or is released with its `delete_data`; an authenticate method that released it by itself must stop doing it.
- **The data of a handler** (`handler_set_data_delete ()`) is also released when the handler never had a thread, like a direct handler.
- **Events and errors with `drop_after_trigger`**: their arguments are released by the `*_data_delete ()` of their only trigger, not by the unregister. `CerverEventData` and `ClientEventData` have a new `owns_action_args`, and `CerverErrorEventData` and `ClientErrorData` a new `delete_action_args` and `owns_action_args` (ABI).
- **New events:** `CLIENT_EVENT_CERVER_INFO`, `CLIENT_EVENT_CERVER_TEARDOWN` and `CERVER_EVENT_ON_HOLD_DISCONNECTED` are triggered; a cerver that ends tells its clients before it shuts their connections down, and a client that gets it no longer stops all of its connections.
- **Stricter setters:** `cerver_set_multiple_handlers ()` with 0 or twice, `cerver_handlers_add ()` with an id out of range or repeated, `cerver_set_welcome_msg ()` with `NULL`, and the registers of events and errors with a type out of range are errors; the receive buffer sizes, the biggest packet and the poll of the admins ignore 0; `cerver_set_sessions ()` with `NULL` uses the default generator.
- **`connection_send_packet ()`** deletes a packet that it can not queue.
- **`file_read ()`** gives the bytes that it read, and takes a `NULL` size.
- **`c_string_concat_safe ()`** takes a `char *` destination (it was `const char *`).

## From 1.8.1 to 1.8.2

The protocol and the ABI do not change: a 1.8.2 talks with a 1.8.1, and a program built with the 1.8.1 runs with it.

- **The base64 macros** take their argument in parentheses: `base64_decode_len (size + 1)` is now the length of `size + 1` characters.
- **`va_copy`** always comes from `<stdarg.h>`, that `config.h` includes.
- **`cerver_udp_peers_count ()`**: a peer that it does not count anymore is already out of the stats of the cerver.
- **New:** the canonical base64url of [[Base64]] and `CERVER_HIDDEN` (see [[Configuration]]).

## See also

- [[Protocol]]: the 1.8 protocol on the wire
- [[Version]]: the version that a program was built with
