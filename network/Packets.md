# Packets

`#include <cerver/packets.h>`

A packet is a message of the cerver [[Protocol]]: a header of 32 bytes and its data. This page has how to create, fill, send and check packets, the packet types and the protocol id and version of an application. The bytes on the wire are in [[Protocol]].

## Concepts

### The header

| Field | What it is |
|---|---|
| `packet_type` | The `PacketType`, a byte on the wire |
| `packet_size` | The bytes of the whole packet, the header included |
| `handler_id` | The handler of a cerver with multiple app handlers (see [[Handlers]]), or the service of a `BALANCER_TYPE_HANDLER_ID` balancer |
| `flags` | `PACKET_FLAG_SERIALIZED` when the data is an object of the [[Serializer]] |
| `request_type` | The subtype of the packet, chosen by the application for its own types |
| `route_id` | The route of a client of a load balancer (see [[Balancer]]) |
| `version` | The protocol id and version of the sender |

### Packet types

| Type | Who handles it |
|---|---|
| `PACKET_TYPE_CERVER` | Cerver: its info and its teardown (`CERVER_PACKET_TYPE_INFO`, `_TEARDOWN`) |
| `PACKET_TYPE_CLIENT` | Cerver: closing a connection and disconnecting a client (`CLIENT_PACKET_TYPE_CLOSE_CONNECTION`, `_DISCONNECT`) |
| `PACKET_TYPE_ERROR` | The error actions (see [[Errors]]) |
| `PACKET_TYPE_REQUEST` | Cerver: the file requests (`REQUEST_PACKET_TYPE_GET_FILE`, `_SEND_FILE`, see [[Files]]) |
| `PACKET_TYPE_AUTH` | Cerver: the authentication (see [[Auth]]) |
| `PACKET_TYPE_GAME` | A game cerver (see [[Game]]) |
| `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR`, `PACKET_TYPE_CUSTOM` | The handlers of the application |
| `PACKET_TYPE_TEST` | Cerver: a ping, answered with the same packet |
| `PACKET_TYPE_BAD` | Never sent: the type of a packet that could not be read |

### The life of a packet that is sent

1. **Create** it: `packet_create ()` with its type, request type and a copy of its data, `packet_create_serialized ()` with an object of a schema, or `packet_new ()` and the setters.
2. **Generate** it: `packet_generate ()` builds the bytes of the header and the data that are sent. A packet that changes after it has to be generated again.
3. **Send** it: `packet_send_to ()` with its destination, or `packet_set_network_values ()` and `packet_send ()`. Each send locks the write mutex of the socket, so packets from different threads never mix.
4. **Delete** it: `packet_delete ()`. The packets that the handlers get belong to cerver (see [[Handlers]]).

A packet can be sent many times, to many connections, while it is not changed. `packet_generate_request ()` and `packet_create_request ()` give packets that are ready to be sent.

### Protocol id and version

Each application can set a **protocol id**, a number of its own, and a **protocol version** (major and minor) with `packets_set_protocol_id ()` and `packets_set_protocol_version ()`. Every header carries them, and `packet_check ()` accepts a packet with the same id and major and a minor that is not newer than the local one. They are only checked by a cerver or a client with `*_set_check_packets (…, true)`.

### Recycling

The packets and their buffers of data, from 2^6 to 2^12 bytes, are kept to be reused instead of released: each thread keeps up to `PACKET_THREAD_CACHE_SIZE` (16) of each class, and the rest go to lists shared by every thread. A build with a sanitizer disables it, so every allocation is seen.

## Usage

``` c
#define MY_REQUEST		1

typedef struct MyMessage {

	u32 id;
	char text[64];

} MyMessage;

MyMessage message = { 7, "hello" };

Packet *packet = packet_create (PACKET_TYPE_APP, MY_REQUEST, &message, sizeof (MyMessage));
if (packet) {
	if (!packet_generate (packet)) {
		size_t sent = 0;
		(void) packet_send_to (packet, &sent, false, NULL, client, connection, NULL);
	}

	packet_delete (packet);
}
```

A packet with an object of a schema, and its decoding in the handler (see [[Serializer]]):

``` c
Packet *packet = packet_create_serialized (PACKET_TYPE_APP, MY_REQUEST, &my_object_schema, &object);

// in the handler
MyObject received;
if (packet_get_object (packet, &my_object_schema, NULL, &received) == SERIALIZER_RESULT_NONE) {
	// use received
}
```

## Reference

### Protocol

#### `packets_set_protocol_id ()`, `packets_get_protocol_id ()`

