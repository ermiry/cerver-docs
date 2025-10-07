# Auth

`#include <cerver/auth.h>`, the authentication setters of `#include <cerver/cerver.h>`, and the auth data of `#include <cerver/connection.h>`.

A cerver can require each new connection to authenticate before it becomes a client. Until then the connection is **on hold**: a dedicated thread receives it, and only its authentication packets are handled. The method that checks the credentials is the application's. With [[Sessions]], a client can then add more connections with the session token that it got.

## Concepts

### The flow

1. The cerver accepts a connection, puts it on hold (`CERVER_EVENT_ON_HOLD_CONNECTED`) and sends it its info, a `CerverSummary` with `auth_required`.
2. The client, seeing that the cerver requires authentication, answers with the auth data of its connection (`CLIENT_EVENT_AUTH_SENT`): its credentials (`AUTH_PACKET_TYPE_CLIENT_AUTH`, or `ADMIN_AUTH` for an [[Admin]]), or a session token (`CLIENT_TOKEN`, `ADMIN_TOKEN`). A connection without auth data does not answer, and stays on hold until its timeout.
3. The cerver calls the authenticate method of the application with an `AuthMethod`.
4. **Success:** the cerver creates a client with the connection, moves it out of hold to be received like any other, answers `AUTH_PACKET_TYPE_SUCCESS` (with the session token when it uses sessions), and triggers `CERVER_EVENT_CLIENT_SUCCESS_AUTH`. The client triggers `CLIENT_EVENT_SUCCESS_AUTH`.
5. **Failure:** the cerver sends a `CERVER_ERROR_FAILED_AUTH` error with the message of the method, triggers `CERVER_EVENT_CLIENT_FAILED_AUTH`, and takes one of the tries of the connection; without tries left, the connection is dropped (`CERVER_EVENT_ON_HOLD_DROPPED`).

An on hold connection is also dropped when it does not authenticate in `cerver_set_on_hold_timeout ()` seconds, or when it sends `cerver_set_on_hold_max_bad_packets ()` bad packets. One that its client closes triggers `CERVER_EVENT_ON_HOLD_DISCONNECTED`.

### The authenticate method

The method is a `delegate`, `u8 (*) (void *)`, that gets an `AuthMethod *` and returns **0 to accept** the connection and 1 to reject it:

| Field | What it is |
|---|---|
| `packet` | The authentication packet, with its cerver and connection |
| `auth_data->auth_data`, `auth_data->auth_data_size` | The credentials as the client sent them, or the decoded object of the auth schema (see below) |
| `auth_data->token` | The session token, when the client sent one (see [[Sessions]]) |
| `auth_data->data`, `auth_data->delete_data` | Data that the method can recover, like the user of the credentials. With a success that creates a client, it becomes the data of the client (`client->data`, released with the client); otherwise it is released with `delete_data` |
| `error_message` | A `String` that the method can set (`str_new ()`), sent to the client in the failed authentication error |

The method runs in the on hold thread, one authentication at a time.

### Credentials as bytes or as an object

By default the credentials are the bytes of `connection_set_auth_data ()`, and the method interprets them. With `cerver_set_auth_schema ()`, they are an object of a [[Serializer]] schema: the client sets them with `connection_set_auth_object ()`, cerver decodes and validates them before the method runs, and the method gets the object in `auth_data->auth_data`. A packet that is not an object of the schema fails the authentication without calling the method.

## Usage

The cerver:

``` c
#define MY_AUTH_TRIES		2
#define MY_ON_HOLD_SECONDS	8

typedef struct MyCredentials {

	char username[64];
	char password[64];

} MyCredentials;

static u8 my_authenticate (void *auth_method_ptr) {

	AuthMethod *auth_method = (AuthMethod *) auth_method_ptr;

	const MyCredentials *credentials = (const MyCredentials *) auth_method->auth_data->auth_data;
	if (!credentials || (auth_method->auth_data->auth_data_size != sizeof (MyCredentials))) {
		auth_method->error_message = str_new ("Missing credentials!");

		return 1;
	}

	if (my_check_credentials (credentials->username, credentials->password)) {
		auth_method->error_message = str_new ("Invalid credentials!");

		return 1;
	}

	return 0;

}

(void) cerver_set_auth (cerver, MY_AUTH_TRIES, my_authenticate);
cerver_set_on_hold_timeout (cerver, MY_ON_HOLD_SECONDS);
```

