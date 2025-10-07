# Handlers

`#include <cerver/handler.h>`, and the handler setters of `#include <cerver/cerver.h>`.

A handler is the function of the application that gets the packets of a type. This page has how the handlers run, the handlers of a cerver (by packet type, by request type and by id) and their stats. The handlers of a client are the same structure, set with the setters of [[Clients]]; how a cerver receives its connections, its **handler type**, is in [[Cerver]].

## Concepts

### Direct handlers and handlers with a queue

A `Handler` wraps an `Action`, `void (*) (void *)`, that is called with each packet. What it gets depends on how it runs:

| | Direct | With a queue (the default) |
|---|---|---|
| Enabled with | `handler_set_direct_handle (handler, true)` | `handler_create ()` alone |
| The action gets | The `Packet *` | A `HandlerData *`: `handler_id`, the `data` of the handler and the `packet` |
| Runs in | The thread that received the packet | The thread of the handler, that cerver starts with the cerver or the client |
| Order | In parallel, when several threads receive (`CERVER_HANDLER_TYPE_THREADS`, or a client with several connections) | One packet at a time, in the order they were queued |
| The data of the handler | Not used: the action only gets the packet | Passed in each `HandlerData` |
| Good for | Short work, with no delay | Slow work, that must not block the thread that receives |

``` c
// a direct handler gets the packet
static void direct_handler (void *packet_ptr) {

	const Packet *packet = (const Packet *) packet_ptr;

	(void) packet;

}

// a handler with a queue gets a HandlerData
static void queued_handler (void *handler_data_ptr) {

	const HandlerData *handler_data = (const HandlerData *) handler_data_ptr;

	const Packet *packet = handler_data->packet;
	MyData *my_data = (MyData *) handler_data->data;

	(void) packet;
	(void) my_data;

}
```

### The data of a handler

A handler with a queue can have its own data, passed in every `HandlerData`:

- `handler_set_data ()` sets it directly, or `handler_set_data_create ()` sets a function that creates it in the thread of the handler, before it handles any packet;
- `handler_set_data_delete ()` sets how it is released: once, when the thread of the handler ends, or when the handler is deleted if it never had a thread (a direct handler, or a cerver that never started).

A direct handler has no thread, so `handler_set_data_create ()` is never called for it.

### Who owns the packet

The cerver deletes a received packet after its handler returns. `cerver_set_app_handler_delete (cerver, false)`, `cerver_set_app_error_handler_delete ()` and `cerver_set_custom_handler_delete ()` give it to the application instead, that then deletes it with `packet_delete ()`; it is useful to keep a packet after the handler, like to answer it later from another thread. The packets of a client are always deleted after its handlers.

A packet without a handler, or that can not be queued, is deleted by cerver.

### Who owns the handler

A handler belongs to the cerver (or the client) from the moment that a setter takes it: `cerver_set_app_handlers ()`, `cerver_set_custom_handler ()`, `cerver_set_request_handler ()` and `cerver_handlers_add ()`. The teardown stops it and deletes it. A setter that fails leaves the handler to its caller.

### Which handler gets a packet

For each packet of `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` or `PACKET_TYPE_CUSTOM`, the cerver looks for, in this order:

1. the handler of its request type (`cerver_set_request_handler ()`), if it has one;
2. for `PACKET_TYPE_APP`, with multiple handlers (`cerver_set_multiple_handlers ()`), the handler of the `handler_id` of the header;
3. the handler of its packet type (`cerver_set_app_handlers ()`, `cerver_set_custom_handler ()`).

The other packet types are handled by cerver itself (see [[Concepts]]).

## Usage

A cerver with a direct handler for `PACKET_TYPE_APP`, a handler with a queue for one request type, and a custom handler:

``` c
#define MY_SLOW_REQUEST		4

Cerver *cerver = cerver_create (
	CERVER_TYPE_CUSTOM, "my-cerver",
	CERVER_DEFAULT_PORT, PROTOCOL_TCP, false, CERVER_DEFAULT_CONNECTION_QUEUE
);

cerver_set_handler_type (cerver, CERVER_HANDLER_TYPE_THREADS);

Handler *app_handler = handler_create (direct_handler);
handler_set_direct_handle (app_handler, true);
cerver_set_app_handlers (cerver, app_handler, NULL);

// the slow requests are handled one at a time, in their own thread
Handler *slow_handler = handler_create (queued_handler);
handler_set_data_create (slow_handler, my_data_create, NULL);
handler_set_data_delete (slow_handler, my_data_delete);
(void) cerver_set_request_handler (cerver, PACKET_TYPE_APP, MY_SLOW_REQUEST, slow_handler);

Handler *custom_handler = handler_create (queued_handler);
cerver_set_custom_handler (cerver, custom_handler);
```

