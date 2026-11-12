# Errors

`#include <cerver/errors.h>`, and the errors of `#include <cerver/client.h>`.

An error is a `PACKET_TYPE_ERROR` packet with an `ErrorMessage`. A cerver sends them to its clients, like for a failed authentication, a bad packet or a file that was not found, and the side that receives one runs the action registered for its error type. The actions are registered like the [[Events]].

## Concepts

### The error types

The cerver and the client use the same values, `CerverErrorType` and `ClientErrorType`:

| Type | What it means |
|---|---|
| `CERVER_ERROR` | An internal error of the cerver |
| `PACKET_ERROR` | The cerver could not handle a packet: a bad packet |
| `FAILED_AUTH` | The authentication failed (see [[Auth]]) |
| `GET_FILE`, `SEND_FILE`, `FILE_NOT_FOUND` | A file request failed (see [[Files]]) |
| `CREATE_LOBBY`, `JOIN_LOBBY`, `LEAVE_LOBBY`, `FIND_LOBBY`, `GAME_INIT`, `GAME_START`, `DESTROY_LOBBY` | A game request failed (see [[Game]]) |
| `UNKNOWN` | A received error of a type that is not known |

The client triggers the action of the type that it received. A cerver gets the error packets that its clients send to it, and reports its own failures to its clients with error packets and to the application with [[Events]].

### The message

An `ErrorMessage` is an object of the [[Serializer]] with the `error_type`, a `timestamp` in seconds since the epoch and a `msg` of up to `ERROR_MESSAGE_LENGTH - 1` bytes, cut without splitting a UTF-8 character. The action gets the message as a `String` in `error_message`, or `NULL` without one.

### Who owns the data

The action releases its data with `cerver_error_event_data_delete ()` or `client_error_data_delete ()`. The arguments of the action follow the same rules as the ones of an event: they belong to the registration, and with `drop_after_trigger` they go with the data of the only trigger.

## Usage

``` c
static void *on_failed_auth (void *error_data_ptr) {

	ClientErrorData *error_data = (ClientErrorData *) error_data_ptr;

	if (error_data->error_message) {
		cerver_log_error ("Failed to authenticate: %s", error_data->error_message->str);
	}

	client_error_data_delete (error_data);

	return NULL;

}

(void) client_error_register (
	client, CLIENT_ERROR_FAILED_AUTH,
	on_failed_auth, NULL, NULL,
	false, false
);
```

A cerver answers a request that it can not handle with an error packet:

``` c
(void) error_packet_generate_and_send (
	CERVER_ERROR_PACKET_ERROR, "Unknown request",
	packet->cerver, packet->client, packet->connection
);
```

## Reference

### The errors of a cerver

#### `cerver_error_event_register ()`

``` c
u8 cerver_error_event_register (
	Cerver *cerver,
	const CerverErrorType error_type,
	Work work, void *work_args, Action delete_action_args,
	bool create_thread, bool drop_after_trigger
);
```

Registers the action of an error that the clients send to the cerver. The action gets a `CerverErrorEventData *`.

- **Returns** 0 on success, 1 on error, like a type beyond `CERVER_MAX_ERRORS`.

#### `cerver_error_event_unregister ()`

``` c
u8 cerver_error_event_unregister (Cerver *cerver, const CerverErrorType error_type);
```

Unregisters the action of an error and releases its arguments.

- **Returns** 0 on success, 1 on error or if the error had no action.

#### `cerver_error_event_data_delete ()`

``` c
void cerver_error_event_data_delete (CerverErrorEventData *error_event_data);
```

Releases the data of an error, its message, and its arguments when the error was dropped after this trigger.

#### `cerver_error_event_delete ()`

``` c
void cerver_error_event_delete (void *event_ptr);
```

Deletes a registered `CerverErrorEvent` and its arguments with its `delete_action_args`. `cerver_error_event_unregister ()` and the teardown of the cerver call it.

#### `cerver_error_type_description ()`

``` c
const char *cerver_error_type_description (const CerverErrorType type);
```

The description of an error type.

#### `error_packet_generate ()`

``` c
Packet *error_packet_generate (const CerverErrorType type, const char *msg);
```

Creates a `PACKET_TYPE_ERROR` packet, ready to be sent, with the type, the current time and the message.

- **Returns** the packet, that the caller deletes, or `NULL` on error.

#### `error_packet_generate_and_send ()`

``` c
u8 error_packet_generate_and_send (
	const CerverErrorType type, const char *msg,
	Cerver *cerver, Client *client, Connection *connection
);
```

Creates an error packet and sends it to a connection.

- **Returns** 0 on success, 1 on error.

### The errors of a client

#### `client_error_register ()`

``` c
u8 client_error_register (
	Client *client,
	const ClientErrorType error_type,
	Work work, void *work_args, Action delete_action_args,
	bool create_thread, bool drop_after_trigger
);
```

Registers the action of an error that a cerver sends to the client. The action gets a `ClientErrorData *`.

- **Returns** 0 on success, 1 on error, like a type beyond `CLIENT_MAX_ERRORS`.

#### `client_error_unregister ()`

``` c
u8 client_error_unregister (Client *client, const ClientErrorType error_type);
```

Unregisters the action of an error and releases its arguments.

- **Returns** 0 on success, 1 on error or if the error had no action.

#### `client_error_data_delete ()`

``` c
void client_error_data_delete (ClientErrorData *error_data);
```

Releases the data of an error, its message, and its arguments when the error was dropped after this trigger.

#### `client_error_type_description ()`

``` c
const char *client_error_type_description (const ClientErrorType type);
```

The description of an error of a client.

#### `client_error_packet_generate ()`

``` c
Packet *client_error_packet_generate (const ClientErrorType type, const char *msg);
```

Creates an error packet, ready to be sent, from a client.

- **Returns** the packet, that the caller deletes, or `NULL` on error.

#### `client_error_packet_generate_and_send ()`

``` c
u8 client_error_packet_generate_and_send (
	const ClientErrorType type, const char *msg,
	Client *client, Connection *connection
);
```

Creates an error packet and sends it to the cerver of a connection.

- **Returns** 0 on success, 1 on error.

## Types

| Type | What it is |
|---|---|
| `CerverErrorType`, `ClientErrorType` | The error types above |
| `CerverErrorEvent`, `ClientError` | A registered action; set them only with the register functions |
| `CerverErrorEventData` | `cerver`, `client`, `connection`, `action_args`, `delete_action_args`, `owns_action_args`, `error_message` |
| `ClientErrorData` | `client`, `connection`, `action_args`, `delete_action_args`, `owns_action_args`, `error_message` |
| `ErrorMessage` | The data of an error packet: `timestamp`, `error_type`, `msg` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CERVER_MAX_ERRORS`, `CLIENT_MAX_ERRORS` | 32 | The errors that can have an action |
| `ERROR_MESSAGE_LENGTH` | 128 | The buffer of the message, with the NUL |
| `ERROR_MESSAGE_MAX_ENCODED_SIZE` | 160 | The biggest encoded `ErrorMessage` |
| `ERROR_MESSAGE_VERSION` | 1 | The version of its schema |
| `CERVER_ERROR_MAP (XX)`, `CLIENT_ERROR_MAP (XX)` | | The X-macros of the types, with their descriptions |

## See also

- [[Events]]: the events of a cerver and a client
- [[Packets]]: `PACKET_TYPE_ERROR`
- [[Protocol]]: the `ErrorMessage` on the wire
