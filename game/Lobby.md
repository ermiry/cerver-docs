# Lobby

`#include <cerver/game/lobby.h>`

A lobby is a group of [[Players]] of a game cerver that play a game of a [[game type|Gametypes]]: it has an id, an owner, its players, the state of its game, its own threads while it runs, and the packets of its players. The requests that create, join and start the lobbies are in [[Game]].

## Concepts

### References

A lobby is released with its **last reference**: the game cerver holds one while the lobby is registered, and each thread of the lobby, each packet of one of its players and each event of the lobby holds another. So an event or a handler can use its lobby after the lobby was torn down; `lobby_release ()` gives the reference back. `lobby_teardown ()` stops the lobby and releases the reference of the game cerver.

### State

`running` (its threads are receiving), `in_game` (the game started), `initialized` (the owner sent `GAME_INIT`) and `ended` (it was torn down). The players, the owner, the counts and the state are read and written with the lock of the game cerver (`game_cerver_lock ()`).

### Threads

`lobby_start ()` starts the **handler** of the lobby, the lobby poll by default (`lobby_poll ()`), that receives the packets of its players with `CERVER_HANDLER_TYPE_POLL`, and the **update**, that runs `lobby_set_update ()` every frame of the game type and removes the players that passed their timeout. Both get a `CerverLobby *` with the cerver and the lobby. A lobby that a thread of its own ends is detached, never joined from its own thread.

## Usage

A lobby created by the cerver itself, outside of the requests of the clients:

``` c
Lobby *lobby = lobby_create (cerver, owner_client, true, NULL, 4);
if (lobby) {
	lobby_set_packet_handler (lobby, my_lobby_packet_handler);
	lobby_set_update (lobby, my_lobby_update);

	(void) lobby_start (cerver, lobby);
}
```

## Reference

### Create and run

#### `lobby_create ()`

``` c
Lobby *lobby_create (
	Cerver *cerver, Client *client,
	bool use_default_handler, Action custom_handler, u32 max_players
);
```

Creates a lobby with a new player of the client as its owner, and registers it. With `use_default_handler` its handler is the lobby poll, otherwise `custom_handler`. `max_players` 0 takes `LOBBY_DEFAULT_MAX_PLAYERS` (4).

- **Returns** the lobby, or `NULL` on error.

#### `lobby_join ()`

``` c
u8 lobby_join (Cerver *cerver, Client *client, Lobby *lobby);
```

Adds a new player of the client to the lobby.

- **Returns** 0 on success, 1 on error: a full lobby, one in game, one that ended, or a client that is already in it.

#### `lobby_leave ()`

``` c
u8 lobby_leave (Cerver *cerver, Lobby *lobby, Player *player);
```

The player leaves the lobby and is released; the owner passes to the next player, and the last player destroys the lobby.

- **Returns** 0 on success, 1 on error.

#### `lobby_start ()`

``` c
u8 lobby_start (Cerver *cerver, Lobby *lobby);
```

Starts the handler and the update threads of the lobby.

- **Returns** 0 on success, 1 on error or if it was already started.

#### `lobby_teardown ()`

``` c
int lobby_teardown (GameCerver *game_cerver, Lobby *lobby);
```

Stops the threads of the lobby, gives the connections of its players back to the cerver, releases its game data and the reference of the game cerver. The lobby is released with its last reference.

- **Returns** 0 on success, 1 on error.

#### `lobby_is_running ()`

``` c
bool lobby_is_running (const Lobby *lobby);
```

Returns `true` while the lobby runs.

### Configuration

#### `lobby_set_handler ()`, `lobby_set_poll_time_out ()`

``` c
void lobby_set_handler (Lobby *lobby, Action handler);
void lobby_set_poll_time_out (Lobby *lobby, unsigned int timeout);
```

Replace the handler of the lobby, that gets a `CerverLobby *`, and set the timeout in milliseconds of each `poll ()` of the lobby poll (`LOBBY_DEFAULT_POLL_TIMEOUT`, 2000).

#### `lobby_set_packet_handler ()`

``` c
void lobby_set_packet_handler (Lobby *lobby, Action packet_handler);
```

Sets the handler of the `PACKET_TYPE_APP` packets of the players while the lobby runs; it gets the `Packet *`, released after it.

#### `lobby_set_update ()`

``` c
void lobby_set_update (Lobby *lobby, Action update);
```

Sets the action that runs every frame of the game type while the lobby runs, with a `CerverLobby *`.

#### `lobby_set_game_settings ()`, `lobby_set_game_data ()`

``` c
void lobby_set_game_settings (Lobby *lobby, void *game_settings, Action game_settings_delete);
void lobby_set_game_data (Lobby *lobby, void *game_data, Action game_data_delete);
```

Keep the settings and the data of the game in the lobby, released with their delete functions when the lobby is.

### Search

#### `lobby_search_by_id ()`

``` c
Lobby *lobby_search_by_id (Cerver *cerver, const char *id);
```

