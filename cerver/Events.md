# Events

`#include <cerver/events.h>`, and the events of `#include <cerver/client.h>`.

An event runs a function of the application when something happens in a cerver or a client: a cerver that starts, a client that connects, a lobby that is created. The errors that a cerver and a client send to each other are in [[Errors]], and they are registered the same way.

## Concepts

### Registering an action

Every event is registered with the same arguments, one action per event:

| Argument | What it is |
|---|---|
| `work` | A `Work`, `void *(*) (void *)`, called with a newly allocated `CerverEventData` or `ClientEventData` |
| `work_args` | The arguments of the action, passed in the `action_args` of each data |
| `delete_action_args` | How to release `work_args` when the action is unregistered, or with the data of a dropped trigger |
| `create_thread` | `true` runs the action in a new thread; the teardown of the cerver or the client waits for it. `false` runs it in the thread where the event happened, so it has to be short |
| `drop_after_trigger` | `true` unregisters the action after its first trigger |

Registering an event again replaces its action, and releases the arguments of the previous one.

### Who owns the data

- **The action releases its data** with `cerver_event_data_delete ()` or `client_event_data_delete ()`, in every case: without it, the data leaks.
- **The arguments belong to the registration**: they are released when the action is unregistered, when it is replaced, and in the teardown of the cerver or the client. With `drop_after_trigger`, they go with the data of the only trigger, and its `*_data_delete ()` releases them, so an action in its own thread can use them until it ends.
- **The pointers of the data** (`cerver`, `client`, `connection`) are only valid during the event. A lobby event keeps a reference to its `lobby` until its data is deleted (see [[Game]]).
- **The response of a client event** (`response_data`), like a `LobbySummary`, is released with the data when `owns_response_data` is set.

### The events of a cerver

| Event | When |
|---|---|
| `CERVER_EVENT_STARTED` | `cerver_start ()` is accepting connections |
| `CERVER_EVENT_TEARDOWN` | `cerver_teardown ()` starts |
| `CERVER_EVENT_ON_HOLD_CONNECTED` | A new connection was put on hold to authenticate (see [[Auth]]) |
| `CERVER_EVENT_ON_HOLD_DISCONNECTED` | An on hold connection was closed or lost by its client |
| `CERVER_EVENT_ON_HOLD_DROPPED` | An on hold connection was dropped: its tries, its bad packets or its timeout |
| `CERVER_EVENT_CLIENT_SUCCESS_AUTH`, `_CLIENT_FAILED_AUTH` | A connection authenticated, or failed to |
| `CERVER_EVENT_CLIENT_CONNECTED` | A new client was registered |
| `CERVER_EVENT_CLIENT_NEW_CONNECTION` | A connection was added to an existing client (see [[Sessions]]) |
| `CERVER_EVENT_CLIENT_CLOSE_CONNECTION` | A connection of a client was closed |
| `CERVER_EVENT_CLIENT_DISCONNECTED`, `_CLIENT_DROPPED` | A client left, or was dropped (inactive, bad packets) |
| `CERVER_EVENT_ADMIN_FAILED_AUTH`, `_ADMIN_CONNECTED`, `_ADMIN_NEW_CONNECTION`, `_ADMIN_CLOSE_CONNECTION`, `_ADMIN_DISCONNECTED`, `_ADMIN_DROPPED` | The same, for the admins (see [[Admin]]) |
| `CERVER_EVENT_LOBBY_CREATE`, `_LOBBY_JOIN`, `_LOBBY_LEAVE`, `_LOBBY_START`, `_LOBBY_DESTROY` | The requests of a game cerver (see [[Game]]) |

### The events of a client

| Event | When |
|---|---|
| `CLIENT_EVENT_CONNECTED` | A connection connected to its cerver, also after a reconnection |
| `CLIENT_EVENT_DISCONNECTED` | The cerver disconnected the client |
| `CLIENT_EVENT_CONNECTION_FAILED` | A connection failed to connect |
| `CLIENT_EVENT_CONNECTION_CLOSE` | The connection was closed: by the client, by the cerver, or lost (a receive returned 0 or less) |
| `CLIENT_EVENT_CONNECTION_DATA` | The response of `client_request_to_cerver_async ()` was handled (see [[Clients]]) |
| `CLIENT_EVENT_CERVER_INFO` | The info of the cerver arrived; it is in the `cerver_report` of the `connection` |
| `CLIENT_EVENT_CERVER_TEARDOWN` | The cerver is going to teardown; the connection is closed (or reconnects) right after |
| `CLIENT_EVENT_AUTH_SENT`, `_SUCCESS_AUTH` | The authentication of the connection was sent, and it succeeded (see [[Auth]]) |
| `CLIENT_EVENT_LOBBY_CREATE`, `_LOBBY_JOIN`, `_LOBBY_LEAVE`, `_LOBBY_START`, `_LOBBY_UPDATE`, `_LOBBY_DESTROY`, `_GAME_INIT` | The answers of a game cerver, with a `LobbySummary` in `response_data` (see [[Game]]) |
| `CLIENT_EVENT_CERVER_STATS`, `_CERVER_GAME_STATS`, `_MAX_AUTH_TRIES` | Reserved: the 1.x protocol has no packet that triggers them |

