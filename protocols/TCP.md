# TCP

How cerver implements TCP: the listening socket, how it accepts, receives and sends, the limits that protect it, and how its connections end. A TCP cerver is created with `PROTOCOL_TCP` (see [[Cerver]]); its datagram counterpart is [[UDP]], and the bytes of its packets are in [[Protocol]].

## The listening socket

`cerver_start ()` creates a socket of the family of the bind address, sets its options and binds it:

| Option | When |
|---|---|
| `SO_REUSEADDR` | `cerver_set_reusable_address_flags (cerver, true)`: binds a port right after another socket closed it |
| `SO_REUSEPORT` | `cerver_set_reusable_port (cerver, true)`: several processes share the port and the kernel spreads the connections |
| `IPV6_V6ONLY` | An IPv6 cerver, `false` by default, so `"::"` also takes IPv4 connections (dual stack, see [[Network]]) |
| `O_NONBLOCK` | `CERVER_HANDLER_TYPE_POLL`: the listening socket is polled with the connections |

Then it listens with a backlog of `connection_queue` connections (`SOMAXCONN` by default). A failure in any step closes the socket once and `cerver_start ()` returns 1.

## Accepting

The thread of `cerver_start ()` accepts the connections:

- **`CERVER_HANDLER_TYPE_POLL`:** the listening socket is one more fd of the `poll ()` of the connections; when it is readable, the cerver accepts.
- **`CERVER_HANDLER_TYPE_THREADS`:** a blocking `accept ()`. Before each one, with a thpool that has no space and the `CERVER_OVERLOAD_POLICY_WAIT` policy, the cerver waits for space (`thpool_wait_space ()`), so the new connections stay in the backlog of the kernel instead of piling up in memory.

Every accepted socket is made **blocking**, gets `FD_CLOEXEC` (a child process does not inherit it) and, with a send timeout, `SO_SNDTIMEO` (see below). Then the connection is registered: with [[Auth]], on hold until it authenticates; without it, as a new client, that gets the info of the cerver.

### Out of file descriptors

When the process has no fds left, `accept ()` fails while the connection stays in the backlog, so the listening socket is always readable and the accept loop would spin. Cerver keeps one fd in reserve: it closes it, accepts the pending connection with that number and closes it at once, counts it in `total_rejected_connections`, waits 2^24 nanoseconds and opens its reserve again.

### Stopping

A signal that interrupts `accept ()` or `poll ()` (a handler without `SA_RESTART`) stops the cerver, and `cerver_shutdown ()` wakes up a thread that is blocked in them: the listening socket is shut down, so its fd can not be reused while the thread is still inside `accept ()`. See [[Lifecycle]].

## Receiving

| | `CERVER_HANDLER_TYPE_POLL` | `CERVER_HANDLER_TYPE_THREADS` |
|---|---|---|
| Who receives | The thread of `poll ()`, with every connection in its array | A thread per connection, from the thpool or detachable |
| Limit | `cerver_set_poll_fds ()`: the size of the array and the most connections | The threads of the thpool and its queue, and the overload policy |
| A `recv ()` | Only when `poll ()` says that the fd is readable | Blocking, with `SO_RCVTIMEO` of `CERVER_DEFAULT_SOCKET_RECEIVE_TIMEOUT` (5 s) so the thread checks if the cerver still runs |

Each `recv ()` reads up to `cerver_set_receive_buffer_size ()` bytes, and the bytes become packets no matter where the stream was split (see [[Receive]]). A packet bigger than `cerver_set_max_received_packet_size ()`, or a header that is not valid, closes the connection at once: TCP is a stream, and its bytes can not be skipped safely. A `recv ()` that returns 0 is a client that closed its connection.

## Sending

Every send of a packet or a file locks the **write mutex** of the socket of the connection, so the packets of different threads never mix, and goes through the sends with a **deadline** of [[Sockets]]:

- `MSG_NOSIGNAL` on every send, so a reset connection returns `EPIPE` instead of raising `SIGPIPE` (and `cerver_start ()` ignores `SIGPIPE` too, see [[Cerver]]);
- `MSG_DONTWAIT` and a wait with `poll ()` while the socket is full, so the deadline also holds in a blocking socket;
- the deadline is the send timeout of the cerver (`cerver_set_send_timeout ()`, 32 seconds) or of the connection: a client that never reads can not keep a thread blocked forever;
- the files go with `sendfile ()`, in the free space of the send buffer.

## Limits that protect a cerver

| Limit | Setter | What it stops |
|---|---|---|
| Connections being received | The thpool, or `cerver_set_poll_fds ()` | Running out of threads or of memory |
| Biggest packet | `cerver_set_max_received_packet_size ()` | A header that announces gigabytes |
| Bad packets | `cerver_set_max_bad_packets ()` | A peer that only sends garbage |
| Authentication | `cerver_set_on_hold_timeout ()`, the tries of `cerver_set_auth ()` | A connection that never authenticates |
| Inactive clients | `cerver_set_inactive_clients ()` | Connections that stay open without traffic |
| Send deadline | `cerver_set_send_timeout ()` | A client that does not read |

## Ending a connection

- **The client closes:** the next `recv ()` returns 0, and the connection is removed, with `CERVER_EVENT_CLIENT_CLOSE_CONNECTION` (and `CERVER_EVENT_CLIENT_DISCONNECTED` for the last connection of a client).
- **The cerver closes it:** a bad packet limit, an inactive client, a drop from a handler (`connection_drop_request ()`): the socket is shut down first, so the receive thread wakes up and removes the connection; it is never closed with its write mutex taken.
- **The cerver ends:** the teardown sends `CERVER_PACKET_TYPE_TEARDOWN` to every client, then shuts their sockets down and drains the work in progress (see [[Lifecycle]]).

## The client side

A client connects with a non blocking `connect ()` to each address of the host, with a timeout for each one and a backoff between the rounds, and then receives in the update thread of its connection, with blocking `recv ()` calls and its own receive timeout. Its sends use the same deadlines. See [[Connections]].

## See also

- [[UDP]]: the same cerver over datagrams
- [[Protocol]]: the header and the packets on the wire
- [[Performance]]: the costs of each handler type