Returns the lobby of a game cerver with the id, with the lock of the game cerver, or `NULL`.

#### `lobby_get ()`

``` c
Lobby *lobby_get (GameCerver *game_cerver, Lobby *query);
```

Returns the lobby that compares equal to `query` (by its id), with a reference that the caller releases with `lobby_release ()`, or `NULL`.

#### `lobby_player_of_client ()`

``` c
Player *lobby_player_of_client (const Lobby *lobby, const Client *client);
```

Returns the player of a client in the lobby, or `NULL`. Call it with the lock of the game cerver.

#### `lobby_comparator ()`

``` c
int lobby_comparator (const void *one, const void *two);
```

Compares two lobbies by their ids.

### References

#### `lobby_reference ()`, `lobby_release ()`

``` c
Lobby *lobby_reference (Lobby *lobby);
void lobby_release (Lobby *lobby);
```

Take a reference to a lobby (it returns the same lobby), and release one; the last one deletes the lobby. `NULL` is safe in both.

#### `lobby_new ()`, `lobby_delete ()`

``` c
Lobby *lobby_new (void);
void lobby_delete (void *lobby_ptr);
```

Allocate a lobby with the reference of its creator, and release a lobby right away, without looking at its references; use `lobby_release ()` for a lobby that is shared.

### Summaries

#### `lobby_summary_set ()`

``` c
void lobby_summary_set (LobbySummary *summary, const Lobby *lobby, const bool created);
```

Fills a `LobbySummary` with the state of the lobby, with the lock of the game cerver; `created` tells a client that the lobby was just created.

#### `lobby_summary_packet_create ()`

``` c
Packet *lobby_summary_packet_create (const Lobby *lobby, const u32 request_type, const bool created);
```

Creates a `GAME` packet of the request type with the summary of the lobby, ready to be sent, that the caller deletes.

#### `lobby_send ()`

``` c
void lobby_send (Lobby *lobby, bool created, Cerver *cerver, Client *client, Connection *connection);
```

Sends the summary of the lobby to a connection, as the answer of `LOBBY_CREATE` when it was just created, or of `LOBBY_JOIN`.

#### `lobby_send_to_players ()`

``` c
void lobby_send_to_players (Lobby *lobby, const u32 request_type);
```

Sends the summary of the lobby to every player, in a `GAME` packet of the request type.

#### `lobby_poll ()`

``` c
void lobby_poll (void *ptr);
```

The default handler of a lobby, that receives the packets of its players while the lobby runs; it gets a `CerverLobby *`.

### Stats

#### `lobby_stats_print ()`, `lobby_stats_set_threshold_time ()`

``` c
void lobby_stats_print (Lobby *lobby);
void lobby_stats_set_threshold_time (Lobby *lobby, time_t threshold_time);
```

Print the packets, receives and bytes of the lobby, and set how often they are reset.

## Objects

The objects of the requests and answers, as schemas of the [[Serializer]]:

| Object | Fields |
|---|---|
| `LobbyCreate` | 1: `game_type` (required) |
| `LobbyJoin` | 1: `lobby_id`, 2: `game_type` (required) |
| `LobbyRequest` | 1: `lobby_id` (required) |
| `LobbyPlayerSummary` | 1: `client_id` (required), 2: `player_id`, 3: `owner` |
| `LobbySettingsSummary` | 1: `max_players`, 2: `player_timeout`, 3: `fps` |
| `LobbySummary` | 1: `id` (required), 2: `creation`, 3: `game_type`, 4: `created`, 5: `running`, 6: `in_game`, 7: `max_players`, 8: `n_players`, 9: `owner_client_id`, 10: `players` (up to 32 `LobbyPlayerSummary`), 11: `settings` |

## Types

| Type | What it is |
|---|---|
| `Lobby` | A lobby: `id`, its references, its state, `players`, `owner`, its counts, its handler, packet handler and update, its `game_type`, settings and data, and its `stats` |
| `CerverLobby` | What the handler and the update get: `cerver`, `lobby` |
| `GameSettings` | `game_type`, `player_timeout`, `fps`, `min_players`, `max_players`, `duration` |
| `LobbyStats` | The packets, receives and bytes of a lobby |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `LOBBY_DEFAULT_MAX_PLAYERS` | 4 | |
| `LOBBY_DEFAULT_POLL_TIMEOUT` | 2000 | Milliseconds of the lobby poll |
| `LOBBY_ID_SIZE`, `LOBBY_GAME_TYPE_SIZE`, `LOBBY_PLAYER_ID_SIZE` | 80, 64, 64 | The texts of the objects, with their NUL |
| `LOBBY_SUMMARY_PLAYERS_MAX` | 32 | The players that a summary sends |
| `LOBBY_OBJECTS_VERSION` | 1 | The version of their schemas |

## See also

- [[Game]]: the requests of the clients
- [[Players]] and [[Gametypes]]
