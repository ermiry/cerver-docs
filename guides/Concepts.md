# Concepts

The pieces of cerver, how a packet travels through them, and the rules of memory and threads that every application has to follow.

## The pieces

| Piece | Header | What it is |
|---|---|---|
| **Cerver** | `cerver/cerver.h` | A server: a listening socket (TCP) or a bound socket (UDP), its clients, its handlers, its threads and its configuration |
| **Client** | `cerver/client.h` | On the side of a cerver, a remote peer with one or more connections. On the side of an application, the object that connects to cervers. Both are the same `Client` structure |
| **Connection** | `cerver/connection.h` | A socket of a client, with its receive buffer, its state, its stats and, on the client side, its send queue and its reconnection |
| **Packet** | `cerver/packets.h` | A message of the cerver protocol: a 32 bytes header and its data |
| **Handler** | `cerver/handler.h` | A function of the application that gets the packets of a type, directly in the thread that received them or in its own thread with a queue |
| **Event** | `cerver/events.h` | A function of the application that runs when something happens, like a client that connects or a cerver that starts |
| **Error** | `cerver/errors.h` | A `PACKET_TYPE_ERROR` packet that a cerver sends to a client, and the function that a client runs when it gets one |
| **Admin** | `cerver/admin.h` | A dedicated connection to monitor and control a cerver |
| **Balancer** | `cerver/balancer.h` | A cerver that forwards the packets of its clients to its services, other cervers |

## Cerver types

`cerver_create ()` takes a `CerverType`, that selects the data that the cerver keeps for its application:

| Type | What it adds |
|---|---|
| `CERVER_TYPE_CUSTOM` | Nothing: the application decides everything, the most common type |
| `CERVER_TYPE_GAME` | A `GameCerver` with the game types, the lobbies and the players (see [[Game]]) |
| `CERVER_TYPE_FILES` | A `FileCerver` with the paths that the clients can upload to and download from (see [[Files]]) |
| `CERVER_TYPE_BALANCER` | Used by the load balancer for its own cerver (see [[Balancer]]) |
| `CERVER_TYPE_WEB` | Reserved for the HTTP cerver of the 2.x line; it has no implementation in the core |

## The path of a packet

```
          client                                   cerver
  ┌─────────────────────┐                ┌──────────────────────────────────┐
  │ packet_send_to ()   │ ── header ──▶  │ receive: poll () or its thread   │
  │                     │    + data      │   │                              │
  │                     │                │   ▼                              │
  │                     │                │ receive_handle_buffer ()         │
  │                     │                │   splits the bytes in packets    │
  │                     │                │   │                              │
  │                     │                │   ▼                              │
  │                     │                │ by packet type:                  │
  │                     │                │   CERVER, CLIENT, AUTH, REQUEST, │
  │                     │                │   GAME, TEST → cerver itself     │
  │                     │                │   APP, APP_ERROR, CUSTOM →       │
  │                     │                │     the handlers of the app      │
  │ client handlers  ◀──│ ◀── response ──│   │                              │
  └─────────────────────┘                │   ▼                              │
                                         │ handler: direct or its queue     │
                                         └──────────────────────────────────┘
```

1. The cerver receives the bytes of a connection in the thread of its handler type: the single thread of `poll ()` (`CERVER_HANDLER_TYPE_POLL`), or the thread of the connection (`CERVER_HANDLER_TYPE_THREADS`).
2. `receive_handle_buffer ()` decodes each header, waits for the data of a packet that came in pieces, and checks the size against `cerver_set_max_received_packet_size ()`.
3. The packet type decides who handles it. The protocol types (`PACKET_TYPE_CERVER`, `CLIENT`, `AUTH`, `REQUEST`, `GAME` and `TEST`) are handled by cerver itself: the info of the cerver, closing a connection, authentication, sessions, games and pings. `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` and `PACKET_TYPE_CUSTOM` go to the handlers of the application.
4. The handler gets the packet. A direct handler runs in the receiving thread; a handler with a queue gets it in its own thread, in order.
5. A packet of an unknown type, or one that fails `packet_check ()` when the cerver checks packets, is a **bad packet**: its sender gets a `CERVER_ERROR_PACKET_ERROR`, and a connection with `cerver_set_max_bad_packets ()` of them is dropped.

The client side is the same: its connection receives in its own thread and passes the packets of the application to the handlers of the client.

## Handlers

A `Handler` wraps an `Action`, `void (*) (void *)`, that is called with each packet:

| Kind | How | What the action gets | Where it runs |
|---|---|---|---|
| **Direct** | `handler_set_direct_handle (handler, true)` | The `Packet *` | In the thread that received the packet, so with `CERVER_HANDLER_TYPE_THREADS` it runs in many threads at the same time |
| **With a queue** (the default) | `handler_create ()` | A `HandlerData *` with `handler_id`, the `data` of the handler and the `packet` | In the thread of the handler, one packet at a time, in order |

A cerver has a handler for `PACKET_TYPE_APP` and `PACKET_TYPE_APP_ERROR` (`cerver_set_app_handlers ()`), one for `PACKET_TYPE_CUSTOM` (`cerver_set_custom_handler ()`), handlers by request type (`cerver_set_request_handler ()`) and several app handlers chosen by the `handler_id` of the header (`cerver_set_multiple_handlers ()`). [[Handlers]] has all of them.

## Rules of memory

- **A cerver, a client or a handler is owned by whoever created it until it is given away.** A handler given to a cerver or a client with a setter belongs to it, and it is deleted in its teardown. An event keeps its arguments, and deletes them with the `delete_action_args` that it was registered with.
- **A received packet belongs to cerver.** It is deleted after the handler returns, unless `cerver_set_app_handler_delete (cerver, false)` (and the setters of the other types) says that the application deletes it. Its data is only valid while the handler runs: copy what has to live longer.
- **A packet that the application creates** with `packet_create ()` and the others is deleted by the application with `packet_delete ()` after it is sent.
- **The data of an event** (`CerverEventData`, `ClientEventData`) is created for each trigger and released by the action with `cerver_event_data_delete ()` or `client_event_data_delete ()`.
- **`cerver_teardown ()` and `client_teardown ()`** release everything that the cerver or the client owns. If a connection is still working when the drain of the teardown times out, its memory is not released, to avoid a use after free.

## Rules of threads

- **The configuration is set before `cerver_start ()`**, from a single thread; the setters are not synchronized.
- **The handlers must be thread safe** with `CERVER_HANDLER_TYPE_THREADS`, or when they are direct handlers of a client with many connections: the same action runs in several threads at once.
- **`cerver_shutdown ()`** can be called from any thread or from a signal handler.
- **The stats** are updated with atomics from many threads; read them with `CERVER_STATS_GET ()` or with the getters, like `cerver_get_request_stats ()`.
- **A slow direct handler** blocks the thread that receives: with `CERVER_HANDLER_TYPE_POLL`, every connection waits for it. Use a handler with a queue for slow work.
- **Sending** locks the write mutex of the socket of the connection, so packets from different threads never mix in the same connection.

## Protocol and versions

Every packet starts with the same portable header, in network byte order, with the protocol id and version of the application (`packets_set_protocol_id ()`, `packets_set_protocol_version ()`). A cerver 1.8 does not talk with a cerver 1.7: [[Protocol]] has the format and [[Migration]] the changes.

## See also

- [[Lifecycle]]: the order of the calls
- [[Cerver]], [[Clients]], [[Handlers]], [[Packets]]: the reference of each piece
