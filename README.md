# anyctr

**One CLI for any container runtime.** You type Docker's convention; anyctr runs it
on whatever the engineer in front of the keyboard actually has installed — Docker,
Podman, or Apple's `container` — on macOS, Linux, and Windows.

```console
$ anyctr run -d -p 8080:80 --name web nginx:alpine
$ anyctr ps
$ anyctr logs -f web
$ anyctr exec -it web sh
```

Those four commands work unchanged on all three runtimes. Under the hood the last
one is `docker exec -it web sh`, or `podman exec -it web sh`, or
`container exec --interactive --tty web sh`.

## Why

Container runtimes have diverged. Apple's `container` calls it `list`, not `ps`;
images live under `container image ls`; there is no `--restart`, no `--privileged`,
no Compose. Podman is argv-compatible with Docker but rootless, with its own GPU
and logging story. Teams end up with per-machine scripts, or a mandated runtime
nobody wanted.

anyctr's premise: engineers pick their runtime, the commands stay the same.

**And when they can't stay the same, anyctr says so.** A gateway that quietly drops
`--restart` because the target runtime lacks it has changed what you asked for:

```console
$ anyctr run --restart=always -d nginx
anyctr: error: --restart is not supported by runtime "container" (apple/container 1.2.2)

  Apple's container has no restart policy.
  Nearest option: supervise the container yourself (launchd, a wrapper script),
  or select a runtime that has one:
      anyctr --runtime=podman run --restart=always ...

  To drop unsupported flags instead of failing:
    anyctr --relax run ...        (warns on stderr)
    anyctr config set strict false
```

## Install

```console
brew install pedrospdc/tap/anyctr
```

or, on macOS and Linux:

```console
curl -fsSL https://raw.githubusercontent.com/pedrospdc/anyctr/main/install.sh | sh
```

or with Go, or from a clone:

```console
go install github.com/pedrospdc/anyctr/cmd/anyctr@latest   # requires Go 1.25+
make install
```

