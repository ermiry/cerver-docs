# Examples

The `examples` directory of the repository has a program for each feature. `make examples` builds them in `examples/bin`, after `make` (see [[Building]]). Most of them share a small library, `examples/app`, with the handlers, the events and the authentication of the examples; run them with both libraries in the path:

``` bash
LD_LIBRARY_PATH=bin:examples/bin/app ./examples/bin/welcome
```

## Cervers

| Example | What it shows |
|---|---|
| `start` | The echo cerver and client of the [[Quickstart]]: `start server` and `start client` |
| `welcome` | The smallest cerver: a welcome message, the poll handler type and the events |
| `test` | A cerver that answers the test requests of its clients |
| `handlers` | The app handlers, direct or with a queue (`-t q`) |
| `multi` | Multiple app handlers, chosen by the `handler_id` of each packet, each with its own data |
| `requests` | Handlers that answer requests, with the data of the handler |
| `threads` | `CERVER_HANDLER_TYPE_THREADS` with its thpool |
| `packets` | Sending and receiving the packets of the application |
| `advanced` | Several features of a cerver together |
| `auth` | A cerver that requires authentication |
| `sessions` | Authentication with sessions |
| `admin` | A cerver with admins |
| `files` | A files cerver: files to download and an uploads path |
| `logs` | The log: a file, the time and the types |
| `game` | A game cerver with a game type and its lobbies |
| `balancer`, `service` | A load balancer and the services behind it (`--type round_robin` or `handler_id`) |

## Clients

| Example | What it shows |
|---|---|
| `client/client` | A client that connects, sends requests and handles the answers |
| `client/auth` | A client that authenticates, with credentials and with a session |
| `client/files` | A client that downloads and uploads files |
| `client/game` | A client that creates, joins and starts lobbies |

## As tests

`test/integration.sh` runs several of them as pairs of a cerver and a client: `examples/start`, `welcome`, `test`, `handlers`, `requests`, `threads` and `game` (see [[Testing]]). So the examples are built and run on every change.

## See also

- [[Quickstart]]: the first example, step by step
- [[Testing]]: how the examples are tested
