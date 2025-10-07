# Sessions

`#include <cerver/sessions.h>`, `cerver_set_sessions ()` of `#include <cerver/cerver.h>`, and the session id of `#include <cerver/client.h>`.

A session lets a client have several connections with the same cerver, like the same user on different devices or a client that opens a second connection for its files. The first connection authenticates with its credentials and gets a **session token**; the next ones authenticate with the token and join the same `Client` on the cerver. Sessions need [[Auth]].

## Concepts

### The flow

1. A cerver with `cerver_set_auth ()` and `cerver_set_sessions ()` authenticates a first connection with its credentials, creates a client, and generates its **session id** with the session id generator.
2. The `AUTH_PACKET_TYPE_SUCCESS` packet carries the session id as a `SessionToken`, and the client keeps it (`client_get_session_id ()`).
3. A new connection sets the token with `connection_set_auth_token ()`, so it authenticates with it (`AUTH_PACKET_TYPE_CLIENT_TOKEN`) instead of the credentials.
4. The cerver finds the client of the token (`client_get_by_session_id ()`), adds the connection to it and triggers `CERVER_EVENT_CLIENT_NEW_CONNECTION`. An unknown token is a failed authentication (`CERVER_AUTH_ERROR_INVALID_SESSION`).

A client of the cerver stays while it has connections: `CERVER_EVENT_CLIENT_CLOSE_CONNECTION` for each one that closes, and `CERVER_EVENT_CLIENT_DISCONNECTED` when the last one does.

### The session id generator

The generator gets a `SessionData *` with the authentication `packet`, its `auth_data` and the new `client`, and returns a string allocated with `malloc ()`, up to `TOKEN_SIZE - 1` (255) characters, or `NULL` on error. Cerver copies it and releases it with `free ()`.

The default generator returns `SESSION_ID_LENGTH` (64) hex characters of `SESSION_ID_RANDOM_BYTES` (32) random bytes of the kernel, so a session id can not be guessed. A custom generator must be just as unpredictable: whoever knows a session id can add connections to its client.

## Usage

The cerver, with the default generator:

``` c
(void) cerver_set_auth (cerver, MY_AUTH_TRIES, my_authenticate);
(void) cerver_set_sessions (cerver, NULL);
```

A second connection of the client, after the first one authenticated:

``` c
char session_id[TOKEN_SIZE] = { 0 };
if (client_get_session_id (client, session_id, sizeof (session_id))) {
	Connection *second = client_connection_create (client, "127.0.0.1", 7000, PROTOCOL_TCP, false);

	(void) connection_set_auth_token (second, session_id, false);
	(void) client_connect_and_start (client, second);
}
```

## Reference

#### `cerver_set_sessions ()`

``` c
u8 cerver_set_sessions (Cerver *cerver, void *(*session_id_generator) (const void *));
```

Makes the cerver use sessions, with a session id generator, or the default one with `NULL`. The cerver must also require authentication. Call it before `cerver_start ()`.

- **Returns** 0 on success, 1 for a `NULL` cerver.

#### `client_get_session_id ()`

``` c
size_t client_get_session_id (const Client *client, char *session_id, const size_t size);
```

Copies the session id of a client into `session_id`, cut to `size - 1` bytes and terminated with a NUL. It is safe to call it while the connection receives.

- **Returns** the bytes copied, 0 if the client does not have a session id.

#### `client_set_session_id ()`

``` c
u8 client_set_session_id (Client *client, const char *session_id);
```

Sets the session id of a client. Cerver sets it on both sides; an application only calls it to give a client a token that it got in another way.

- **Returns** 0 on success, 1 on error.

#### `client_get_by_session_id ()`

``` c
Client *client_get_by_session_id (Cerver *cerver, const char *session_id);
```

Returns the client of a cerver with a session id, or `NULL`. The cerver must use sessions.

## Types

| Type | What it is |
|---|---|
| `SessionData` | What a session id generator gets: `packet`, `auth_data`, `client` |
| `SessionToken` | The object of the [[Serializer]] that carries a session id: `token` |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `TOKEN_SIZE` | 256 | The biggest session id, with the NUL |
| `SESSION_ID_RANDOM_BYTES` | 32 | The random bytes of a default session id |
| `SESSION_ID_LENGTH` | 64 | The hex characters of a default session id |
| `SESSION_TOKEN_MAX_ENCODED_SIZE` | 272 | The biggest encoded `SessionToken` |
| `SESSION_TOKEN_VERSION` | 1 | The version of its schema |

## See also

- [[Auth]]: the authentication that sessions are built on
- [[Clients]]: the connections of a client
- [[Events]]: `CERVER_EVENT_CLIENT_NEW_CONNECTION`