``` c
void packets_set_protocol_id (ProtocolID protocol_id);
ProtocolID packets_get_protocol_id (void);
```

Set and get the protocol id of the process, sent in every header. Set it once, at the start, before any cerver or client starts.

#### `packets_set_protocol_version ()`, `packets_get_protocol_version ()`

``` c
void packets_set_protocol_version (ProtocolVersion version);
ProtocolVersion packets_get_protocol_version (void);
```

Set and get the protocol version of the process, `{ major, minor }`.

#### `packet_version_local ()`

``` c
PacketVersion packet_version_local (void);
```

Returns the protocol id and version of the process, the ones that go in every header.

#### `packet_check ()`

``` c
bool packet_check (const Packet *packet);
```

Checks that a packet has the local protocol id, the same major and a minor that is not newer.

- **Returns** `true` for a compatible packet, `false` for a bad one.

#### `packet_version_new ()`, `packet_version_create ()`, `packet_version_delete ()`, `packet_version_copy ()`, `packet_version_print ()`

``` c
PacketVersion *packet_version_new (void);
PacketVersion *packet_version_create (void);
void packet_version_delete (PacketVersion *version);
u8 packet_version_copy (PacketVersion *dest, const PacketVersion *source);
void packet_version_print (const PacketVersion *version);
```

Allocate an empty version or one with the local values, delete it, copy it (0 on success, 1 on error) and print it.

### Create

#### `packet_create ()`

``` c
Packet *packet_create (const PacketType type, const u32 req_type, const void *data, const size_t data_size);
```

Creates a packet with a type, a request type and a copy of the data, that can be released after the call. It still has to be generated.

- **Returns** the packet, or `NULL` on error.

#### `packet_create_serialized ()`

``` c
Packet *packet_create_serialized (
	const PacketType type, const u32 request_type,
	const SerializerSchema *schema, const void *object
);
```

Creates a packet that is ready to be sent, with the object encoded by its schema as its data and `PACKET_FLAG_SERIALIZED` in its header.

- **Returns** the packet, or `NULL` if the object can not be encoded.

#### `packet_generate_request ()`

``` c
Packet *packet_generate_request (
	const PacketType packet_type, const u32 req_type,
	const void *data, const size_t data_size
);
```

Creates a packet that is ready to be sent, with a copy of the data.

- **Returns** the packet, or `NULL` on error.

#### `packet_create_request ()`, `packet_create_ping ()`

``` c
Packet *packet_create_request (const PacketType packet_type, const u32 request_type);
Packet *packet_create_ping (void);
```

Create a packet without data that is ready to be sent, or a `PACKET_TYPE_TEST` one.

#### `packet_init_request ()`, `packet_init_ping ()`

``` c
void packet_init_request (Packet *packet, const PacketType packet_type, const u32 request_type);
void packet_init_ping (Packet *packet);
```

Initialize a packet that the caller allocated, like on the stack, as a request without data or as a ping, ready to be sent without allocations.

#### `packet_new ()`, `packet_delete ()`

``` c
Packet *packet_new (void);
void packet_delete (void *ptr);
```

Allocate an empty packet, and delete a packet with its data (the data of a reference is not released). `packet_delete ()` calls the `release` of the packet first, if it has one.

### Data

#### `packet_set_data ()`

``` c
u8 packet_set_data (Packet *packet, const void *data, const size_t data_size);
```

Replaces the data of the packet with a copy of `data`.

- **Returns** 0 on success, 1 on error.

#### `packet_append_data ()`

``` c
u8 packet_append_data (Packet *packet, const void *data, const size_t data_size);
```

Appends a copy of `data` to the data of the packet, or creates its data. It does not work on data set as a reference.

- **Returns** 0 on success, 1 on error.

#### `packet_create_data ()`

``` c
unsigned int packet_create_data (Packet *packet, const size_t data_size);
```

Allocates a buffer of data of `data_size` bytes for the packet, to be filled after.

- **Returns** 0 on success, 1 on error.

#### `packet_set_data_ref ()`

``` c
u8 packet_set_data_ref (Packet *packet, void *data, size_t data_size);
```

Points the data of the packet to a buffer, without copying it or releasing it. With `packet_send ()` and `raw`, only those bytes are sent, without a header, like to talk to a peer that is not cerver.

- **Returns** 0 on success, 1 on error.

#### `packet_set_packet ()`, `packet_set_packet_ref ()`

