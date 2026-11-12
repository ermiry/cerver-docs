# Building

Cerver builds in Linux, with gcc 16.2 (the compiler of its CI), g++ or clang, and GNU `make`. It only links pthreads and libm, both in glibc. macOS is not supported yet: its port is planned for cerver 1.9.

## Dependencies

| What | Debian / Ubuntu |
|---|---|
| A C compiler and `make` | `sudo apt-get install build-essential` |
| clang, for `CC=clang` and the fuzzers | `sudo apt-get install clang libclang-rt-dev` |
| valgrind, gdb (optional, for the tests) | `sudo apt-get install valgrind gdb` |

## Targets

| Command | What it does |
|---|---|
| `make` | Builds `bin/libcerver.so` from `src/cerver` |
| `make examples` | Builds `examples/bin`, against `bin/libcerver.so` (run `make` first) |
| `make test` | Builds the unit and integration tests in `test/bin` |
| `make fuzz` | Builds the fuzzers in `test/bin/fuzz`, with `CC=clang` |
| `make coverage` | Runs gcov over the objects of a `TYPE=test COVERAGE=1` build and writes `coverage/` |
| `make bench` | Builds the benchmarks of base64 and of the handlers in `benchmarks/bin` |
| `sudo make install` | Copies `bin/libcerver.so` to `/usr/local/lib` and `include/cerver` to `/usr/local/include` |
| `sudo make uninstall` | Removes both |
| `make clean` | Removes the objects and the binaries of the library, the examples, the tests, the coverage and the benchmarks |

After `make install`, run `sudo ldconfig` so the linker finds `libcerver.so`, and link your program with `-l cerver -l pthread`.

## Options

Every option is a `make` variable, like `make TYPE=test SANITIZER=address -j4`:

| Variable | Values | Default | What it does |
|---|---|---|---|
| `TYPE` | `development`, `test`, `beta`, `production` | `development` | `development` builds with `-g` and the debug logs and stats of every module (`CERVER_DEBUG`, `CERVER_STATS`, `CLIENT_DEBUG`, …). `test` builds with `-O2`, `-g`, `_FORTIFY_SOURCE` and the stack protector. `beta` builds with `-O2` and `-g`. Any other value, like `production`, builds with `-O2` and `_FORTIFY_SOURCE` |
| `CC` | `gcc`, `g++`, `clang` | `gcc` | The compiler; `g++` builds the library as C++ 11 |
| `WERROR` | `0`, `1` | `0` | `1` turns every warning into an error, like the CI does |
| `SANITIZER` | `address`, `undefined`, `thread`, or a list like `address,undefined` | empty | Builds with `-fsanitize`; it also disables the recycling of connections, sockets, clients and packets (`CERVER_RECYCLE_DISABLED`), so the sanitizers see every allocation |
| `COVERAGE` | `0`, `1` | `0` | With `TYPE=test`, builds with gcov |
| `NATIVE` | `0`, `1` | `0` | Builds with `-march=native` |
| `DEBUG` | `0`, `1` | `0` | Builds the library of the examples with its debug logs |

Always run `make clean` before changing `TYPE`, `CC`, `SANITIZER` or `COVERAGE`: the objects of another build are not rebuilt.

## Docker images

The repository has a Dockerfile for each use, all on `gcc:16.2` for the build and `debian:trixie-slim` to run:

| Dockerfile | What it has |
|---|---|
| `Dockerfile` | The release of `version.txt`, downloaded from GitHub and built with `TYPE=production`, installed in a slim image |
| `Dockerfile.dev` | The local sources with the examples and the tests, in a development build |
| `Dockerfile.test` | The local sources with `TYPE=test`, the tests and the examples, `WERROR=1`; `--build-arg SANITIZER=…` and `COVERAGE=1` select the variant |
| `Dockerfile.local` | The local sources with `TYPE=test` and the tests, in a slim image |
| `Dockerfile.beta`, `Dockerfile.prod` | The local sources in a beta or a production build, with the examples |
| `Dockerfile.hub` | The local sources in a production build, without the examples |
| `Dockerfile.builder`, `Dockerfile.buildev` | A `gcc:16.2` image with cerver installed, in a production or a development build, to build other programs |
| `Dockerfile.fuzz` | The fuzzers, built with clang |

``` bash
docker build -t cerver:test -f Dockerfile.test .
docker run --rm cerver:test bash test/run.sh
```

## Installing a release

`cerver-install.sh` downloads the release of its `CERVER_VERSION` from GitHub to `/opt`, builds it and installs it:

``` bash
bash cerver-install.sh
```

## See also

- [[Quickstart]]: a first cerver
- [[Testing]]: the tests and the sanitizers
