# UDP

`#include <cerver/udp.h>`

A cerver created with `PROTOCOL_UDP` receives datagrams instead of connections. Each datagram carries a single packet of the cerver [[Protocol]], and the cerver keeps a **peer** for each address that sends it valid datagrams: a logical connection, with its `Connection` and its `Client`, so the handlers, the events, the authentication and the sessions work like with TCP. The client side is the same `Client`, with a connection of `PROTOCOL_UDP`.

## Concepts

### Peers

- A peer is created with the first **valid** datagram of an address: one that has a single packet of a valid size. A datagram of an unknown address that is not valid is dropped without creating anything.
- A peer is removed when it does not send anything in `cerver_set_udp_peer_timeout ()` seconds (16 by default), checked every `CERVER_UDP_PEERS_CHECK_INTERVAL` second, when its client disconnects, or when the cerver ends.
- The cerver keeps at most `cerver_set_udp_max_peers ()` peers (1024 by default); a datagram of a new address after it is dropped.
- A peer is released with its last reference: the peers of the cerver hold one, and every packet that a handler still has holds another, so a handler with a queue can use the connection of its packet after the peer expired.
- A peer has no socket of its own (its fd is -1): the cerver answers with its socket, with `sendto ()`.

### Authentication

With [[Auth]], a new peer is **pending**: it only sends authentication, ping and cerver info packets, it counts against the on hold connections, it expires with the on hold timeout, and it has the tries of the cerver. When it authenticates, the cerver gives it a **datagram token** of `CONNECTION_DATAGRAM_TOKEN_SIZE` (16) bytes, and from then on every datagram of the peer, in both ways, carries the token after the packet. The cerver checks it in constant time.

- **A peer that moves:** a datagram with the token of a peer from another address moves the peer to the new address, like a client that changed its network.
- **Sessions:** the peers that authenticate with the same session token share their `Client`, like the connections of a TCP client (see [[Sessions]]).
- **Admins:** a UDP cerver can not have admins; `cerver_start ()` fails if it has them.

### Amplification

A peer whose address is not proven (it did not authenticate) gets answers of the cerver of up to `CERVER_UDP_AMPLIFICATION_FACTOR` (4) times the bytes that it sent, so a forged address can not be used to flood a third party. A repeated authentication of a peer that already authenticated is only answered if its datagram is at least a quarter of the size of the answer.

### Threads

A UDP cerver ignores the handler type: a single thread receives every datagram, with `poll ()` and `recvfrom ()`. Direct handlers run in that thread; the handlers with a queue take the packets out of it.

### The size of a packet

A packet is sent in a single datagram, so it has to fit in `MAX_UDP_PACKET_SIZE` (65507 bytes, the payload of an IPv4 datagram), with the token. A bigger packet can not be sent.

## Usage

The cerver:

``` c
Cerver *cerver = cerver_create (
	CERVER_TYPE_CUSTOM, "udp-cerver",
	7000, PROTOCOL_UDP, false, CERVER_DEFAULT_CONNECTION_QUEUE
);

cerver_set_udp_peer_timeout (cerver, 32);

Handler *handler = handler_create (echo_handler);
handler_set_direct_handle (handler, true);
cerver_set_app_handlers (cerver, handler, NULL);

(void) cerver_start (cerver);
```

The client connects the same way as with TCP, with `PROTOCOL_UDP`: the connection asks for the info of the cerver and sends its authentication, if it has auth data.

``` c
Connection *connection = client_connection_create (client, "127.0.0.1", 7000, PROTOCOL_UDP, false);

(void) client_connect_and_start (client, connection);
```

## Reference

#### `cerver_set_udp_peer_timeout ()`

``` c
void cerver_set_udp_peer_timeout (Cerver *cerver, const u32 peer_timeout);
```

Sets the seconds without datagrams after which a peer is removed; 0 keeps the peers until the cerver ends. The default is `CERVER_UDP_DEFAULT_PEER_TIMEOUT`, 16.

#### `cerver_set_udp_max_peers ()`

``` c
void cerver_set_udp_max_peers (Cerver *cerver, const u32 max_peers);
```

Sets how many peers the cerver keeps at the same time; 0 does not limit them. The default is `CERVER_UDP_DEFAULT_MAX_PEERS`, 1024.

#### `cerver_udp_peers_count ()`

``` c
size_t cerver_udp_peers_count (const Cerver *cerver);
```

Returns the peers of the cerver, the pending ones too. It can be called from any thread.

## Types

| Type | What it is |
|---|---|
| `UdpPeer` | A peer: its `key` (ip:port), its `client` and `connection`, its `state`, its activity and its references |
| `UdpPeerState` | `UDP_PEER_STATE_PENDING` (not authenticated yet), `OPEN` (the cerver does not require authentication), `AUTHENTICATED` (with a token) |
| `UdpDatagram` | What follows the packet of a datagram: `UDP_DATAGRAM_INVALID`, `PLAIN` or `TOKEN` |
| `CerverUdp` | The peers of a cerver, by address, by token and by session |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CERVER_UDP_DEFAULT_PEER_TIMEOUT` | 16 | Seconds without datagrams |
| `CERVER_UDP_DEFAULT_MAX_PEERS` | 1024 | Peers at the same time |
| `CERVER_UDP_PEERS_CHECK_INTERVAL` | 1 | Seconds between the checks of the inactive peers |
| `CERVER_UDP_AMPLIFICATION_FACTOR` | 4 | The answers to an address that is not proven |
| `CERVER_UDP_PEER_KEY_SIZE` | | The ip with its zone and the port |
| `CONNECTION_DATAGRAM_TOKEN_SIZE` | 16 | The bytes of the token of a datagram |
| `MAX_UDP_PACKET_SIZE` | 65507 | The biggest packet |

## See also

- [[Protocol]]: the datagram on the wire, with its token
- [[Auth]] and [[Sessions]]: the authentication of a peer
- [[Network]]: the addresses and the families