Multiple app handlers, chosen by the `handler_id` of the header of each packet (see [[Packets]] to set it):

``` c
#define MY_HANDLERS		2

(void) cerver_set_multiple_handlers (cerver, MY_HANDLERS);

(void) cerver_handlers_add (cerver, handler_create_with_id (0, queued_handler));
(void) cerver_handlers_add (cerver, handler_create_with_id (1, queued_handler));
```

## Reference

### Handlers

#### `handler_create ()`

``` c
Handler *handler_create (Action handler_method);
```

Creates a handler with a queue that calls `handler_method` with each packet. Its id is -1, so it can not be added with `cerver_handlers_add ()`.

- **Returns** the new handler, or `NULL` if it could not be allocated.

#### `handler_create_with_id ()`

``` c
Handler *handler_create_with_id (int id, Action handler_method);
```

Creates a handler for the multiple handlers of a cerver, that gets the `PACKET_TYPE_APP` packets with `id` in the `handler_id` of their header.

- **`id`**: from 0 to the `n_handlers` of `cerver_set_multiple_handlers ()` minus 1.
- **Returns** the new handler, or `NULL` if it could not be allocated.

#### `handler_set_direct_handle ()`

``` c
void handler_set_direct_handle (Handler *handler, bool direct_handle);
```

With `true`, the handler runs its action in the thread that received the packet, with the `Packet *`, instead of queueing it for its own thread. The default is `false`. Set it before its cerver or its client starts, when the threads of the handlers are started.

#### `handler_set_data ()`

``` c
void handler_set_data (Handler *handler, void *data);
```

Sets the data of a handler with a queue, that its action gets in `HandlerData.data`. `handler_set_data_delete ()` releases it.

#### `handler_set_data_create ()`

``` c
void handler_set_data_create (
	Handler *handler,
	void *(*data_create) (void *args), void *data_create_args
);
```

Sets a function that creates the data of the handler in its thread, before it handles any packet, called with `data_create_args`. It replaces the data of `handler_set_data ()`. A direct handler never calls it.

#### `handler_set_data_delete ()`

``` c
void handler_set_data_delete (Handler *handler, Action data_delete);
```

Sets how the data of the handler is released: once, when the thread of the handler ends, or when the handler is deleted if it never had a thread. Without it, the data is not released.

#### `handler_get_stats ()`

``` c
void handler_get_stats (const Handler *handler, HandlerStats *stats);
```

Copies the stats of the packets that the handler handled into `stats`: a histogram of the microseconds of each packet and one of their sizes in bytes, with the header (see [[Histograms]]). It can be called while the handler runs.

#### `handler_stats_print ()`

``` c
void handler_stats_print (const Handler *handler);
```

Prints the stats of the handler with the log of cerver.

### The handlers of a cerver

#### `cerver_set_app_handlers ()`

``` c
void cerver_set_app_handlers (
	Cerver *cerver,
	Handler *app_handler, Handler *app_error_handler
);
```

Sets the handlers of `PACKET_TYPE_APP` and `PACKET_TYPE_APP_ERROR`; either can be `NULL`. The cerver owns them. Call it before `cerver_start ()`.

#### `cerver_set_custom_handler ()`

``` c
void cerver_set_custom_handler (Cerver *cerver, Handler *custom_handler);
```

Sets the handler of `PACKET_TYPE_CUSTOM`. The cerver owns it.

#### `cerver_set_app_handler_delete ()`, `cerver_set_app_error_handler_delete ()`, `cerver_set_custom_handler_delete ()`

``` c
void cerver_set_app_handler_delete (Cerver *cerver, bool delete_packet);
void cerver_set_app_error_handler_delete (Cerver *cerver, bool delete_packet);
void cerver_set_custom_handler_delete (Cerver *cerver, bool delete_packet);
```

With `false`, the packets of that type are not deleted after their handler returns, and the application deletes them with `packet_delete ()`. The default is `true`. It also applies to the handlers by request type and by id of the same packet type.

#### `cerver_set_request_handler ()`

``` c
u8 cerver_set_request_handler (
	Cerver *cerver,
	const PacketType packet_type, const u32 request_type,
	Handler *handler
);
```

Sets the handler of the packets of a request type of `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` or `PACKET_TYPE_CUSTOM`. The packets of the request types without one go to the handler of their packet type. The cerver owns the handler.

- **`request_type`**: below `CERVER_REQUEST_HANDLERS_MAX` (1024).
- **Returns** 0 on success, 1 on error: another packet type, a request type out of range, a request type that already has a handler, or a cerver that is running.

