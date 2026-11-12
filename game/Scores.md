# Scores

`#include <cerver/game/score.h>`

A scoreboard keeps the scores of the players of a game, by **score type**: every player has a value for each type, like `"kills"` and `"deaths"`. The values are never negative: they are clamped to 0.

## Usage

``` c
ScoreBoard *scores = game_score_create (4, 2, "kills", "deaths");

game_score_add_lobby_players (scores, lobby);

game_score_update (scores, "player-one", "kills", 1);
i32 kills = game_score_get (scores, "player-one", "kills");

game_score_delete (scores);
```

## Reference

#### `game_score_create ()`, `game_score_new ()`, `game_score_delete ()`

``` c
ScoreBoard *game_score_create (unsigned int playersNum, unsigned int scoresNum, ...);
ScoreBoard *game_score_new (void);
void game_score_delete (ScoreBoard *sb);
```

Create a scoreboard with room for `playersNum` players and `scoresNum` score types, passed as `char *` arguments; allocate an empty one; delete one.

#### `game_score_add_scoreType ()`, `game_score_remove_scoreType ()`

``` c
void game_score_add_scoreType (ScoreBoard *sb, char *newScore);
u8 game_score_remove_scoreType (ScoreBoard *sb, char *oldScore);
```

Add a score type, with 0 for every player (an existing one is ignored), and remove one from every player (0 on success, 1 if it does not exist).

#### `game_score_add_player ()`, `game_score_remove_player ()`, `game_score_add_lobby_players ()`

``` c
u8 game_score_add_player (ScoreBoard *sb, char *playerName);
u8 game_score_remove_player (ScoreBoard *sb, char *playerName);
void game_score_add_lobby_players (ScoreBoard *sb, const Lobby *lobby);
```

Add a player with every score in 0 (1 if it already exists), remove one (1 if it does not exist), and add the players of a lobby by their ids, skipping the ones without an id.

#### `game_score_set ()`, `game_score_get ()`, `game_score_update ()`, `game_score_reset ()`

``` c
void game_score_set (ScoreBoard *sb, char *playerName, char *scoreType, i32 value);
i32 game_score_get (ScoreBoard *sb, char *playerName, char *scoreType);
void game_score_update (ScoreBoard *sb, char *playerName, char *scoreType, i32 value);
void game_score_reset (ScoreBoard *sb, char *playerName);
```

Set a score (a negative value is 0), get it (-1 if it does not exist), add a value that can be negative (the result is clamped to 0), and set every score of a player to 0.

## Types

| Type | What it is |
|---|---|
| `ScoreBoard` | The scores by player and score type, the names of the players and the score types |

## Constants

| Constant | Value | What it is |
|---|---|---|
| `SCORE_DEFAULT_SIZE` | 5 | The players that a scoreboard has room for when it is created |

## See also

- [[Players]]: the ids that the scoreboards use