The client sets the credentials of its connection before it connects:

``` c
MyCredentials *credentials = (MyCredentials *) calloc (1, sizeof (MyCredentials));
(void) strncpy (credentials->username, "username", sizeof (credentials->username) - 1);
(void) strncpy (credentials->password, "password", sizeof (credentials->password) - 1);

connection_set_auth_data (connection, credentials, sizeof (MyCredentials), free, false);

(void) client_connect_and_start (client, connection);
```

## Reference

### The cerver

#### `cerver_set_auth ()`

``` c
u8 cerver_set_auth (Cerver *cerver, u8 max_auth_tries, delegate authenticate);
```

Makes the cerver require the authentication of each new connection, with `max_auth_tries` failed tries for each one (`CERVER_DEFAULT_MAX_AUTH_TRIES`, 2) and the authenticate method. A cerver without a method rejects every authentication. Call it before `cerver_start ()`.

- **Returns** 0 on success, 1 on error.

#### `cerver_set_auth_max_tries ()`

``` c
void cerver_set_auth_max_tries (Cerver *cerver, u8 max_auth_tries);
```

Sets the failed tries of each connection before it is dropped.

#### `cerver_set_auth_method ()`

``` c
void cerver_set_auth_method (Cerver *cerver, delegate authenticate);
```

Replaces the authenticate method; `NULL` is ignored.

#### `cerver_set_auth_schema ()`

``` c
void cerver_set_auth_schema (Cerver *cerver, const SerializerSchema *schema);
```

Sets the schema of the credentials: the method gets them decoded and validated. The fields of the schema can not take memory from an arena. `NULL` takes the credentials as bytes again.

#### `cerver_set_on_hold_timeout ()`

``` c
void cerver_set_on_hold_timeout (Cerver *cerver, u32 on_hold_timeout);
```

Sets the seconds that a new connection has to authenticate; then it is dropped (`CERVER_EVENT_ON_HOLD_DROPPED`). It is checked every on hold poll timeout. 0 disables it. The default is `CERVER_DEFAULT_ON_HOLD_SECONDS`, 16.

#### `cerver_set_on_hold_poll_timeout ()`

``` c
void cerver_set_on_hold_poll_timeout (Cerver *cerver, u32 on_hold_poll_timeout);
```

Sets the timeout in milliseconds of each `poll ()` of the on hold thread. The default is `CERVER_DEFAULT_ON_HOLD_TIMEOUT`, 2000.

#### `cerver_set_on_hold_poll_fds ()`

``` c
u8 cerver_set_on_hold_poll_fds (Cerver *cerver, const u32 max_connections);
```

Sets, before the cerver starts, how many connections can be on hold at the same time (`CERVER_DEFAULT_ON_HOLD_POLL_FDS`, 64). A new connection when it is full is closed.

- **Returns** 0 on success, 1 on error, like 0 or a cerver that already started.

#### `cerver_set_on_hold_max_bad_packets ()`

``` c
void cerver_set_on_hold_max_bad_packets (Cerver *cerver, u8 on_hold_max_bad_packets);
```

Sets how many bad packets an on hold connection can send before it is dropped. A bad packet is one that the on hold thread can not handle, or that fails `packet_check ()` when the on hold packets are checked. The default is `CERVER_DEFAULT_ON_HOLD_MAX_BAD_PACKETS`, 4.

#### `cerver_set_on_hold_check_packets ()`

``` c
void cerver_set_on_hold_check_packets (Cerver *cerver, bool check);
```

With `true`, the packets of the on hold connections are checked with `packet_check ()`. The default is `false`.

#### `cerver_set_on_hold_receive_buffer_size ()`

``` c
void cerver_set_on_hold_receive_buffer_size (Cerver *cerver, const size_t on_hold_receive_buffer_size);
```

