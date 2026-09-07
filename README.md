# create-minecraft-server

A [Claude Code](https://claude.com/claude-code) skill for standing up a Minecraft server end-to-end on Hetzner Cloud with [OMCSI](https://github.com/Stephenson-Software/open-mc-server-infrastructure).

It covers the whole path: gathering the decisions, creating a private config repository and an ops repository, provisioning a single-node Kubernetes cluster with Terraform, installing plugins, locking the server down behind a whitelist, and verifying it against the running system.

## Usage

```
/create-minecraft-server
```

## Why it exists

Most of this skill is not the happy path — that part is a `terraform apply`. It is the set of things that are stated somewhere and are not true: documented prices that are out by a factor of two, an availability endpoint that lists server types the API then refuses, a first-boot time that is off by an order of magnitude, and plugins that enable cleanly and then fail on every event.

Its central instruction is to verify against the live system rather than against documentation — including its own.

## Self-audit

The skill carries a self-audit section. Run it when the environment has moved (OMCSI defaults, Hetzner pricing or availability, published image tags) and it will check itself and file issues here.
