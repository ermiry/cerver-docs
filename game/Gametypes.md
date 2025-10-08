# Gametypes

`#include <cerver/game/gametype.h>`

A game type describes a game that the [[lobbies|Lobby]] of a game cerver can play: its name, what to do when a game starts and ends, how its lobbies are configured, and the callbacks of its players. A client asks for a lobby of a game type by its name (see [[Game]]).

## Concepts

| Part | What it is |
|---|---|
| `start`, `end` | Called with the `Lobby *` when the game of a lobby starts and ends; `NULL` is safe |
| `use_default_handler`, `custom_handler`, `max_players` | The configuration of its lobbies: the lobby poll or a handler of its own, and how many players |
| `on_lobby_join`, `on_lobby_leave` | Called when a player joins and leaves, with the player and the lobby |
| `init` | Called with a `LobbyPlayer` of the owner when it sends `GAME_INIT`, before the game starts |
| `player_timeout` | The seconds without packets after which a player leaves its lobby, 0 never |
| `fps` | The frames per second of the update of its lobbies |
| `data`, `delete_data` | Data of the game type |

The callbacks run with the lock of the game cerver taken, and can use the API of the lobbies.

## Usage

``` c
GameType *arcade = game_type_create ("arcade", NULL, NULL, arcade_start, arcade_end);

(void) game_type_add_lobby_config (arcade, true, NULL, 4);
game_type_set_on_lobby_join (arcade, arcade_on_join);
game_type_set_player_timeout (arcade, 30);
game_type_set_fps (arcade, 30);

GameCerver *game_cerver = (GameCerver *) cerver->cerver_data;
(void) game_type_register (game_cerver->game_types, arcade);
```

## Reference

#### `game_type_create ()`, `game_type_new ()`, `game_type_delete ()`

``` c
GameType *game_type_create (
	const char *name, void *data, void (*delete_data)(void *),
	void *(*start)(void *), void *(*end) (void *)
);

GameType *game_type_new (void);
void game_type_delete (void *ptr);
```

Create a game type with a name (required), its data (`delete_data` `NULL` releases it with `free ()`), and its start and end; allocate an empty one; delete one.

#### `game_type_add_lobby_config ()`

``` c
int game_type_add_lobby_config (GameType *game_type, bool use_default_handler, Action custom_handler, u32 max_players);
```

Sets the configuration of the lobbies of the game type. `max_players` 0 takes `LOBBY_DEFAULT_MAX_PLAYERS`.

- **Returns** 0 on success, 1 on error.

#### `game_type_set_on_lobby_join ()`, `game_type_set_on_lobby_leave ()`, `game_type_set_init ()`

``` c
void game_type_set_on_lobby_join (GameType *game_type, Action on_lobby_join);
void game_type_set_on_lobby_leave (GameType *game_type, Action on_lobby_leave);
void game_type_set_init (GameType *game_type, Action init);
```

Set the callbacks of the players and of `GAME_INIT`.

#### `game_type_set_player_timeout ()`, `game_type_set_fps ()`

``` c
void game_type_set_player_timeout (GameType *game_type, const u32 player_timeout);
void game_type_set_fps (GameType *game_type, const u32 fps);
```

Set the timeout of the players (`GAME_TYPE_DEFAULT_PLAYER_TIMEOUT`, 0: never) and the frames per second of the update (`GAME_TYPE_DEFAULT_FPS`, 20; 0 is ignored).

#### `game_type_register ()`, `game_type_unregister ()`, `game_type_get_by_name ()`

``` c
int game_type_register (DoubleList *game_types, GameType *game_type);
int game_type_unregister (DoubleList *game_types, const char *name);
GameType *game_type_get_by_name (DoubleList *game_types, const char *name);
```

Add a game type to the game types of a game cerver (`game_cerver->game_types`), remove it by its name, and find it by its name. Register them before the cerver starts. The two first return 0 on success, 1 on error; the last one returns `NULL` when it is not there.

## Types

| Type | What it is |
|---|---|
| `GameType` | The parts above |

## Constants

| Constant | Value |
|---|---|
| `GAME_TYPE_DEFAULT_PLAYER_TIMEOUT` | 0 |
| `GAME_TYPE_DEFAULT_FPS` | 20 |

## See also

- [[Game]]: the game cerver and its requests
- [[Lobby]]: the lobbies of a game type