#### `cerver_set_multiple_handlers ()`

``` c
int cerver_set_multiple_handlers (Cerver *cerver, unsigned int n_handlers);
```

Enables the multiple app handlers: each `PACKET_TYPE_APP` packet goes to the handler of the `handler_id` of its header, added with `cerver_handlers_add ()`. A packet with an id that has no handler is deleted with a warning. The `handler_id` of the header is a byte, so more than 256 handlers are never used.

- **Returns** 0 on success, 1 on error: 0 handlers, a second call or a failed allocation.

#### `cerver_handlers_add ()`

``` c
u8 cerver_handlers_add (Cerver *cerver, Handler *handler);
```

Adds a handler of `handler_create_with_id ()` to the multiple handlers. The cerver owns it.

- **Returns** 0 on success, 1 on error: no multiple handlers, an id that is not below `n_handlers`, or an id that already has a handler; then the handler still belongs to the caller.

#### `cerver_get_n_handlers ()`

``` c
unsigned int cerver_get_n_handlers (Cerver *cerver);
```

Returns the `n_handlers` of `cerver_set_multiple_handlers ()`, 0 without multiple handlers.

#### `cerver_get_n_handlers_alive ()`

``` c
unsigned int cerver_get_n_handlers_alive (Cerver *cerver);
```

Returns how many handler threads of the cerver are running, 0 before the cerver starts.

#### `cerver_get_n_handlers_working ()`

``` c
unsigned int cerver_get_n_handlers_working (Cerver *cerver);
```

Returns how many handlers are handling a packet at this moment.

#### `cerver_handlers_print_info ()`

``` c
void cerver_handlers_print_info (Cerver *cerver);
```

Logs the handlers that are alive and working, with the debug log.

### Request stats

The cerver keeps a histogram of durations and one of sizes for each request type of `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` and `PACKET_TYPE_CUSTOM` that its handlers handled. The request types from 0 to `CERVER_REQUEST_STATS_SIZE - 1` (31) have their own, and the bigger ones share a last entry.

#### `cerver_get_request_stats ()`

``` c
u8 cerver_get_request_stats (
	const Cerver *cerver,
	const PacketType packet_type, const u32 request_type,
	RequestStats *stats
);
```

Copies the stats of a request type into `stats`; a request type of `CERVER_REQUEST_STATS_SIZE` or bigger gets the shared entry.

- **Returns** 0 on success, 1 on error, like another packet type.

#### `cerver_request_stats_print ()`

``` c
void cerver_request_stats_print (const Cerver *cerver);
```

Prints the stats of the request types that have packets.

### Handler errors

#### `cerver_handler_error_to_string ()`, `cerver_handler_error_description ()`

``` c
const char *cerver_handler_error_to_string (const CerverHandlerError error);
const char *cerver_handler_error_description (const CerverHandlerError error);
```

The name and the description of a `CerverHandlerError`, the result of cerver handling a packet: `CERVER_HANDLER_ERROR_NONE`, `CERVER_HANDLER_ERROR_PACKET` (a bad packet) or `CERVER_HANDLER_ERROR_DROPPED` (the connection was ended).

## Types

| Type | What it is |
|---|---|
| `Handler` | A handler. Its fields are read by cerver; set them with the functions above |
| `HandlerData` | What a handler with a queue gets: `int handler_id`, `void *data`, `Packet *packet` |
| `HandlerStats` | `Histogram durations` (microseconds) and `Histogram sizes` (bytes) |
| `HandlerType` | Who owns the handler: `HANDLER_TYPE_CERVER`, `HANDLER_TYPE_CLIENT` or `HANDLER_TYPE_ADMIN`, set by the setter that takes it |
| `RequestStats` | The same two histograms, for a request type |
| `CerverHandlerError` | See above |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CERVER_REQUEST_HANDLERS_MAX` | 1024 | The request types below it can have their own handler |
| `CERVER_REQUEST_STATS_SIZE` | 32 | The request types below it have their own stats |
| `CERVER_REQUEST_STATS_ENTRIES` | 33 | The entries of the stats of a packet type, with the shared one |
| `CERVER_REQUEST_PACKET_TYPES` | 3 | `PACKET_TYPE_APP`, `PACKET_TYPE_APP_ERROR` and `PACKET_TYPE_CUSTOM` |

## See also

- [[Cerver]]: the handler types (`CERVER_HANDLER_TYPE_POLL`, `CERVER_HANDLER_TYPE_THREADS`) and the thpool
- [[Clients]]: the handlers of a client
- [[Packets]]: the header, the request type and the handler id
- [[Jobs]]: the job queues of the handlers
