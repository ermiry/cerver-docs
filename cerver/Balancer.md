# Balancer

`#include <cerver/balancer.h>`

A load balancer is a cerver that receives the packets of its clients and routes each one to one of its **services**, other cervers that it is connected to, and routes the responses of the services back to the client that sent the request. The clients and the services speak the normal cerver [[Protocol]]; the balancer only reads the header of each packet to choose its service and to remember the way back.

## Concepts

### The route of a response

Each client connection gets a **route id** when it sends its first packet: a slot of 20 bits and a generation of 12 bits. The balancer writes it in the `route_id` of the header of every packet that it routes to a service, and **the service writes the same `route_id` in its response**, so the balancer knows which connection gets it. When a client connection ends, the generation of its slot changes: a late response of a service for that connection is discarded (`discarded_responses`), even if the slot is now used by another connection.

A service that answers does it like this:

``` c
Packet *response = packet_create (PACKET_TYPE_APP, packet->header.request_type, data, size);
response->header.route_id = packet->header.route_id;

if (!packet_generate (response)) {
	size_t sent = 0;
	(void) packet_send_to (response, &sent, false, packet->cerver, packet->client, packet->connection, NULL);
}

packet_delete (response);
```

### Choosing the service

| Type | Service of each packet |
|---|---|
| `BALANCER_TYPE_ROUND_ROBIN` | The next service that is working, in turns |
| `BALANCER_TYPE_HANDLER_ID` | The service with the index of the `handler_id` of the header |
| `BALANCER_TYPE_CUSTOM` | The index that `balancer_set_select_service ()` returns from the header and the body |

When the selected service is not working, or no service is, the packet goes to the method of `balancer_set_on_unavailable_services ()`, that gets the packet, and is counted in `backup_handled_packets`; without one, it is counted in `unhandled_packets`.

### The services

A service is registered with its address before the balancer starts, up to the `n_services` of `balancer_create ()`. `balancer_start ()` connects to every service: it starts if at least one of them is working, and a dedicated thread keeps trying the ones that were down. A connection to a service that ends reconnects by itself, waiting the reconnect time of the service between attempts. Each change of the status of a service runs its `on_status_change`.

### Threads

The cerver of a balancer is a normal `Cerver` (`balancer->cerver`): set its handler type and its options before `balancer_start ()`, and stop it with `cerver_shutdown (balancer->cerver)`. It receives each client in its own thread (`CERVER_HANDLER_TYPE_THREADS`). The responses of each service are received in the thread of its connection. The packets are routed with a complete `send` under the write mutex of the socket, so the packets of different clients never mix in the connection of a service.

## Usage

``` c
#define MY_BALANCER_PORT		7000
#define MY_SERVICES				2

static Balancer *balancer = NULL;

static void end (int signal_number) {

	(void) signal_number;

	if (balancer) (void) cerver_shutdown (balancer->cerver);

}

int main (void) {

	cerver_init ();

	(void) signal (SIGINT, end);

	balancer = balancer_create (
		"balancer", BALANCER_TYPE_ROUND_ROBIN,
		MY_BALANCER_PORT, CERVER_DEFAULT_CONNECTION_QUEUE, MY_SERVICES
	);

	if (balancer) {
		cerver_set_handler_type (balancer->cerver, CERVER_HANDLER_TYPE_THREADS);
		cerver_set_handle_detachable_threads (balancer->cerver, true);

		(void) balancer_service_register (balancer, "127.0.0.1", 7001);
		(void) balancer_service_register (balancer, "127.0.0.1", 7002);

		if (balancer_start (balancer)) {
			cerver_log_error ("Failed to start the balancer!");
		}

		balancer_stats_print (balancer);
		(void) balancer_teardown (balancer);
	}

	cerver_end ();

	return 0;

}
```

