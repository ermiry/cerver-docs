# Protocol

The bytes that a cerver and its clients send to each other, so a client can be written in any language. Every message is a **packet**: a header of 32 bytes, in network byte order (big endian), followed by its data. In TCP the packets follow each other in the stream; in UDP each datagram carries a single packet.

## The header

| Offset | Size | Field | What it is |
|---|---|---|---|
| 0 | 4 | `protocol_id` | The protocol id of the sender (`packets_set_protocol_id ()`) |
| 4 | 2 | `major` | The protocol version of the sender |
| 6 | 2 | `minor` | |
| 8 | 1 | `packet_type` | The `PacketType` |
| 9 | 1 | `handler_id` | The handler of a cerver with multiple app handlers, or the service of a balancer by handler id |
| 10 | 1 | `format` | The format of the header, `PACKET_HEADER_FORMAT` (1) |
| 11 | 1 | `flags` | `0x01` (`PACKET_FLAG_SERIALIZED`) when the data is a CBOR object; any other bit is not valid |
| 12 | 4 | `request_type` | The subtype of the packet |
| 16 | 4 | `route_id` | The route of a client of a load balancer, 0 without one |
| 20 | 8 | `packet_size` | The bytes of the whole packet, these 32 included |
| 28 | 4 | reserved | Always 0 |

A header with another `format`, with unknown `flags`, with reserved bytes that are not 0, or with a `packet_size` smaller than 32 or bigger than the maximum of the receiver, closes its connection. The receiver only checks the protocol id and version when it checks packets (see [[Packets]]).

## Packet types

| Value | Type | Request types |
|---|---|---|
| 1 | `CERVER` | 1 `INFO`: the info of the cerver, a `CerverSummary`; 2 `TEARDOWN`: the cerver is going to teardown |
| 2 | `CLIENT` | 1 `CLOSE_CONNECTION`: closes this connection; 2 `DISCONNECT`: disconnects the client |
| 3 | `ERROR` | The data is an `ErrorMessage` |
| 4 | `REQUEST` | 1 `GET_FILE`, 2 `SEND_FILE`: a `FileHeader`, followed by the file for a send |
| 5 | `AUTH` | 2 `CLIENT_AUTH`, 3 `ADMIN_AUTH`: the credentials; 4 `SUCCESS`: with a `SessionToken` when the cerver uses sessions; 5 `CLIENT_TOKEN`, 6 `ADMIN_TOKEN`: a `SessionToken` |
| 6 | `GAME` | 1 `GAME_INIT`, 2 `GAME_START`, 3 `LOBBY_CREATE`, 4 `LOBBY_JOIN`, 5 `LOBBY_LEAVE`, 6 `LOBBY_UPDATE`, 7 `LOBBY_DESTROY` (see [[Game]]) |
| 7 | `APP` | The application's own |
| 8 | `APP_ERROR` | The application's own |
| 9 | `CUSTOM` | The application's own |
| 10 | `TEST` | A ping: the cerver sends back the same packet |

## The objects of cerver

The data of the packets of cerver are objects of the [[Serializer]]: CBOR maps (RFC 8949) from field ids to values, with the version of the schema in the key 0, and `PACKET_FLAG_SERIALIZED` in the header. A reader ignores the keys that it does not know.

| Object | Fields (id: name) |
|---|---|
| `CerverSummary` | 1: `type` (enum), 2: `name` (text, required), 3: `welcome` (text), 4: `use_ipv6` (bool), 5: `protocol` (enum), 6: `port` (u16), 7: `auth_required` (bool), 8: `uses_sessions` (bool) |
| `SessionToken` | 1: `token` (text, required) |
| `ErrorMessage` | 1: `timestamp` (i64, seconds since the epoch), 2: `error_type` (u32, required), 3: `msg` (text) |
| `FileHeader` | 1: `filename` (text, required), 2: `len` (u64, the bytes of the file that follow the packet) |

The objects of the lobbies are in [[Lobby]].

## A connection

1. The client connects (TCP).
2. The cerver sends its info: `CERVER` / `INFO` with a `CerverSummary`.
3. If `auth_required` is true, the client sends `AUTH` / `CLIENT_AUTH` with its credentials, as raw bytes or as a CBOR object of the auth schema of the cerver, or `AUTH` / `CLIENT_TOKEN` with the `SessionToken` of a session. The cerver answers `AUTH` / `SUCCESS`, with the session token when it uses sessions, or an `ERROR` with `error_type` 3 (`FAILED_AUTH`) and the message; after its last try, the connection is closed.
4. The client and the cerver exchange their packets: the `APP`, `APP_ERROR` and `CUSTOM` ones are the application's.
5. Either side closes: the client with `CLIENT` / `CLOSE_CONNECTION` or by closing its socket; the cerver sends `CERVER` / `TEARDOWN` to every client before it ends.

## UDP

A datagram has a single packet, of up to `MAX_UDP_PACKET_SIZE` (65507) bytes with what follows it. After a peer authenticates, every datagram of it, in both ways, carries the **datagram token** of the peer, 16 bytes, right after the packet; the `packet_size` of the header does not count them. The `SUCCESS` of the authentication, that brings the token, is sent without one. See [[UDP]].

## Writing a client

- Read the 32 bytes of a header, check the `format`, the `flags` and the reserved bytes, and read `packet_size - 32` bytes of data.
- Decode the data of the packets of cerver with any CBOR library, by their field ids.
- Write the header of your packets with your protocol id and version, the `format` 1 and the `packet_size` of the whole packet; the cerver only checks the protocol when it checks packets.
- Copy the `route_id` of a request into its response when your program is a service behind a balancer (see [[Balancer]]).

## Constants

| Constant | Value |
|---|---|
| `PACKET_HEADER_SIZE` | 32 |
| `PACKET_HEADER_FORMAT` | 1 |
| `PACKET_FLAG_SERIALIZED` | `0x01` |
| `SERIALIZER_VERSION_KEY` | 0 |
| `CONNECTION_DATAGRAM_TOKEN_SIZE` | 16 |
| `MAX_UDP_PACKET_SIZE` | 65507 |

## See also

- [[Packets]]: the functions of the packets
- [[Serializer]]: the CBOR objects
- [[Migration]]: the protocol of cerver 1.7, that a 1.8 does not speak
