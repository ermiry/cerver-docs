# Receive

`#include <cerver/receive.h>`

How the bytes that a socket receives become packets. A `recv ()` can end anywhere: in the middle of a header, in the middle of the data of a packet, or with several packets at once. Every connection has a `ReceiveHandle` that keeps what was left of the previous `recv ()`, so the packets are assembled no matter where the buffers are split.

## Concepts

### The states of a receive

| State | What it means |
|---|---|
| `RECEIVE_HANDLE_STATE_NORMAL` | The next byte starts a header |
| `RECEIVE_HANDLE_STATE_SPLIT_HEADER` | The previous buffer ended in the middle of a header; its first bytes are kept in `header` |
| `RECEIVE_HANDLE_STATE_COMP_HEADER` | The header was completed with the bytes of this buffer |
| `RECEIVE_HANDLE_STATE_SPLIT_PACKET` | The data of a packet continues in the next buffer; the packet that is being filled is the `spare_packet` |
| `RECEIVE_HANDLE_STATE_LOST` | The stream can not be followed anymore, and the connection is closed |

A packet whose size is smaller than its header, or bigger than the maximum of the cerver or the client (`cerver_set_max_received_packet_size ()`, `client_set_max_received_packet_size ()`), closes its connection right away: its bytes can not be skipped safely. A header of another format, with unknown flags or with reserved bytes that are not zero closes it too (see [[Protocol]]).

### Bytes after a packet

Some packets are followed by bytes that are not packets, like the contents of a file after its `FileHeader` (see [[Files]]). The `trailing` bytes of the handle are the ones of the buffer that come after the packet that is being handled, and the upload handler takes them before reading the rest of the file from the socket.

### A custom receive

`cerver_set_handle_recieved_buffer ()` (see [[Cerver]]) and `connection_set_custom_receive ()` (see [[Connections]]) replace this assembly for an application with its own protocol: the method gets the raw bytes of each `recv ()`.

### Receive types

A cerver receives from three kinds of connections, the `type` of the handle: `RECEIVE_TYPE_NORMAL` (a client), `RECEIVE_TYPE_ON_HOLD` (a connection that has not authenticated, see [[Auth]]) and `RECEIVE_TYPE_ADMIN` (an [[Admin]]).

## Reference

#### `receive_error_to_string ()`

``` c
const char *receive_error_to_string (const ReceiveError error);
```

The name of the result of a `recv ()`: `RECEIVE_ERROR_NONE`, `TIMEOUT` (the receive timeout expired), `EMPTY` (the peer closed the connection) or `FAILED`.

#### `receive_type_to_string ()`

``` c
const char *receive_type_to_string (const ReceiveType type);
```

The name of a receive type.

#### `receive_handle_state_to_string ()`

``` c
const char *receive_handle_state_to_string (const ReceiveHandleState state);
```

The name of a state of a receive handle.

## Types

| Type | What it is |
|---|---|
| `ReceiveHandle` | The receive of a connection: its `type`, its `cerver`, `socket`, `connection`, `client`, `admin` and `lobby`, the `buffer` of the last `recv ()` and its `received_size`, its `state`, the part of a split `header`, the `spare_packet` that is being filled, and the `trailing` bytes |
| `ReceiveHandleState` | The states above |
| `ReceiveType` | `RECEIVE_TYPE_NONE`, `NORMAL`, `ON_HOLD`, `ADMIN` |
| `ReceiveError` | `RECEIVE_ERROR_NONE`, `TIMEOUT`, `EMPTY`, `FAILED` |

## See also

- [[Packets]]: the header that is decoded
- [[Protocol]]: the bytes on the wire
- [[Performance]]: the receive buffers and their sizes
