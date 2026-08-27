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

## Authenticate Claude or Codex

Outcall distinguishes provider configuration from a portable credential. Codex
can normally copy `~/.codex/auth.json`. Claude on Linux can copy
`~/.claude/.credentials.json`, but a macOS Claude `/login` stays in Keychain and
cannot authenticate a Linux container.

For Claude on macOS, either start it interactively once:

```sh
outcall run claude
# Complete /login inside the managed container.
```

That writes a Linux credential into the ignored, owner-only
`.outcall/home/claude/` directory. Later `claude -p` and detached runs reuse it.
For unattended subscription use, run `claude setup-token` on the host and export
`CLAUDE_CODE_OAUTH_TOKEN`. API users can export `ANTHROPIC_API_KEY` or
`ANTHROPIC_AUTH_TOKEN`.

Default `copy` mode copies only portable credential files into
`.outcall/home/<recipe>/`. `--include-global-config` opts into bounded selected
settings, instructions, agents, commands, or hooks; review host-only MCP and
hook paths because macOS executables do not run in Linux. `mount` is an explicit
read-write opt-in for the complete provider directory (`~/.claude` or
`~/.codex`), never the complete host home. Batch and detached runs fail before
building when no portable credential is available.

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

`outcall stop` removes the stopped agent so the name can be reused. Add
`--keep` only when you need postmortem logs or inspection.

Without `--name`, containers are named from the project directory: `foobar-1`,
`foobar-2`, and so on. The project workspace is mounted into the container;
files outside it are unavailable unless you deliberately declare and expose a
host resource.