``` c
u8 packet_set_packet (Packet *packet, void *data, size_t data_size);
u8 packet_set_packet_ref (Packet *packet, void *data, size_t packet_size);
```

Set the bytes that are sent, header included, as a copy or as a reference, for a packet built by hand. Do not generate the packet after them.

- **Return** 0 on success, 1 on error.

#### `packet_get_object ()`

``` c
SerializerResult packet_get_object (
	const Packet *packet, const SerializerSchema *schema,
	Arena *arena, void *object
);
```

Decodes the object of a packet with `PACKET_FLAG_SERIALIZED` with its schema. The arena gives the memory of its arena fields, `NULL` for a schema without them; the object is only written when the result is `SERIALIZER_RESULT_NONE`.

### Header

#### `packet_generate ()`

``` c
u8 packet_generate (Packet *packet);
```

Builds the bytes that the packet sends, its header and its data.

- **Returns** 0 on success, 1 on error.

#### `packet_set_header ()`, `packet_set_header_values ()`

``` c
void packet_set_header (Packet *packet, const PacketHeader *header);

void packet_set_header_values (
	Packet *packet,
	PacketType packet_type, size_t packet_size,
	u8 handler_id, u32 request_type,
	u32 route_id
);
```

Copy a header into the packet, or set its values, like a `handler_id`.

#### `packet_set_route_id ()`

``` c
void packet_set_route_id (Packet *packet, const u32 route_id);
```

Sets the route id of the packet, like the one of a request that came through a load balancer, so its response goes back to its client. The header of a generated packet is encoded again.

#### `packet_header_encode ()`, `packet_header_decode ()`

``` c
void packet_header_encode (const PacketHeader *header, u8 buffer[PACKET_HEADER_SIZE]);
PacketHeaderResult packet_header_decode (const u8 buffer[PACKET_HEADER_SIZE], PacketHeader *header);
```

Write a header in its wire format, and read one; the buffer does not need to be aligned. The header is only written when the result is `PACKET_HEADER_RESULT_NONE`; the others are `FORMAT` (another format of header), `FLAGS` (unknown flags) and `RESERVED` (the reserved bytes are not zero).

#### `packet_header_result_string ()`

``` c
const char *packet_header_result_string (const PacketHeaderResult result);
```

The description of a `PacketHeaderResult`.

#### `packet_header_new ()`, `packet_header_create ()`, `packet_header_create_from ()`, `packet_header_delete ()`, `packet_header_copy ()`, `packet_header_print ()`, `packet_header_log ()`

``` c
PacketHeader *packet_header_new (void);
PacketHeader *packet_header_create (const PacketType packet_type, const size_t packet_size, const u32 req_type);
PacketHeader *packet_header_create_from (const PacketHeader *source);
void packet_header_delete (PacketHeader *header);
u8 packet_header_copy (PacketHeader *dest, const PacketHeader *source);
void packet_header_print (const PacketHeader *header);
void packet_header_log (const PacketHeader *header);
```

Allocate a header (empty, with values, or as a copy), delete it, copy it (0 on success, 1 on error), and print it with `printf ()` or with the log.

### Send

#### `packet_set_network_values ()`

``` c
void packet_set_network_values (
	Packet *packet,
	Cerver *cerver, Client *client, Connection *connection, Lobby *lobby
);
```

Sets where `packet_send ()` sends the packet.

#### `packet_send ()`

``` c
u8 packet_send (const Packet *packet, int flags, size_t *total_sent, bool raw);
```

Sends a generated packet to its connection, with the flags of `send ()` (`MSG_NOSIGNAL` is always added) and the send timeout of the connection. With `raw`, only its data is sent, without the header. `total_sent` gets the bytes sent, and can be `NULL`.

- **Returns** 0 on success, 1 on error.

#### `packet_send_to ()`

``` c
u8 packet_send_to (
	const Packet *packet, size_t *total_sent, bool raw,
	Cerver *cerver, Client *client, Connection *connection, Lobby *lobby
);
```

Sends a packet to a connection without changing the packet, so many threads can send the same packet to different connections.

- **Returns** 0 on success, 1 on error.

#### `packet_send_split ()`, `packet_send_to_split ()`

``` c
u8 packet_send_split (const Packet *packet, int flags, size_t *total_sent);

u8 packet_send_to_split (
	const Packet *packet, size_t *total_sent,
	Cerver *cerver, Client *client, Connection *connection, Lobby *lobby
);
```

Send the header and then the data, from their own buffers, without joining them first; useful to forward a big packet. The write mutex is held for both.