[`examples/balancer.c`](https://github.com/ermiry/cerver/blob/master/examples/balancer.c) and [`examples/service.c`](https://github.com/ermiry/cerver/blob/master/examples/service.c) are a complete balancer and service.

## Reference

### The balancer

#### `balancer_create ()`

``` c
Balancer *balancer_create (
	const char *name, const BalancerType type,
	const u16 port, const u16 connection_queue,
	const unsigned int n_services
);
```

Creates a balancer of a type, with its cerver in the port, and room for `n_services` services.

- **Returns** the new balancer, or `NULL` on error.

#### `balancer_start ()`

``` c
u8 balancer_start (Balancer *balancer);
```

Connects to the services and starts the cerver of the balancer; it blocks while the cerver runs, like `cerver_start ()`.

- **Returns** 0 after the balancer was stopped, 1 if it failed to start, like without any working service.

#### `balancer_teardown ()`

``` c
u8 balancer_teardown (Balancer *balancer);
```

Ends the cerver of the balancer, disconnects from the services and releases the balancer. It also releases a balancer that failed to start.

- **Returns** 0 on success, 1 on error.

#### `balancer_set_select_service ()`

``` c
void balancer_set_select_service (
	Balancer *balancer, const BalancerSelectService select_service, void *args
);
```

Sets, before the balancer starts, the method of a `BALANCER_TYPE_CUSTOM` balancer. It runs in the thread of the client connection, without the lock of the balancer, with the header, the body and `args`, and returns the index of the service; a negative index, or one without a working service, sends the packet to the method of unavailable services.

#### `balancer_set_on_unavailable_services ()`

``` c
void balancer_set_on_unavailable_services (Balancer *balancer, const Action on_unavailable_services);
```

Sets the method that gets each packet that can not be routed, like to answer it from the balancer itself.

#### `balancer_get_type ()`, `balancer_type_to_string ()`

``` c
BalancerType balancer_get_type (const Balancer *balancer);
const char *balancer_type_to_string (const BalancerType type);
```

The type of a balancer, and its name, like `"Round-Robin"`.

#### `balancer_stats_print ()`

``` c
void balancer_stats_print (const Balancer *balancer);
```

Prints the stats of the balancer and of each of its services.

### Services

#### `balancer_service_register ()`

``` c
unsigned int balancer_service_register (Balancer *balancer, const char *ip_address, const u16 port);
```

Registers a service, connected when the balancer starts. Its index is the order of registration, from 0.

- **Returns** 0 on success, 1 on error, like more services than `n_services`.

#### `balancer_service_set_name ()`, `balancer_service_get_name ()`

``` c
void balancer_service_set_name (Service *service, const char *name);
const char *balancer_service_get_name (const Service *service);
```

The name of a service, up to `SERVICE_NAME_SIZE - 1` characters, used in the logs.

#### `balancer_service_set_reconnect_wait_time ()`

``` c
void balancer_service_set_reconnect_wait_time (Service *service, const unsigned int wait_time);
```

Sets the seconds to wait before each reconnection to the service. The default is `SERVICE_DEFAULT_WAIT_TIME`, 20.

#### `balancer_service_set_on_status_change ()`

``` c
void balancer_service_set_on_status_change (Service *service, const Action on_status_change);
```

Sets the method that runs with the service every time that its status changes.

#### `balancer_service_get_status ()`, `balancer_service_get_status_string ()`, `balancer_service_status_to_string ()`, `balancer_service_status_description ()`

``` c
ServiceStatus balancer_service_get_status (const Service *service);
const char *balancer_service_get_status_string (const Service *service);
const char *balancer_service_status_to_string (const ServiceStatus status);
const char *balancer_service_status_description (const ServiceStatus status);
```

The status of a service, read atomically, and its name and description: `CONNECTING`, `READY`, `WORKING`, `DISCONNECTING`, `DISCONNECTED`, `UNAVAILABLE` or `ENDED`.

#### `balancer_service_stats_print ()`

``` c
void balancer_service_stats_print (const Service *service);
```

Prints the packets and bytes routed to the service and received from it.

### Route ids

#### `balancer_route_id ()`, `balancer_route_id_slot ()`, `balancer_route_id_generation ()`, `balancer_route_next_generation ()`

``` c
u32 balancer_route_id (const u32 slot, const u16 generation);
u32 balancer_route_id_slot (const u32 route_id);
u16 balancer_route_id_generation (const u32 route_id);
u16 balancer_route_next_generation (const u16 generation);
```

Build a route id from a slot and a generation, take them apart, and give the generation that follows another one, never 0. A route id is never 0, the id of a packet without a route.

## Types

| Type | What it is |
|---|---|
| `Balancer` | A balancer: its `cerver`, its `services`, its `stats` |
| `BalancerType` | `BALANCER_TYPE_NONE`, `ROUND_ROBIN`, `HANDLER_ID`, `CUSTOM` |
| `BalancerSelectService` | `int (*) (const PacketHeader *header, const char *body, const size_t body_size, void *args)` |
| `BalancerStats` | The packets and bytes received from the clients, routed to the services and sent back, the bad ones, the discarded responses, the ones handled when no service was available, and the ones not handled, by packet type |
| `Service` | A service: `id`, `name`, `status`, its `connection`, its reconnect time, its `stats` |
| `ServiceStatus` | The status of a service, see above |
| `ServiceStats` | The packets and bytes routed to a service and received from it |
| `BalancerRoute` | A slot of the routes: the `socket` of its connection and its `generation` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `BALANCER_ROUTE_SLOT_BITS`, `BALANCER_ROUTE_GENERATION_BITS` | 20, 12 | The bits of a route id |
| `BALANCER_ROUTES_MAX` | 2^20 | The client connections that are routed at the same time |
| `BALANCER_DEFAULT_ROUTES_SIZE`, `BALANCER_ROUTES_GROWTH_FACTOR` | 64, 2 | The table of routes and how it grows |
| `BALANCER_RECEIVE_MAX_TIMEOUTS` | 4 | The receive timeouts in the middle of a packet before the connection ends |
| `SERVICE_DEFAULT_WAIT_TIME` | 20 | Seconds between the reconnections to a service |
| `SERVICE_NAME_SIZE` | 64 | The buffer of the name of a service |

## See also

- [[Packets]]: the `route_id` and the `handler_id` of the header
- [[Connections]]: the reconnection of the connections to the services
- [[Examples]]: the balancer and its services
