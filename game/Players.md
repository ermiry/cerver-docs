# Players

`#include <cerver/game/player.h>`

A player is a client inside a [[lobby|Lobby]]: its `Client` with the connections that it plays with, an optional id, the last time that it sent a packet to its lobby, and data of the game. The lobbies create and release their players.

## Concepts

- A player belongs to a single lobby; the same client can be a player of different lobbies.
- `last_activity` is the time, in milliseconds of the monotonic clock, of the last packet that the player sent to its lobby, read and written atomically. The update of the lobby removes the players that passed the `player_timeout` of the game type.
- The players of a lobby are read and changed with the lock of the game cerver (`game_cerver_lock ()`).

## Reference

#### `player_new ()`, `player_delete ()`

``` c
Player *player_new (void);
void player_delete (void *player_ptr);
```

Allocate a player, and delete one with its id and its data.

#### `player_set_id ()`

``` c
void player_set_id (Player *player, const char *id);
```

Sets the id of the player, like the name of its user; the scoreboards use it (see [[Scores]]).

#### `player_set_data ()`

``` c
void player_set_data (Player *player, void *data, Action data_delete);
```

Keeps data of the game in the player, released with `data_delete` when the player is deleted.

#### `player_get_from_lobby ()`, `player_get_le_from_lobby ()`, `player_get_by_sock_fd_list ()`

``` c
Player *player_get_from_lobby (Lobby *lobby, Player *query);
ListElement *player_get_le_from_lobby (Lobby *lobby, Player *player);
Player *player_get_by_sock_fd_list (Lobby *lobby, i32 sock_fd);
```

Find a player of a lobby that compares equal to `query` with the comparator of the game cerver, the element of the list of a player, and the player with a connection of the socket. They return `NULL` when it is not there.

#### `player_broadcast_to_all ()`

``` c
void player_broadcast_to_all (
	Cerver *cerver, const Lobby *lobby, Packet *packet,
	Protocol protocol, int flags
);
```

Sends a generated packet to every player of the lobby.

#### `player_comparator_by_id ()`, `player_comparator_client_id ()`

``` c
int player_comparator_by_id (const void *a, const void *b);
int player_comparator_client_id (const void *a, const void *b);
```

Compare two players by their ids, or by the ids of their clients; one of them is the comparator of the game cerver (`game_set_player_comparator ()`).

#### `player_activity_now ()`, `player_activity_update ()`

``` c
u64 player_activity_now (void);
void player_activity_update (Player *player);
```

The current time in milliseconds of the monotonic clock that the activity uses, and marking that a player sent a packet now.

## Types

| Type | What it is |
|---|---|
| `Player` | `id`, `client`, `last_activity`, `data`, `data_delete` |

## See also

- [[Lobby]]: where the players are
- [[Scores]]: the scores of the players
