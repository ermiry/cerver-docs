# Lifecycle

The order of the calls in the life of a cerver and of a client, and what cerver does in each step.

## A cerver

```
cerver_init ()
  └─ cerver_create ()
       └─ cerver_set_* (), handlers, events          configuration, from a single thread
            └─ cerver_start ()                         blocks while the cerver runs
                 ◀─ cerver_shutdown ()                 from a signal handler or another thread
            cerver_start () returns
       └─ cerver_teardown ()                           drains and releases the cerver
cerver_end ()
```

### `cerver_init ()` and `cerver_end ()`

`cerver_init ()` prepares the global values of cerver, the log, and runs once at the start of the program, before any other call. `cerver_end ()` releases them once at the end, after every cerver and client was torn down.

### Create and configure

`cerver_create (type, name, port, protocol, use_ipv6, connection_queue)` returns a cerver that has not opened anything yet. Everything else is configured with its setters before `cerver_start ()`: the handler type, the handlers, the events, the authentication, the timeouts and the limits (see [[Cerver]] and [[Configuration]]). The setters are not synchronized, so they run in a single thread.

**A TCP cerver needs a handler type:** the default is `CERVER_HANDLER_TYPE_NONE`, and with it `cerver_start ()` fails with «handler type has NOT been configured». Set `CERVER_HANDLER_TYPE_POLL` or `CERVER_HANDLER_TYPE_THREADS` with `cerver_set_handler_type ()`.

### Start

`cerver_start ()`:

1. ignores `SIGPIPE` when the process has not set a handler for it (`cerver_set_ignore_sigpipe ()`), so writing to a connection that the client reset does not kill the process;
2. creates the socket, binds the bind address (every address of the family by default) and sets its options;
3. starts the threads of the handlers with a queue, the authentication (the on hold poll), the admins, the updates and the check of inactive clients;
4. creates the thpool, only with `CERVER_HANDLER_TYPE_THREADS`;
5. listens, triggers `CERVER_EVENT_STARTED` and accepts connections until the cerver is stopped.

It returns 0 after the cerver was stopped, and 1 if it failed to start: a port in use, an address that is not valid, a handler that did not start. In both cases the cerver is released with `cerver_teardown ()`. A UDP cerver receives datagrams instead of accepting connections (see [[UDP]]).

### Stop

`cerver_shutdown ()` stops accepting connections and wakes up `cerver_start ()`. It is safe to call it from a signal handler, from another thread and more than once, and a stop before `cerver_start ()` cancels the start. The usual way:

``` c
static Cerver *my_cerver = NULL;

static void end (int signal_number) {

	(void) signal_number;

	if (my_cerver) (void) cerver_shutdown (my_cerver);

}

int main (void) {

	(void) signal (SIGINT, end);
	(void) signal (SIGTERM, end);

	cerver_init ();

	my_cerver = cerver_create (
		CERVER_TYPE_CUSTOM, "my-cerver",
		CERVER_DEFAULT_PORT, PROTOCOL_TCP, false, CERVER_DEFAULT_CONNECTION_QUEUE
	);

	if (my_cerver) {
		cerver_set_handler_type (my_cerver, CERVER_HANDLER_TYPE_POLL);

		(void) cerver_start (my_cerver);

		(void) cerver_teardown (my_cerver);
	}

	cerver_end ();

	return 0;

}
```

The signal handler only stops the cerver: it is released after `cerver_start ()` returns, outside of the signal handler.

### Teardown

`cerver_teardown ()` releases the cerver in this order:

1. triggers `CERVER_EVENT_TEARDOWN`;
2. stops accepting connections, if it was not stopped yet;
3. stops the lobbies of a game cerver and joins the update threads and the check of inactive clients;
4. **drains** the work in progress (see below);
5. if the drain finished, stops and joins the handlers, destroys the thpool and releases the data of the cerver type;
6. closes the socket and ends the admins;
7. waits for the events that run in their own threads, and releases the cerver.

### Drain

The drain waits for the connections in progress, with the **drain policy** (`cerver_set_drain_policy ()`) and the **drain timeout** (`cerver_set_drain_timeout ()`, 32 seconds by default, 0 waits forever):

| Policy | What happens with the connections that were queued in the thpool |
|---|---|
| `CERVER_DRAIN_POLICY_RUN` (the default) | They are handled until the drain timeout |
| `CERVER_DRAIN_POLICY_REJECT` | They are rejected right away |

A cerver with threads shuts its clients down and waits for the thpool and for the detachable threads of its connections. If they did not finish, it waits a grace of `CERVER_DRAIN_GRACE_TIMEOUT` (4 seconds) more, and then it logs what is still alive. **When the drain does not finish, the teardown does not release the memory of the cerver**, because those connections still use it: the process should exit after it.

## A client

```
client_create ()
  └─ client_set_* (), handlers, events
       └─ client_connection_create ()                 one or more connections
            └─ client_connect_and_start ()            connects and starts receiving
                 send packets, handle the responses
            └─ client_connection_end ()               closes a connection
  └─ client_teardown ()                                stops everything and releases the client
```

- **`client_connect_and_start ()`** blocks until the connection is connected or failed, and starts the update thread that receives its packets. `client_connect_and_start_async ()` does it in another thread.
- **A connection reconnects** when it is configured with it (see [[Connections]]): the update thread waits, connects with a new socket and keeps receiving, and the client gets `CLIENT_EVENT_CONNECTED` again.
- **`client_teardown ()`** ends every connection, waits for the update threads, the async connections and the events in their own threads, stops the handlers and releases the client. It can not run in the update thread of one of its connections: use `client_teardown_async ()` from a handler.

## See also

- [[Concepts]]: the pieces and their rules
- [[Cerver]] and [[Clients]]: every function of both
- [[Events]]: the events of each step
