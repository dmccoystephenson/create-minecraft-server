# create-minecraft-server

Stand up a new Minecraft server end-to-end on Hetzner Cloud using [OMCSI](https://github.com/Stephenson-Software/open-mc-server-infrastructure): gather the decisions, create the config and ops repositories, provision a single-node Kubernetes cluster with Terraform, install plugins, lock the server down, and verify it live.

Use this when someone asks for a new Minecraft server, a second server alongside an existing one, or a rebuild of one that was lost.

---

## The one rule that matters most

**Verify against the live system, never against documentation — including OMCSI's own, and including this file.**

Every serious problem in the session this skill came from was a case of trusting a stated fact instead of checking:

| Stated | Actual |
|---|---|
| `cax31` costs ~EUR 12.49/mo (OMCSI docs at the time; since corrected) | EUR 24.99 — the quoted figure is `cax21`'s price |
| Hetzner's API lists CAX types as `available` in a location | Every CAX create is refused there |
| "First boot takes 10–15 min for a BuildTools compile" | ~90 seconds; BuildTools runs at image-build time |
| A plugin "enables cleanly", so it works | One threw on every block break and buried the logs |

Prices, capacity, plugin compatibility, and upstream defaults all move. Check them at the time you act.

---

## Steps

### 1 — Gather the decisions

Ask, and do not guess:

1. **Name** — becomes the GitHub org or repo prefix, the Hetzner server name, and the MOTD. Check availability: `curl -s -o /dev/null -w '%{http_code}' https://github.com/<candidate>` (404 = free).
2. **Where the repos live** — a new GitHub org, or an existing one.
3. **Who plays, and where they are** — drives the region. Hetzner's EU locations are `fsn1`/`nbg1`/`hel1`; US are `ash`/`hil`; also `sin`. Latency matters more than a couple of euros.
4. **How big** — see Step 2 for real prices. A friends-and-family server of under ~10 players runs comfortably on 8 GB.
5. **Plugins** — none, a small quality-of-life set, or a full stack.
6. **Minecraft version** — "latest" needs the check in Step 4.

State a recommendation with each question rather than presenting a bare menu.

---

### 2 — Establish Hetzner ground truth *before* promising anything

This is the step that prevents a failed apply and a wrong quote. It needs an API token (Step 3), so do it as soon as one exists — and redo it if the plan sits unapplied for long.

**Prices, from the API and not from any document:**

```bash
curl -s -H "Authorization: Bearer $HCLOUD_TOKEN" \
  'https://api.hetzner.cloud/v1/server_types?per_page=100' \
| python3 -c "
import sys,json
for s in json.load(sys.stdin)['server_types']:
    for p in s['prices']:
        print(f\"{s['name']:8} {s['architecture']:4} {s['cores']:>2}vCPU {int(s['memory']):>3}GB {s['disk']:>4}GB {p['location']:5} EUR {float(p['price_monthly']['net']):6.2f}\")
" | sort -k11 -n
```

**Availability — and do not trust it.** `/v1/datacenters` exposes `server_types.available`, and it can list types that every create then refuses. Treat it as a shortlist, then **probe a real create** for each candidate and delete it immediately. A server billed for seconds costs a fraction of a cent, and it is the only reliable answer:

```bash
probe() {  # probe <location> <type>
  body=$(curl -s -H "Authorization: Bearer $HCLOUD_TOKEN" -X POST https://api.hetzner.cloud/v1/servers \
    -H 'Content-Type: application/json' \
    -d "{\"name\":\"probe-$2-$1\",\"server_type\":\"$2\",\"image\":\"ubuntu-24.04\",\"location\":\"$1\",\"start_after_create\":false}")
  id=$(echo "$body" | python3 -c "import sys,json;print((json.load(sys.stdin).get('server') or {}).get('id',''))" 2>/dev/null)
  if [ -n "$id" ]; then
    echo "  $2@$1 -> ORDERABLE (deleting)"
    curl -s -H "Authorization: Bearer $HCLOUD_TOKEN" -X DELETE "https://api.hetzner.cloud/v1/servers/$id" -o /dev/null
  else
    echo "  $2@$1 -> $(echo "$body" | python3 -c "import sys,json;print(json.load(sys.stdin).get('error',{}).get('message'))" 2>/dev/null)"
  fi
}
```

Read the two failure messages differently:

- **`unsupported location for server type`** — the project cannot order that type there at all. Observed for *every* ARM (CAX) type across every EU location on a fresh project, while x86 succeeded in the same locations. Do not assume ARM is available; probe it.
- **`error during placement`** — capacity. Try another location; it may come back later.

Only after this do you know what to quote and what to put in `TF_VAR_server_type` / `TF_VAR_location`.

> **ARM matters beyond price.** OMCSI's default is an ARM type, and its images are multi-arch. If ARM turns out to be unorderable, an x86 type of the same size is usually available and sometimes cheaper — but re-derive the price rather than assuming a mapping.

---

### 3 — What the user must do (you cannot)

Both need a human; ask early so they are not the blocker at the end.

1. **Create the GitHub organisation**, if a new one is wanted. github.com has no org-creation API — it is web-UI only, at `github.com/organizations/new` — and a normal token lacks `admin:org` regardless. Once it exists, confirm with `gh api orgs/<org> --jq .login` and check the role is `admin`.
2. **Create a Hetzner Cloud project and a Read & Write API token.**

**Give each server its own Hetzner project.** Tokens are scoped to one project (there is no account-wide Cloud token, and no `/v1/projects` endpoint), so a per-server project is what keeps a leaked token from reaching another server. Reusing an existing token puts the new server in that project and puts a credential controlling the old server into the new server's config repo.

Verify the token before using it:

```bash
# is the project empty (i.e. actually new)?
curl -s -H "Authorization: Bearer $HCLOUD_TOKEN" https://api.hetzner.cloud/v1/servers \
  | python3 -c "import sys,json;print('servers:',json.load(sys.stdin)['meta']['pagination']['total_entries'])"

# is it read-WRITE? a read-only token 403s before validation; read-write reaches 422
curl -s -H "Authorization: Bearer $HCLOUD_TOKEN" -X POST https://api.hetzner.cloud/v1/ssh_keys \
  -H 'Content-Type: application/json' -d '{}' -o /dev/null -w '%{http_code}\n'
```

`422` means write access and creates nothing. `403` means the token is read-only and the apply will fail late.

---

### 4 — Align the Minecraft version with the image

**The runtime `MINECRAFT_VERSION` only selects a jar that is already baked into the image.** OMCSI's `Dockerfile` runs BuildTools at *build* time via `ARG MINECRAFT_VERSION`, and `resources/post-create.sh` copies `spigot-${MINECRAFT_VERSION}.jar` out of the image under `set -euo pipefail`. A mismatch exits the container; it does not fall back.

**OMCSI builds Minecraft 1.17 and later only.** BuildTools and the server each accept only a range of JDKs, so `resources/java-for-minecraft.sh` picks one per version for both the build and the run: 17 up to 1.20.4, 21 for 1.20.5–1.21.x, 25 from 26.x. Anything older needs Java 8–16, which the image does not carry, so the build refuses it rather than trying.

```bash
# what the latest Minecraft release is
curl -s https://launchermeta.mojang.com/mc/game/version_manifest.json \
  | python3 -c "import sys,json;print(json.load(sys.stdin)['latest']['release'])"
# whether Spigot BuildTools has that rev
curl -s https://hub.spigotmc.org/versions/ | grep -o '[0-9.]*\.json' | sort -V | tail -5
# what images actually exist
curl -s 'https://hub.docker.com/v2/repositories/dmccoystephenson/open-mc-server/tags?page_size=25' \
  | python3 -c "import sys,json;print([t['name'] for t in json.load(sys.stdin)['results']])"
```

If the wanted version has no published image, it needs an upstream bump to the `Dockerfile` `ARG` (CI then republishes) or a `workflow_dispatch` of the publish workflow. Budget for it: BuildTools compiles Spigot from source, and the arm64 leg runs under emulation.

**Two image tags, not one.** CI publishes a version tag for `open-mc-server` **only**; the supporting images (`webapp`, `nginx`, `alert-manager`, `backup-manager`, `agent-manager`) are pushed as `latest` and as the SHA of the commit that built them — never a version. Set `TF_VAR_image_tag` to the Minecraft version and `TF_VAR_supporting_image_tag` to `latest` or a commit SHA — pinning the supporting images to a version tag yields `ImagePullBackOff`. A SHA tag exists only for commits that rebuilt the images, since the publish workflow is path-filtered, so confirm it on Docker Hub before pinning it.

**Check plugin compatibility separately**, and be honest that you cannot fully. A plugin's `plugin.yml` `api-version` is a compatibility *floor*, not a claim of support. That a plugin loads is not evidence it works — see the trap in Step 9.

---

### 5 — Create the config repository

One **private** repo holding the environment as a committed `.env`. This is deliberate: it is the single canonical copy every machine and every session sources, instead of each carrying a drifting one.

```bash
gh repo create <org>/<name>-config --private --description "..."
```

It must contain:

- **`.env`, committed**, every variable `export`ed so a plain `source` works. A `.gitignore` that pointedly does *not* ignore `.env`, with a comment saying why, so nobody "fixes" it later.
- **`KUBECONFIG` resolved relative to the file** — derive the repo directory from `${BASH_SOURCE[0]}` so any clone on any machine works without editing a path. The kubeconfig does not exist until after the first apply; warn to stderr rather than `exit`, since a hard failure inside a sourced file kills an interactive shell.
- **A README that states the bargain plainly**: this repo holds live credentials in plaintext on purpose; anyone with read access controls the Hetzner account, the cluster, RCON, and the dashboard; values live in git history forever; the kubeconfig is a cluster-admin certificate that **cannot be revoked**, because Kubernetes has no CRL. Keep it private, never fork it to a public namespace.
- **Comments that explain the non-obvious values** — especially any figure that was measured rather than assumed, and any setting whose default is wrong for this server. Record *why*, so a future reader does not tidy it away.
- **`TF_VAR_allowed_ssh_cidr` as a required value, not a default.** OMCSI defaults it to `0.0.0.0/0`, which (with `allowed_api_cidrs` left empty) puts SSH *and* the cluster-admin API on the open internet. Set it to a `/32`. Note in a comment that it is the provisioning machine's egress IP, may not be stable, and that a timeout on 22 or 6443 is most likely this rule (or, for 6443, `allowed_api_cidrs` if set) — fixable via the Hetzner API without touching the server.
- **`TF_VAR_allowed_api_cidrs` only if the API needs a different audience than SSH.** It is a list that governs 6443 alone, and OMCSI defaults it to empty, which **falls back to `allowed_ssh_cidr`**. Leave it unset for a single operator. Set it to give a monitoring host or a second operator `kubectl` access without also handing them a shell on the node. If you set it, record in a comment that 6443 no longer follows the SSH value.
- **`TF_VAR_rbac_enabled` if anyone but the author will run `kubectl`.** It creates namespace-scoped ServiceAccounts, so day-to-day access need not use the non-revocable cluster-admin kubeconfig. It is off by default because enabling it mints credentials. The extra `TF_VAR_rbac_operator_enabled` account is namespace-admin and can still read the Secrets holding the RCON and admin passwords, so it is a reduction from cluster-admin, not a low-privilege credential.

Generate secrets rather than inventing them (`secrets.choice` over `string.ascii_letters+string.digits`, 32+ chars). Never print a secret into the transcript, a commit message, or a log.

---

### 6 — Create the ops repository (optional but recommended)

A second **private** repo for documentation and a Claude Code ops skill. Worth it when someone other than the author will operate the server, or when future sessions should be able to drive it.

Keep it proportionate — a small server does not need a community server's staff handbook. What earns its place:

- a deployment runbook, written so the apply is a followable checklist;
- a live spec page: node, versions, ports, what is public and what is cluster-internal;
- one short page per plugin covering *this server's* configuration, not upstream documentation;
- an ops skill that sources the config repo and never echoes a credential.

**Write it against the running server, not against intent.** Pages written ahead of deployment go stale in ways that actively mislead — Step 10 is the correction pass, and it is not optional.

---

### 7 — Provision

**Use a clone of OMCSI dedicated to this server.** Terraform keeps state in the directory it runs from, so a shared checkout mixes two servers' state.

```bash
git clone https://github.com/Stephenson-Software/open-mc-server-infrastructure ~/<name>-omcsi
cd ~/<name>-omcsi/terraform/hetzner
source ~/<org>/<name>-config/.env
terraform init
terraform plan -out=plan.tfplan
```

Read the plan before applying:

- **5 to add** on a first provision — `hcloud_ssh_key`, `hcloud_firewall`, `hcloud_server`, and two `null_resource`s. Anything to *destroy* means state already exists and this is not a first provision. Stop.
- **Check the firewall rules carry the intended CIDR**, from the plan JSON rather than the human-readable output, which elides list contents:

```bash
terraform show -json plan.tfplan | python3 -c "
import sys,json
for rc in json.load(sys.stdin)['resource_changes']:
    if rc['type']=='hcloud_firewall':
        for r in rc['change']['after']['rule']:
            print(f\"  {r['description']:22} {r['protocol']}/{r['port']:<6} from {r['source_ips']}\")
"
```

SSH (22) must be the `/32`. The Kubernetes API (6443) must be exactly `TF_VAR_allowed_api_cidrs` if it is set, and otherwise the same `/32`, because OMCSI falls back to `allowed_ssh_cidr` when the list is empty. Minecraft (25565) and the dashboard (80/443) are public by design.

Then `terraform apply plan.tfplan`. It blocks through server creation, the cloud-init kubeadm bootstrap, and the Helm install — several minutes. **Run it in the background and watch the log**, rather than in a foreground call that may time out.

Afterwards, copy the generated `kubeconfig.yaml` into the config repo and commit it.

> **The Terraform state is the only record that the server exists**, and OMCSI's local-backend default puts it in one directory on one machine with no backup. Say so in the ops docs, and consider where it should really live.

---

### 8 — Bring the server into service

1. **Op the operator.** Resolve the UUID from Mojang rather than typing it — an entry whose UUID does not match the account ops nobody and fails silently:
   `curl -s https://api.mojang.com/users/profiles/minecraft/<username>`
   Set `TF_VAR_operator_uuid` / `TF_VAR_operator_name` so a rebuild reproduces it. `setup_ops_file` preserves an existing `ops.json`, so runtime `op` changes are safe.

2. **Populate the whitelist.** If `TF_VAR_whitelist_enabled` is set, the server comes up whitelisted and **empty — admitting nobody, including the operator.** That is the correct starting state; do not "fix" it by turning the whitelist off, which briefly opens the server to the internet. Add players over RCON (`kubectl port-forward` to the internal service, then a small RCON client): `whitelist add <name>`. No restart needed.

   **Nothing reproduces the whitelist on a rebuild.** OMCSI never writes `whitelist.json`. Keep a members table with UUIDs in the ops repo — it is the recovery list.

3. **Confirm the plugins installed.** `TF_VAR_default_plugins` runs **only at first server setup**. Anything added later needs the hot-deploy endpoint *and* an entry in the list for next time — forgetting the second half is how a server ends up with a route and no plugin behind it.

4. **Per-plugin post-install steps** have no variables behind them. Two seen in practice: BlueMap renders nothing until `accept-download: true` is set in its `core.conf` (until then `/map/` is a 502); Herald installs inert and reads its config only at plugin enable, so it needs config written *and* a restart.

5. **Restart only with nobody online.** A restart drops every connected player, and a plugin that saves only in `onDisable` rolls back to its last save.

---

### 9 — Verify, mostly without a Minecraft client

Almost everything can be checked from outside. Do it — "the pods are running" is not evidence the server works.

- **Server-list ping on 25565.** The strongest single check: it proves Spigot is serving publicly and returns the version, MOTD and player count. Implement the handshake directly (a few dozen lines of Python) rather than adding a dependency. Note the MOTD may arrive as `{"text":"","extra":["..."]}` — read `extra`, not just `text`.
- **`kubectl get pods -n omcsi`** — all `1/1`, restart counts at zero.
- **Whitelist across a restart.** `white-list=true` must survive a *container* restart, since that is what regenerates `server.properties` — which also means hand-edits to that file do not survive one. Restarting Spigot through the wrapper API does not exercise the same path.
- **Every plugin enabled**: `kubectl logs … | grep -oE "Enabling [A-Za-z]+ v[0-9.]+" | sort -u`.
- **Dashboard** returns 302; **BlueMap** `/map/` returns 200 and serves the webapp.
- **Alerts reached Discord**: `kubectl logs deploy/…-alert-manager | grep Discord`.

> **Loading is not working.** A plugin can enable cleanly on a new Minecraft version and then throw on every event. One shaded a version-mapping library too old for the server version and raised an exception on every block interact, break and place — gameplay survived because Bukkit catches listener exceptions, but each failure printed a ~40-line stack trace. At ~1 MB/min the container log rotated so fast that only **about a minute** of history was retained: joins, plugin notifications and unrelated errors were gone before anyone could read them.
>
> So: after players have been on, **check the log for exceptions**, not just for startup lines. `grep -c "Could not pass event"` is a good first probe. A server whose logs rotate in a minute is effectively un-debuggable, and that is worth fixing before anything else.

State plainly what you did *not* verify. Whether plugins behave correctly in play needs real players.

---

### 10 — Correct the documentation against reality

Anything written before the server existed is now a hypothesis. Re-read every page against the running system and fix what differs — this is where most of the value lands, and it is the step most likely to be skipped.

Check specifically:

- machine type, location, price, IP, versions — all observable now;
- timings that were estimated;
- any "known constraint" or "blocked on" note, which may have been resolved;
- counts (plugins, resources) that have drifted;
- statements like "nothing here has been observed", which stop being true the moment it is;
- **internal links and anchors.** If headings changed, links break. GitHub's slug rule is: lowercase, strip punctuation, replace each space with `-` — it does **not** collapse runs, so `A — B` becomes `a--b` with two hyphens.

Record *why* a value is what it is, especially when it contradicts upstream defaults. A bare `server_type = cx33` invites someone to "restore" the documented default that does not work.

---

### 11 — Hand over

Report: the address, what was verified and how, what was **not** verified, the monthly cost, anything left for the user, and any upstream bug filed. If a Discord webhook is available, post progress there as you go — a long provision is otherwise silent.

When posting issues or PRs under the user's GitHub identity, follow their conventions: passive voice, attribute the work to Claude, and sign off. **Never name a private repository in a public issue** — check with `gh api repos/<owner>/<repo> -q .private` before writing a body, since edit history preserves the leak.

---

## Traps, collected

| Trap | Consequence |
|---|---|
| Trusting `/v1/datacenters` `available` | Apply fails with `unsupported location for server type` |
| Trusting documented prices | Quote can be out by 2× |
| `MINECRAFT_VERSION` ≠ image tag | Container exits on start, no fallback |
| One image tag for all services | `ImagePullBackOff` on everything but the Minecraft image |
| `allowed_ssh_cidr` left at default | SSH open to the internet, and the cluster-admin API too unless `allowed_api_cidrs` is set |
| `TF_VAR_default_plugins` after first setup | Silently does nothing |
| Whitelist enabled, list empty | Nobody can join, including the operator |
| Rebuild onto an empty volume | Whitelist gone; server up, reachable, admitting nobody |
| `server.properties` hand-edits | Regenerated on every container start |
| "It enables, so it works" | Runtime failure floods the log and destroys history |
| Restarting while players are online | Drops everyone; also rolls back plugins that save only in `onDisable` |

---

## Self-audit

Run this section when the skill may have drifted from reality — e.g. after the target environment changed, commands started failing, or results are consistently wrong.

1. Read this skill file from top to bottom.
2. For each command, path, or assumption, verify it is still correct:
   - Commands and flags still exist
   - File paths are valid
   - Environment assumptions still hold
3. Pay particular attention to the things this skill *knows* are moving targets:
   - OMCSI's Terraform variables, defaults, and which settings the Hetzner module templates
   - Which container image tags CI publishes
   - Hetzner server types, prices, and per-project availability
   - Whether the Mojang, Spigot, Docker Hub and Modrinth endpoints quoted here still return what is claimed
4. For each problem found, open a GitHub issue:
   ```bash
gh issue create --repo dmccoystephenson/create-minecraft-server \
  --title "<problem summary>" \
  --body "$(cat <<'EOF'
**Section:** <which step or section is wrong>
**Problem:** <what is incorrect>
**Expected behavior:** <what it should do instead>
EOF
)"
   ```
5. Report a summary: how many issues were filed, or confirm the skill is up to date.