Prebuilt binaries for darwin, linux and windows on amd64 and arm64 are attached to
[each release](https://github.com/pedrospdc/anyctr/releases), with checksums. On
Windows, download the `.zip` and put `anyctr.exe` on your PATH.

## Getting started

See what anyctr found on your machine:

```console
$ anyctr runtime list
   NAME       STATUS             VERSION  BINARY                    DESCRIPTION
   docker     not found on PATH  -        -                         Docker CLI (Docker Engine / Docker Desktop)
   podman     not found on PATH  -        -                         Podman CLI (daemonless, rootless by default)
*  container  available          1.2.2    /usr/local/bin/container  Apple container (one lightweight VM per container)

* selected for this machine (override with --runtime or ANYCTR_RUNTIME)
```

Ask what a command would become, without running it:

```console
$ anyctr --explain --runtime=container run -d -p 8080:80 --name web nginx:alpine
runtime: container (apple/container 1.2.2)
binary:  /usr/local/bin/container
command: /usr/local/bin/container run --detach --name web --publish 8080:80 nginx:alpine
```

`--explain` works for runtimes you do not have installed, which makes it the fastest
way to check portability before you commit a script.

## Commands

| anyctr | docker / podman | apple container |
| --- | --- | --- |
| `run` | `run` | `run` |
| `ps` | `ps` | `list` |
| `images` | `images` | `image list` |
| `build` | `build` | `build` |
| `pull` | `pull` | `image pull` |
| `push` | `push` | `image push` |
| `logs` | `logs` | `logs` |
| `exec` | `exec` | `exec` |
| `stop` | `stop` | `stop` |
| `rm` | `rm` | `delete` |
| `rmi` | `rmi` | `image delete` |
| `tag` | `tag` | `image tag` |
| `compose` | `docker compose` / `podman compose` | `container-compose` |

Flag-by-flag support is in [docs/CAPABILITIES.md](docs/CAPABILITIES.md), generated
from the code. For a live view of just the flags that need attention:

```console
anyctr runtime capabilities            # only what differs
anyctr runtime capabilities --all      # the full matrix
anyctr runtime capabilities run        # one command
```

## Choosing a runtime

Highest precedence first:

```
--runtime=<name>          per invocation
ANYCTR_RUNTIME=<name>     per shell
runtime = "<name>"        config file pin
priority order            first one found on PATH (docker, podman, container)
```

A pinned runtime that is not installed is an error, never a silent fallback —
falling back would run your command on a different engine than you asked for.

## When a flag doesn't translate

Three levers, in increasing order of bluntness:

```console
anyctr --relax run --restart=always -d nginx   # this once: drop it, warn on stderr
anyctr config set flags.restart drop           # always drop this one flag
anyctr config set strict false                 # drop anything untranslatable
```

And one escape hatch that translates nothing at all:

```console
anyctr raw run --rosetta --rm -it alpine sh    # runtime-specific, not portable
```

`raw` exists so a missing mapping never blocks anyone. Use it for genuinely
runtime-specific features; use the flags above when you want portability with your
eyes open.

## Compose

`anyctr compose` routes to whichever Compose the runtime provides, forwarding
arguments verbatim — anyctr does not parse compose files:

| runtime | provider |
| --- | --- |
| docker | the `docker compose` plugin, falling back to a standalone `docker-compose` |
| podman | `podman compose` (which drives a provider of its own) |
| container | [`container-compose`](https://github.com/pedrospdc/apple-container-compose) |

Apple's `container` ships no Compose, so anyctr routes to `container-compose`.
`anyctr runtime list` shows the provider for every runtime and flags a missing
one, and `anyctr compose` offers to install it on the spot:

```console
$ anyctr compose up -d
runtime "container" ships no Compose implementation

  anyctr routes `compose` to container-compose, which is not installed:
    brew install pedrospdc/tap/container-compose
    https://github.com/pedrospdc/apple-container-compose

Install it now with `brew install pedrospdc/tap/container-compose`? [y/N]
```

Answering no — or running without a terminal, as in CI — prints the same
explanation and exits, rather than blocking on a prompt nobody will answer.

## Using it as `docker`

Scripts, Makefiles and CI configs call `docker`. On a machine that runs Apple's
container or Podman instead, `anyctr alias` gives them one:

```console
$ anyctr alias
No docker command found. Create one that runs anyctr (for Makefiles and scripts)? [Y/n] y
Created /opt/homebrew/bin/docker -> /opt/homebrew/bin/anyctr

`docker` now runs anyctr. Undo it with `anyctr alias --remove`.
```

```console
$ docker run --rm alpine echo hi      # on a machine with no Docker
hi
```

It installs a symlink, not a shell alias, because `make` and anything else that
shells out runs `/bin/sh`, which never sees the aliases in your `.zshrc` — and
the hand-typed case is the one that needs the least help.

anyctr will not shadow a real Docker: if `docker` already exists it says so and
leaves it alone, unless you pass `--force`. `anyctr runtime list` offers the
alias when it is missing, and `anyctr alias --remove` undoes it.

Shell completions come from the same command surface:

```console
anyctr completion zsh > "${fpath[1]}/_anyctr"      # bash, fish, powershell too
```

## Configuration

Optional. `~/.config/anyctr/config.toml` (`%APPDATA%\anyctr\config.toml` on
Windows, `$ANYCTR_CONFIG` overrides both):

```toml
runtime  = "podman"                            # pin; omit to auto-detect
strict   = true                                # fail on untranslatable flags
priority = ["container", "podman", "docker"]   # auto-detection order

[binaries]
docker = "/opt/homebrew/bin/docker"            # not on PATH

[flags]
restart = "drop"                               # per-flag: "error" or "drop"
```

```console
anyctr config path | show | get KEY | set KEY VALUE | unset KEY
```

## Troubleshooting

**Apple container: "system services are not running."** Start them once per boot:
`container system start`.

**Apple container: published ports unreachable on localhost.** `-p` is accepted and
recorded, but host port forwarding depends on the runtime's networking being active.
Every container also gets its own routable IP — `anyctr ps` shows it, and it works
even when forwarding does not.

**A flag anyctr doesn't know.** Unmodelled flags are rejected rather than guessed at;
the error points you to `anyctr raw`. Please also open an issue — a missing mapping
is a bug.

## Roadmap

- `containerd`/`nerdctl` adapter
- A built-in Compose engine, so `compose up` needs no external provider
- `inspect`, `cp`, `network`, `volume` commands

## Development

```console
make check              # go vet + unit tests + gofmt    (no runtime needed)
make verify-caps        # capability matrix vs. each installed runtime's --help
make test-integration   # real containers on the detected runtime
make cross              # build for every supported platform
make docs               # regenerate docs/CAPABILITIES.md from the code
```

Tests come in three tiers, because no single machine can run all four runtimes:

| tier | needs | proves |
| --- | --- | --- |
| unit / translation | nothing | the argv anyctr produces is what we intend |
| conformance | the runtime's CLI | the matrix matches that CLI's real `--help` surface |
| integration | a working runtime | the runtime accepts and executes that argv |

The translation layer never executes anything, so tier 1 covers every runtime on
every OS — `--runtime=container` translations are verified on Linux CI with no Mac
in sight. Conformance only reads `--help`, so it needs the CLI but not a running
engine, and it fails in *both* directions: a flag anyctr refuses that the runtime
has since gained is as much a bug as one it claims and the runtime dropped.

CI runs tiers 1–3 for docker and podman on `ubuntu-latest` (both preinstalled) and
tier 1 on macOS and Windows. Apple's `container` cannot run on GitHub-hosted macOS
runners — they are M1 VMs without nested virtualization, which
[the runner-images maintainers confirmed](https://github.com/actions/runner-images/issues/13565)
— so its integration tier needs a self-hosted Apple-silicon runner or a developer
Mac. Its conformance tier may work on hosted runners since it starts no VM; the
`conformance-apple` job is wired up behind `workflow_dispatch` to find out.

### Releasing

Push a tag and the rest is automatic:

```console
git tag v0.1.0 && git push origin v0.1.0
```

GoReleaser builds all six targets, publishes a GitHub release with checksums, and
updates the Homebrew cask in `pedrospdc/homebrew-tap`. The cask step needs a
`HOMEBREW_TAP_DEPLOY_KEY` secret on this repo — an SSH deploy key with write
access to the tap. Without it the release still publishes; only the tap update
fails.

`goreleaser check` runs in CI, so a broken release config fails on the pull
request that breaks it rather than on the tag. To rehearse a release locally:

```console
go run github.com/goreleaser/goreleaser/v2@latest release --snapshot --clean --skip=publish
```

[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) covers the design and how to add a
runtime or a command.

## License

MIT — see [LICENSE](LICENSE).
