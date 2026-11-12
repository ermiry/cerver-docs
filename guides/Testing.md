# Testing

How cerver is tested, and how to run the tests to contribute. Every suite runs in Docker with `gcc:16.2`, the image of the CI, so the results are the same on any machine.

## The suites

| Suite | What it runs | Command |
|---|---|---|
| Unit | The modules on their own, and cervers with their clients inside a single process (`test/bin/...`) | `bash test/run.sh` inside the test image |
| Integration | Pairs of a cerver process and a client process: ping, packets, requests, schemas, authentication, sessions, threads, queues, UDP, games, the handlers, the balancer with its services, and the examples, in IPv4 and in IPv6 (`::1`) | `bash test/integration.sh` |
| Repeat | A test many times, to find the flaky ones; it keeps the logs of a run that fails | `bash test/repeat.sh` |
| Fuzz | libFuzzer over the receive of packets, the packets, the client, the file headers, the datagrams, the addresses, the serializer and the games | `bash test/fuzz.sh` |
| Allocations | The allocations per connection and per packet, with valgrind, against limits | `bash test/allocations.sh` |
| Stress | Many clients against a cerver | `bash test/stress.sh` |

## Running them

``` bash
# the unit tests
docker build -t cerver:test -f Dockerfile.test .
docker run --rm cerver:test bash test/run.sh

# with the sanitizers
docker build -t cerver:test-address -f Dockerfile.test --build-arg SANITIZER=address,undefined .
docker run --rm cerver:test-address bash test/run.sh

# every pair, in both families; or only some of them
bash test/integration.sh
ADDRESSES=127.0.0.1 bash test/integration.sh cerver/ping examples/start

# the fuzzers, 30 seconds each
SECONDS_PER_TARGET=30 bash test/fuzz.sh
```

- `integration.sh` builds its image (`SANITIZER=…`, `COVERAGE=1`) and runs each pair in a container, with `--add-host service-1` and `service-2` for the balancer and IPv6 enabled. `test/pair.sh` starts the servers, waits until they listen, runs the client, stops the servers with a signal and checks that every process exits with 0.
- ThreadSanitizer (`SANITIZER=thread`) needs `--security-opt seccomp=unconfined`, and on some virtual machines a lower `vm.mmap_rnd_bits`.
- valgrind: `RUNNER="valgrind --leak-check=full --error-exitcode=1" bash test/run.sh` inside the image.
- `COVERAGE=1` with `TYPE=test` builds with gcov; `make coverage` writes the reports.

## The rules of a change

- Every bug is fixed with a test, and the test is **seen failing** against the version before the fix.
- Zero warnings with gcc, g++ and clang with `WERROR=1`.
- The unit tests pass without sanitizers, with AddressSanitizer and UndefinedBehaviorSanitizer, with ThreadSanitizer and with valgrind; the integration pairs pass in IPv4 and IPv6.
- The CI runs all of it on every push and pull request (`build.yml`, `test.yml`, `beta.yml`, `production.yml`).

## See also

- [[Building]]: the options of the builds
- [[Examples]]: the examples that run as pairs
- [[Performance]]: what the allocations suite measures
