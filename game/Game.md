# Game

`#include <cerver/game/game.h>`

A game cerver (`CERVER_TYPE_GAME`) has **lobbies**: groups of players that play a game of a **game type**, with an owner, their own threads and the packets of their players. The clients create, join, start and leave the lobbies with `PACKET_TYPE_GAME` requests, and get the state of each lobby back as a `LobbySummary`. The pieces are in [[Lobby]], [[Players]], [[Gametypes]] and [[Scores]].

## Concepts

### The game cerver

`cerver_create (CERVER_TYPE_GAME, ...)` creates the `GameCerver` of the cerver, in `cerver->cerver_data`, with its lobbies, its game types, its stats and a **recursive lock** for the lobbies, their players, their owners and their states. The callbacks of the game types run with the lock taken and can use the API of the lobbies.

### The requests

A client sends each request with `game_request_send ()`: a `PACKET_TYPE_GAME` packet of a `GamePacketType` with its object. The cerver answers with a `LobbySummary`, or with an error of the request.

| Request | Object | What the cerver does | Error |
|---|---|---|---|
| `GAME_PACKET_TYPE_LOBBY_CREATE` | `LobbyCreate`: `game_type` | Creates a lobby of the game type, with the client as its owner | `CREATE_LOBBY` |
| `GAME_PACKET_TYPE_LOBBY_JOIN` | `LobbyJoin`: `lobby_id`, `game_type` | Joins the lobby; without an id, a lobby of the game type with a place, or a new one when none has | `JOIN_LOBBY`, `FIND_LOBBY` |
| `GAME_PACKET_TYPE_LOBBY_LEAVE` | `LobbyRequest`: `lobby_id` | The player leaves; the owner passes to the next player, and the last one destroys the lobby | `LEAVE_LOBBY` |
| `GAME_PACKET_TYPE_GAME_INIT` | `LobbyRequest` | Only the owner: runs the `init` of the game type | `GAME_INIT` |
| `GAME_PACKET_TYPE_GAME_START` | `LobbyRequest` | Only the owner, once: starts the threads of the lobby and the `start` of the game type | `GAME_START` |
| `GAME_PACKET_TYPE_LOBBY_UPDATE` | `LobbyRequest` | Sends the current `LobbySummary` | |
| `GAME_PACKET_TYPE_LOBBY_DESTROY` | `LobbyRequest` | Only the owner: tears the lobby down | `DESTROY_LOBBY` |

A cerver that is not a game cerver answers the game packets with an error. The cerver triggers `CERVER_EVENT_LOBBY_CREATE`, `_JOIN`, `_LEAVE`, `_START` and `_DESTROY`, with the lobby in the event data; the client triggers `CLIENT_EVENT_LOBBY_CREATE`, `_JOIN`, `_LEAVE`, `_START`, `_UPDATE`, `_DESTROY` and `CLIENT_EVENT_GAME_INIT`, with the `LobbySummary` in `response_data` (see [[Events]]).

### While a lobby runs

The `PACKET_TYPE_APP` packets of its players go to the **packet handler of the lobby** (`lobby_set_packet_handler ()`), instead of the app handler of the cerver. With `CERVER_HANDLER_TYPE_POLL`, the **lobby poll** takes the connections of its players from the poll of the cerver while the game runs, in a thread of its own; with `CERVER_HANDLER_TYPE_THREADS`, each connection is already received in its own thread. The **update** of the lobby runs every frame of its game type (`fps`), and removes the players that did not send a packet in the `player_timeout` of the game type. A player whose connection ends leaves its lobby.

### Teardown

`cerver_teardown ()` tears down every lobby before it ends the connections of the cerver: it stops their threads, gives the connections of their players back, and waits for the threads of the lobbies. `game_set_final_action ()` runs before it, like to tell every player.

## Usage

The cerver registers its game types:

``` c
static void *arcade_start (void *lobby_ptr) {

	Lobby *lobby = (Lobby *) lobby_ptr;
	lobby_send_to_players (lobby, GAME_PACKET_TYPE_GAME_START);

	return NULL;

}

Cerver *cerver = cerver_create (
	CERVER_TYPE_GAME, "game-cerver",
	7000, PROTOCOL_TCP, false, CERVER_DEFAULT_CONNECTION_QUEUE
);

cerver_set_handler_type (cerver, CERVER_HANDLER_TYPE_POLL);

GameCerver *game_cerver = (GameCerver *) cerver->cerver_data;

GameType *arcade = game_type_create ("arcade", NULL, NULL, arcade_start, NULL);
(void) game_type_add_lobby_config (arcade, true, NULL, 4);
game_type_set_player_timeout (arcade, 30);
(void) game_type_register (game_cerver->game_types, arcade);
```