- **Return** 0 on success, 1 on error.

#### `packet_send_pieces ()`

``` c
u8 packet_send_pieces (
	const Packet *packet,
	void **pieces, size_t *sizes, u32 n_pieces,
	int flags, size_t *total_sent
);
```

Sends the header of the packet and then each buffer of `pieces` with its size, under the write mutex. The `packet_size` of the header has to count them.

- **Returns** 0 on success, 1 on error.

#### `packet_send_to_socket ()`

``` c
u8 packet_send_to_socket (const Packet *packet, Socket *socket, int flags, size_t *total_sent, bool raw);
```

Sends a packet straight to a socket.

- **Returns** 0 on success, 1 on error.

#### `packet_send_unsafe ()`

``` c
u8 packet_send_unsafe (const Packet *packet, int flags, size_t *total_sent, bool raw);
```

Works like `packet_send ()` without locking the write mutex of the socket, for a caller that holds it.

- **Returns** 0 on success, 1 on error.

#### `packet_send_request ()`, `packet_send_ping ()`

``` c
u8 packet_send_request (
	const PacketType packet_type, const u32 request_type,
	Cerver *cerver, Client *client, Connection *connection, Lobby *lobby
);

u8 packet_send_ping (Cerver *cerver, Client *client, Connection *connection, Lobby *lobby);
```

Send a request without data, or a ping, without allocating a packet. A cerver answers a ping with the same packet.

- **Return** 0 on success, 1 on error.

#### `packet_route_between_connections ()`

``` c
u8 packet_route_between_connections (Connection *from, Connection *to, PacketHeader *header, size_t *sent);
```

Forwards a packet whose header was read from one connection to another: it sends the header, and moves the data between the two sockets with `splice ()` and a pipe, without copying it to user space. Both sockets are locked until the packet is forwarded.

- **Returns** 0 on success, 1 on error.

### Stats

#### `packets_per_type_new ()`, `packets_per_type_delete ()`, `packets_per_type_print ()`, `packets_per_type_array_print ()`

``` c
PacketsPerType *packets_per_type_new (void);
void packets_per_type_delete (void *packets_per_type_ptr);
void packets_per_type_print (const PacketsPerType *packets_per_type);
void packets_per_type_array_print (const u64 packets[PACKETS_MAX_TYPES]);
```

The counts of packets by type of the stats of a cerver, a client or a connection: allocate, delete and print them.

## Types

| Type | What it is |
|---|---|
| `Packet` | A packet: its network values, `packet_type`, `req_type`, its data (`data`, `data_size`), its `header`, its `version` and the bytes that it sends |
| `PacketHeader` | The fields of the header, see above |
| `PacketType` | The packet types, see above |
| `PacketVersion` | `protocol_id` and `protocol_version` |
| `ProtocolID` | `u32` |
| `ProtocolVersion` | `u16 major`, `u16 minor` |
| `PacketHeaderResult` | `NONE`, `FORMAT`, `FLAGS`, `RESERVED` |
| `PacketsPerType` | A count for each packet type, plus `unknown` and `bad` |
| `CerverPacketType`, `ClientPacketType`, `RequestPacketType`, `AuthPacketType`, `GamePacketType` | The request types of the packets that cerver handles |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `PACKET_HEADER_SIZE` | 32 | The bytes of a header on the wire |
| `PACKET_HEADER_FORMAT` | 1 | The format of the header |
| `PACKET_FLAG_SERIALIZED` | `0x01` | The data is an object of the serializer |
| `PACKET_FLAGS_KNOWN` | `PACKET_FLAG_SERIALIZED` | A header with another flag is a bad packet |
| `PACKET_HEADER_*_OFFSET` | | The offset of each field on the wire (see [[Protocol]]) |
| `PACKETS_MAX_TYPES`, `PACKETS_CURRENT_TYPES` | 16, 10 | The size of the counts by type, and the types in use |
| `PACKET_THREAD_CACHE_SIZE` | 16 | Recycled packets and buffers kept by each thread |
| `PACKET_DATA_MIN_EXPONENT`, `PACKET_DATA_MAX_EXPONENT` | 6, 12 | The classes of the recycled buffers of data |
| `PACKET_TYPE_MAP (XX)` and the other `*_PACKET_TYPE_MAP (XX)` | | The X-macros of the types |

## See also

- [[Protocol]]: the bytes of a packet on the wire
- [[Serializer]]: the objects in the data of a packet
- [[Receive]]: how the bytes of a socket become packets
- [[Handlers]]: who gets the packets
