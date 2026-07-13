# First Run

Outcall is intended to have one normal path: install it, choose an agent, and
let the CLI prepare an isolated project container.

```sh
curl -fsSL https://outcall.dev/install.sh | sh
cd /path/to/project
~/.local/bin/outcall run codex
```

The absolute path works immediately after installation. Add `~/.local/bin` to
your shell `PATH` once you are ready to use the shorter `outcall` command.

On macOS, install and start Docker Desktop first. Outcall runs the daemon and
agent containers in Docker Desktop's Linux runtime. On Linux, it uses the
native Docker runtime.

## Pick authentication explicitly

Outcall detects a provider API key or the provider's normal local login files.
Review and stage only the selected provider material before launching:

```sh
outcall auth codex
outcall auth claude --auth env-only
```

`copy` stores only selected provider files under `.outcall/auth/`, which is
ignored by Git and created with owner-only permissions. Env-only runs receive a
separate writable `.outcall/home/<recipe>/` directory, also ignored by Git,
without copying provider credentials. `mount` keeps the files on the host and
mounts only the selected paths. For unattended use, prefer the provider's API
key in the environment; Outcall never copies the whole home directory or reads
browser/keychain session state.

## Repair prerequisites

Use the explicit repair command when Docker is not ready or a project has not
been initialized:

```sh
outcall doctor --fix codex
```

It may open Docker Desktop on macOS, wait for Docker, pull a missing verified
daemon image, write missing project scaffolding, and create the managed daemon
and network. It only performs those changes when `--fix` is supplied.

## Add only the access the agent needs

Every project is default-deny. Recipes supply named grants for their normal
provider endpoints and GitHub. The convenience commands edit the ordinary
project YAML rule file; there is no hidden policy store.

```sh
outcall allow codex github
outcall allow codex https://api.sentry.io
outcall policy explain codex
```

The commands update `.outcall/rules/codex.yaml`, retain existing rules, and
reload the daemon when it is available. Anything not listed remains blocked.

## Run and manage agents

```sh
outcall run codex
outcall run codex --name review-1 --detach
outcall ps
outcall logs review-1 --follow
outcall stop review-1
```

Without `--name`, containers are named from the project directory: `foobar-1`,
`foobar-2`, and so on. The project workspace is mounted into the container;
files outside it are unavailable unless you deliberately declare and expose a
host resource.
