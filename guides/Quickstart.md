# Quickstart

This page builds cerver, runs the echo example and explains each step of it. [[Building]] has every build option.

## 1. Get the sources

Clone the repository for the latest release in `master`, or download a release from the releases page of the repository:

``` bash
git clone https://github.com/ermiry/cerver.git
cd cerver
```

The sources of the library are in `src/cerver`, and its headers in `include/cerver`, with the same modules and names. `examples` has a program for each feature, and `test` has the tests.

## 2. Build

Cerver builds in Linux with gcc and `make`, and only needs pthreads, that comes with glibc:

``` bash
make -j4                # bin/libcerver.so
make examples -j4       # examples/bin, it needs bin/libcerver.so
```

`make examples` links the examples against `bin/libcerver.so`, so it has to run after `make`.

## 3. Run the echo example

[`examples/start.c`](https://github.com/ermiry/cerver/blob/master/examples/start.c) is an echo cerver and a client in a single program. Start the cerver in a terminal:

``` bash
LD_LIBRARY_PATH=bin ./examples/bin/start server
```

And the client in another one:

``` bash
LD_LIBRARY_PATH=bin ./examples/bin/start client
```

The client connects to `127.0.0.1:7000`, sends `Hello from cerver!` in a `PACKET_TYPE_APP` packet, waits for the cerver to send it back and exits with 0. Stop the cerver with `Ctrl + C`: the signal stops it, the cerver drains what is in progress and releases everything.

## 4. What the cerver does

``` c
#include <cerver/cerver.h>
#include <cerver/handler.h>
#include <cerver/packets.h>

static Cerver *echo_cerver = NULL;

// sends the data of every PACKET_TYPE_APP packet back to its connection
static void echo_handler (void *data) {

	const Packet *packet = (const Packet *) data;

	Packet *response = packet_create (
		PACKET_TYPE_APP, packet->header.request_type,
		packet->data, packet->data_size
	);

	if (!response) return;

	if (!packet_generate (response)) {
		size_t sent = 0;
		(void) packet_send_to (
			response, &sent, false,
			packet->cerver, packet->client, packet->connection, NULL
		);
	}

	packet_delete (response);

}

static void echo_stop (int signal_number) {

	(void) signal_number;

	if (echo_cerver) (void) cerver_shutdown (echo_cerver);

}
```

1. **`cerver_init ()`** prepares the global values of cerver, once at the start of the program, and **`cerver_end ()`** releases them at its end.
2. **`cerver_create ()`** creates a cerver of a type (`CERVER_TYPE_CUSTOM`), with a name, a port, a protocol (`PROTOCOL_TCP`), whether it uses IPv6 and the size of the queue of `listen ()`. Nothing is opened yet.
3. **`cerver_set_handler_type ()`** chooses how the connections are received. `CERVER_HANDLER_TYPE_POLL` receives every connection in a single thread with `poll ()`; `CERVER_HANDLER_TYPE_THREADS` receives each one in its own thread (see [[Handlers]]).
4. **`handler_create ()`** wraps a function that gets each packet. With **`handler_set_direct_handle ()`** the function runs in the thread that received the packet; without it, the handler has its own thread and a queue. **`cerver_set_app_handlers ()`** gives the cerver the handlers of `PACKET_TYPE_APP` and `PACKET_TYPE_APP_ERROR`, and the cerver owns them from then on.
5. **The handler** gets a `Packet`: its header, its data and the cerver, client and connection that it came from. The cerver deletes the packet after the handler returns. To answer, it creates a packet with **`packet_create ()`**, builds its bytes with **`packet_generate ()`**, sends it with **`packet_send_to ()`** and deletes it.
6. **`cerver_start ()`** opens the socket and blocks while the cerver runs. **`cerver_shutdown ()`**, that is safe to call from a signal handler, stops it and makes `cerver_start ()` return.
7. **`cerver_teardown ()`** drains the connections and the handlers that are working and releases the cerver. It is also the way to release a cerver that failed to start.

## 5. What the client does

``` c
#include <cerver/client.h>
#include <cerver/connection.h>

Client *client = client_create ();

Handler *handler = handler_create (echo_client_handler);
handler_set_direct_handle (handler, true);
client_set_app_handlers (client, handler, NULL);

Connection *connection = client_connection_create (
	client, "127.0.0.1", 7000, PROTOCOL_TCP, false
);

connection_set_max_sleep (connection, 30);

if (!client_connect_and_start (client, connection)) {
	// send packets with packet_send_to (packet, &sent, false, NULL, client, connection, NULL)
}

(void) client_connection_end (client, connection);
(void) client_teardown (client);
```

- A **`Client`** has one or more **`Connection`**s, to the same cerver or to different ones, and the same kind of handlers as a cerver.
- **`connection_set_max_sleep ()`** is how many seconds the connection keeps trying to connect, with a backoff between the attempts.
- **`client_connect_and_start ()`** connects and starts the thread that receives the packets of the connection; the client handlers run in it.
- **`client_connection_end ()`** closes the connection and **`client_teardown ()`** stops everything and releases the client and its handlers.

## Next

- [[Concepts]]: how these pieces fit together
- [[Lifecycle]]: every step of the life of a cerver
- [[Examples]]: the other examples of the repository
- [[Cerver]] and [[Clients]]: every function of both