## Usage

``` c
static void *on_client_connected (void *event_data_ptr) {

	CerverEventData *event_data = (CerverEventData *) event_data_ptr;

	cerver_log_msg (
		"A new client connected to %s",
		event_data->cerver->info->name
	);

	cerver_event_data_delete (event_data);

	return NULL;

}

static void *on_cerver_info (void *event_data_ptr) {

	ClientEventData *event_data = (ClientEventData *) event_data_ptr;

	const CerverReport *report = event_data->connection->cerver_report;
	cerver_log_msg ("Connected to %s", report->name->str);

	client_event_data_delete (event_data);

	return NULL;

}

(void) cerver_event_register (
	cerver, CERVER_EVENT_CLIENT_CONNECTED,
	on_client_connected, NULL, NULL,
	false, false
);

(void) client_event_register (
	client, CLIENT_EVENT_CERVER_INFO,
	on_cerver_info, NULL, NULL,
	false, false
);
```

## Reference

### The events of a cerver

#### `cerver_event_register ()`

``` c
u8 cerver_event_register (
	Cerver *cerver,
	const CerverEventType event_type,
	Work work, void *work_args, Action delete_action_args,
	bool create_thread, bool drop_after_trigger
);
```

Registers the action of an event of the cerver, replacing the previous one. The action gets a `CerverEventData *`.

- **Returns** 0 on success, 1 on error, like a type beyond `CERVER_MAX_EVENTS`.

#### `cerver_event_unregister ()`

``` c
u8 cerver_event_unregister (Cerver *cerver, const CerverEventType event_type);
```

Unregisters the action of an event and releases its arguments with its `delete_action_args`.

- **Returns** 0 on success, 1 on error or if the event had no action.

#### `cerver_event_data_delete ()`

``` c
void cerver_event_data_delete (CerverEventData *event_data);
```

Releases the data of an event: the reference to its lobby, and its arguments when the event was dropped after this trigger.

#### `cerver_event_delete ()`

``` c
void cerver_event_delete (void *event_ptr);
```

Deletes a registered `CerverEvent` and its arguments with its `delete_action_args`. `cerver_event_unregister ()` and the teardown of the cerver call it.

#### `cerver_event_type_description ()`

``` c
const char *cerver_event_type_description (const CerverEventType type);
```

The description of an event, like `"A new client has connected to the cerver"`.

### The events of a client

#### `client_event_register ()`

``` c
u8 client_event_register (
	Client *client,
	const ClientEventType event_type,
	Work work, void *work_args, Action delete_action_args,
	bool create_thread, bool drop_after_trigger
);
```

Registers the action of an event of the client. The action gets a `ClientEventData *`, with the `client`, the `connection` and, for some events, a `response_data`.

- **Returns** 0 on success, 1 on error, like a type beyond `CLIENT_MAX_EVENTS`.

#### `client_event_unregister ()`

``` c
u8 client_event_unregister (Client *client, const ClientEventType event_type);
```

Unregisters the action of an event and releases its arguments.

- **Returns** 0 on success, 1 on error or if the event had no action.

#### `client_event_data_delete ()`

``` c
void client_event_data_delete (ClientEventData *event_data);
```

Releases the data of an event, its response when it owns it (`owns_response_data`), and its arguments when the event was dropped after this trigger.

#### `client_event_type_description ()`

``` c
const char *client_event_type_description (const ClientEventType type);
```

The description of an event of a client.

## Types

| Type | What it is |
|---|---|
| `CerverEventType`, `ClientEventType` | The events above |
| `CerverEvent`, `ClientEvent` | A registered action; set them only with the register functions |
| `CerverEventData` | `cerver`, `client`, `connection`, `lobby`, `action_args`, `delete_action_args`, `owns_action_args` |
| `ClientEventData` | `client`, `connection`, `response_data`, `delete_response_data`, `owns_response_data`, `action_args`, `delete_action_args`, `owns_action_args` |
| `Work` | `void *(*) (void *)` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CERVER_MAX_EVENTS`, `CLIENT_MAX_EVENTS` | 32 | The events that can have an action |
| `CERVER_EVENT_MAP (XX)`, `CLIENT_EVENT_MAP (XX)` | | The X-macros of the events, with their descriptions |

## See also

- [[Errors]]: the error packets and their actions
- [[Lifecycle]]: when the events of a cerver and a client happen
- [[Clients]]: the connections that trigger the events of a client
