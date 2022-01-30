# Building and testing
# [Building and testing](https://github.com/openark/orchestrator/blob/master/docs/build.md)
Developers have multiple ways to build and test `orchestrator`.

* Using GitHub's CI, no development environment needed
* Using Docker
* Build locally on dev machine

## Build and test via GitHub CI
`orchestrator`'s' [CI Build](https://github.com/openark/orchestrator/blob/master/docs/ci.md) will:

* build
* test (unit, integration)
* upload an artifact: an `orchestrator` binary compatible with Linux `amd64`

The artifact is attached in the build's output, and valid for a couple months per GitHub Actions policy.

This way, a developer only needs to `git checkout/commit/push` and does not require any development environment on their computer. Once CI completes (successfully), the developer may download the binary artifact to test on a Linux environment.

## Build and test via Docker
Requirements: a docker installation.

`orchestrator` provides [various docker builds](https://github.com/openark/orchestrator/blob/master/docs/docker.md). For developers:

* run `script/dock alpine` to build and run `orchestrator` service
* run `script/dock test` to build `orchestrator`, run unit tests, integration tests, documentation tests
* run `script/dock pkg` to build `orchestrator` and create distribution packages (`.deb/.rpm/.tgz`)
* run `script/dock system` to build and launch a full CI environment which includes a MySQL topology, HAProxy, Consul, consul-template and `orchestrator` running as a service.

## Build and test on dev machine
Requirements:

* `go` development setup (at this time `go1.12` or above required)
* `git`
* `gcc` (required to build `SQLite` as part of the `orchestrator` binary)
* Linux, BSD or MacOS

Run:

```bash
    git clone git@github.com:openark/orchestrator.git
    cd orchestrator
```
### Build
Build via:

```Plain Text
    ./script/build
```
This takes care of `GOPATH` and various other considerations.

Alternatively, if you like and if your Go environment is setup, you may run:

```Plain Text
    go build -o bin/orchestrator -i go/cmd/orchestrator/main.go
```
### Run
Find artifacts under `bin/` directory and e.g. run:

```Plain Text
    bin/orchestrator --debug http
```
### Setup backend DB
If running with SQLite backend, no DB setup is needed. The rest of this section assumes you have a MySQL backend.

For `orchestrator` to detect your replication topologies, it must also have an account on each and every topology. At this stage this has to be the same account (same user, same password) for all topologies. On each of your masters, issue the following:

```Plain Text
    CREATE USER 'orc_user'@'%' IDENTIFIED BY 'orc_password';
    GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orc_user'@'%';
```
Replace `%` with a specific hostname/`127.0.0.1`/subnet. Choose your password wisely. Edit `orchestrator.conf.json` to match:

```Plain Text
    "MySQLTopologyUser": "orc_user",
    "MySQLTopologyPassword": "orc_password",
```