The client creates a lobby and waits for its summary:

``` c
static void *on_lobby_create (void *event_data_ptr) {

	ClientEventData *event_data = (ClientEventData *) event_data_ptr;

	const LobbySummary *summary = (const LobbySummary *) event_data->response_data;
	cerver_log_msg ("Created lobby %s", summary->id);

	client_event_data_delete (event_data);

	return NULL;

}

(void) client_event_register (client, CLIENT_EVENT_LOBBY_CREATE, on_lobby_create, NULL, NULL, false, false);

LobbyCreate request;
serializer_object_init (&lobby_create_request_schema, &request);
(void) strncpy (request.game_type, "arcade", sizeof (request.game_type) - 1);

(void) game_request_send (client, connection, GAME_PACKET_TYPE_LOBBY_CREATE, &lobby_create_request_schema, &request);
```

[`examples/game.c`](https://github.com/ermiry/cerver/blob/master/examples/game.c) and [`examples/client/game.c`](https://github.com/ermiry/cerver/blob/master/examples/client/game.c) are a complete game cerver and its client.

## Reference

### The game cerver

#### `game_set_lobby_id_generator ()`

``` c
void game_set_lobby_id_generator (GameCerver *game_cerver, void *(*lobby_id_generator) (const void *));
```

Sets how the ids of the lobbies are generated; it returns a string allocated with `malloc ()`. The default is `lobby_default_id_generator ()`.

#### `lobby_default_id_generator ()`

``` c
void *lobby_default_id_generator (const void *data_ptr);
```

The default id of a lobby: the hash of its creation time and a counter, so the lobbies created in the same second have different ids.

#### `game_set_player_comparator ()`

``` c
void game_set_player_comparator (GameCerver *game_cerver, Comparator player_comparator);
```

Sets how the players of the lobbies are compared (see [[Players]]).

#### `game_set_load_game_data ()`

``` c
void game_set_load_game_data (GameCerver *game_cerver, Action load_game_data, Action delete_game_data);
```

Sets a function that loads the data of the game when the cerver starts, like to connect to a database, and the one that releases it.

#### `game_set_final_action ()`

``` c
void game_set_final_action (GameCerver *game_cerver, Action final_action, void *final_action_args);
```

Sets an action that runs right before the game cerver is torn down.

#### `game_cerver_register_lobby ()`, `game_cerver_unregister_lobby ()`

``` c
void game_cerver_register_lobby (GameCerver *game_cerver, Lobby *lobby);
void game_cerver_unregister_lobby (GameCerver *game_cerver, Lobby *lobby);
```

Add a lobby to the lobbies of the game cerver, or take it out. `lobby_create ()` and `lobby_teardown ()` already do it.

#### `game_cerver_lock ()`, `game_cerver_unlock ()`

``` c
void game_cerver_lock (const Cerver *cerver);
void game_cerver_unlock (const Cerver *cerver);
```

Take and release the recursive lock of the game cerver of a cerver, to read or change the lobbies and their players from a thread of the application. `NULL` is safe.

#### `game_cerver_stats_print ()`

``` c
void game_cerver_stats_print (Cerver *cerver);
```

Prints the lobbies that are active and the ones that were created.

#### `game_set_cerver_reference ()`, `game_new ()`, `game_delete ()`

``` c
void game_set_cerver_reference (GameCerver *game_cerver, Cerver *cerver);
GameCerver *game_new (void);
void game_delete (void *game_ptr);
```

Set the cerver of a game cerver, allocate one and delete one; `cerver_create ()` and the teardown do it.

### The client

#### `game_request_send ()`

``` c
u8 game_request_send (
	Client *client, Connection *connection,
	const u32 request_type, const SerializerSchema *schema, const void *object
);
```

Sends a game request with its object to the game cerver of the connection.

- **Returns** 0 on success, 1 on error.

## Types

| Type | What it is |
|---|---|
| `GameCerver` | The lobbies, the game types, the lock, the data of the game and the stats of a game cerver |
| `GameCerverStats` | `current_active_lobbys`, `lobbys_created` |
| `LobbyPlayer` | A `lobby` and a `player`, what the `init` of a game type gets |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `GAME_DEFAULT_PLAYER_TIMEOUT` | 30 | |
| `GAME_DEFAULT_FPS` | 20 | |
| `GAME_DEFAULT_MIN_PLAYERS`, `GAME_DEFAULT_MAX_PLAYERS` | 2, 4 | |

## See also

- [[Lobby]], [[Players]], [[Gametypes]], [[Scores]]
- [[Protocol]]: the game requests on the wire
- [[Events]]: the lobby events
