# Quickstart

Five minutes to a running agent container that can reach exactly one host
and nothing else.

> **Linux only.** `outcalld` manages a kernel network bridge and applies
> nftables rules — both require Linux. macOS hosts can build the workspace
> and run the CLI, but the daemon will not start.

## 1. Start the daemon

```sh
docker run -d --rm \
  --name outcall-daemon \
  --network host \
  --cap-add NET_ADMIN \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /run/outcall:/run/outcall \
  -v /etc/outcall:/etc/outcall \
  outcall-e2e \
  outcalld --bridge outcall0
```

Verify the bridge is up and nftables rules are active:

```sh
$ outcall bridge status
Bridge:    outcall0
Status:    up
Index:     12
nftables:  active
```

## 2. Drop in a rule

Default-block is implicit — write only the things the agent may do:

```sh
sudo mkdir -p /etc/outcall/rules.d
sudo tee /etc/outcall/rules.d/agent.yaml > /dev/null <<'EOF'
version: "1"
rules:
  - id: allow-openai
    description: "agent may call the OpenAI API"
    condition: 'http.host == "api.openai.com"'
    action: allow
    egress:
      mode: proxy
EOF
```

Reload the rules:

```sh
outcall rules reload
```

Or, equivalently, POST to the host API directly:

```sh
sudo curl -fsS --unix-socket /run/outcall/host.sock \
  -X POST http://localhost/api/v1/rules/reload | jq .
```

The response shows `files_loaded`, `rules_loaded`, and any warnings. If a
rule fails validation, the old set stays active and the error is in the
response body.

## 3. Create the agent network

```sh
$ outcall network create --name agent-net
Network "outcall-agent-net" created (10.200.0.0/24).
```

Names are *suffixes*: `--name agent-net` produces a Docker network
called `outcall-agent-net`. The gateway is the `.1` of the chosen `/24`
(here, `10.200.0.1`) — that's also the address of the DNS filter and HTTP
proxy.

## 4. Run the agent

```sh
docker run -it --rm \
  --network outcall-agent-net \
  --dns 10.200.0.1 \
  -e HTTP_PROXY=http://10.200.0.1:8080 \
  -e HTTPS_PROXY=http://10.200.0.1:8080 \
  -v /run/outcall/agent.sock:/run/outcall/agent.sock \
  -v /usr/local/bin/outcall-agent:/usr/local/bin/outcall-agent:ro \
  python:3.12 \
  bash
```

(For the all-in-one path, `outcall container create --image python:3.12
--network agent-net` will wire up DNS, proxy, and shim mounts for you.)

Inside the container, prove enforcement at every layer:

```sh
# Allowed
curl -s -o /dev/null -w "%{http_code}\n" https://api.openai.com/v1/models
# 401 (the request reached OpenAI, OpenAI rejected the empty auth)

# Blocked at DNS
curl -s -o /dev/null -w "%{http_code}\n" https://example.com
# curl: Could not resolve host: example.com  (DNS filter returned NXDOMAIN)

# Blocked at L7 (when DNS happens to resolve from cache)
curl --resolve example.com:443:93.184.216.34 -s -o /dev/null -w "%{http_code}\n" \
     https://example.com
# 403  (HTTP proxy rejected the SNI before opening the upstream tunnel)

# Blocked at L3/L4 (no proxy in the way — direct IP traffic)
nc -zv 1.1.1.1 443
# nc: connect to 1.1.1.1 port 443 (tcp) failed: Connection timed out
```

## 5. Inspect what happened

```sh
outcall dns cache --entries           # which hostnames the filter has seen
outcall network status --name agent-net
outcall bridge status                 # nftables active, bridge up
outcall proxy status                  # active connections, totals, blocks
```

To inspect rules, query the host API:

```sh
curl --unix-socket /run/outcall/host.sock http://localhost/api/v1/rules | jq .
```

## What just happened

| Step | Layer | Effect |
|---|---|---|
| 1 | host | `outcalld` brought up the bridge and a default-block nftables table. |
| 2 | rules | Your YAML compiled into a CEL expression and a verdict. |
| 3 | docker | A network attached to the bridge; gateway hosts DNS and proxy. |
| 4 | container | DNS, HTTP proxy, and agent shim wired up. Default-deny is in force. |
| 5 | egress | Each blocked request was rejected at the *highest* layer that saw it. |

## Where to go next

- [Writing rules](/docs/guides/rules) — every matcher, every action.
- [Configuration](/docs/guides/configuration) — every daemon flag.
- [CLI reference](/docs/guides/cli) — every `outcall` subcommand.
- [Troubleshooting](/docs/guides/troubleshooting) — diagnosing the most common failures.