Sets the bytes of each receive of an on hold connection. The default is `CERVER_DEFAULT_ON_HOLD_RECEIVE_BUFFER_SIZE`, 4096; 0 is ignored.

#### `cerver_auth_error_to_string ()`, `cerver_auth_error_description ()`

``` c
const char *cerver_auth_error_to_string (const CerverAuthError error);
const char *cerver_auth_error_description (const CerverAuthError error);
```

The name and the description of a `CerverAuthError`, the result of an authentication: `NONE`, `SUCCESS`, `NO_HANDLER`, `MISSING_VALUES`, `FAILED`, `INVALID_SESSION` or `DROPPED`.

### The connection of a client

#### `connection_set_auth_data ()`

``` c
void connection_set_auth_data (
	Connection *connection,
	void *auth_data, size_t auth_data_size, Action delete_auth_data,
	bool admin_auth
);
```

Sets the credentials that the connection sends when its cerver requires authentication, as bytes, and how to release them when the connection is deleted (`NULL` does not release them). With `admin_auth`, the connection authenticates as an admin. Set them before the connection connects.

#### `connection_set_auth_object ()`

``` c
u8 connection_set_auth_object (
	Connection *connection,
	const SerializerSchema *schema, const void *object,
	bool admin_auth
);
```

Sets the credentials as an object of the auth schema of the cerver. The object is encoded now, so it can be released after the call.

- **Returns** 0 on success, 1 if the object can not be encoded.

#### `connection_set_auth_token ()`

``` c
u8 connection_set_auth_token (Connection *connection, const char *token, bool admin_auth);
```

Sets the session token that a cerver with sessions gave to the client, to authenticate the connection with it instead of the credentials (see [[Sessions]]).

- **Returns** 0 on success, 1 on error.

#### `connection_remove_auth_data ()`

``` c
void connection_remove_auth_data (Connection *connection);
```

Releases the auth data of the connection with its delete method, if it has one, and leaves the connection without auth data.

#### `connection_generate_auth_packet ()`

``` c
u8 connection_generate_auth_packet (Connection *connection);
```

Builds the authentication packet of the connection from its auth data. The connection does it by itself when its cerver asks for authentication.

- **Returns** 0 on success, 1 on error.

## Types

| Type | What it is |
|---|---|
| `AuthMethod` | What the authenticate method gets: `packet`, `auth_data`, `error_message` |
| `AuthData` | `token`, `auth_data`, `auth_data_size`, `data`, `delete_data` |
| `CerverAuthError` | The result of an authentication, see above |
| `AuthPacketType` | `AUTH_PACKET_TYPE_REQUEST_AUTH`, `CLIENT_AUTH`, `ADMIN_AUTH`, `SUCCESS`, `CLIENT_TOKEN`, `ADMIN_TOKEN` (see [[Protocol]]) |
| `delegate` | `u8 (*) (void *)` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `CERVER_DEFAULT_MAX_AUTH_TRIES` | 2 | Failed tries of a connection |
| `CERVER_DEFAULT_ON_HOLD_SECONDS` | 16 | Seconds to authenticate |
| `CERVER_DEFAULT_ON_HOLD_TIMEOUT` | 2000 | Milliseconds of each on hold `poll ()` |
| `CERVER_DEFAULT_ON_HOLD_POLL_FDS` | 64 | Connections on hold at the same time |
| `CERVER_DEFAULT_ON_HOLD_MAX_BAD_PACKETS` | 4 | Bad packets of an on hold connection |
| `CERVER_DEFAULT_ON_HOLD_RECEIVE_BUFFER_SIZE` | 4096 | Bytes of each on hold receive |
| `CONNECTION_DEFAULT_MAX_AUTH_TRIES` | 2 | The tries that a connection counts |
| `CERVER_AUTH_ERROR_MAP (XX)` | | The X-macro of `CerverAuthError` |

## See also

- [[Sessions]]: several connections of the same client with a session token
- [[Admin]]: the authentication of the admins
- [[UDP]]: the authentication of a UDP peer, with a token in each datagram
- [[Events]]: the events of the on hold connections and the authentication
