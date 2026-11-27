---
name: appliku
description: Use when working with the appliku CLI or SDK to manage Appliku-hosted applications, deployments, domains, datastores, servers, teams, or SSH keys. Covers authentication, all CLI commands, Python SDK usage, and common deployment workflows.
user-invocable: false
---

# Appliku CLI & SDK Reference

Appliku is a PaaS platform. The `appliku` package provides both a CLI and a Python SDK.

## Installation

### 1. Check for uv

```bash
uv --version
```

If the command is not found, install uv first:

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows PowerShell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

After installing, restart the shell (or source the profile) so `uv` is on `$PATH`.

### 2. Check for the appliku CLI

```bash
appliku --version
```

If not found, install it as a uv tool:

```bash
uv tool install appliku
```

Then verify:

```bash
appliku --help
```

### SDK (inside a Python project)

```bash
uv add appliku
# or
pip install appliku
```

Requires Python 3.10+.

## Authentication

Token resolution order (first match wins):

1. Explicit: `Appliku(token="...")`
2. Environment variable: `APPLIKU_TOKEN=...`
3. Config file: `~/.config/appliku/config.toml` → `[auth] token = "..."`

### For non-interactive / CI environments

```bash
export APPLIKU_TOKEN=your_api_token
```

### Interactive login (browser device flow)

```bash
appliku login              # opens browser to authorize
appliku login --token TOKEN  # direct token login (non-interactive)
appliku logout
appliku whoami
```

## Key Concepts

- **team_path**: a slug identifying your team (e.g. `my-team`). Required by most commands.
- **app id**: numeric ID of an application. Get it from `appliku apps list --team my-team --output json`.
- **resource id**: numeric ID for domains, datastores, volumes, crons, etc.
- Most list commands support `--output json` for machine-readable output.

## CLI Command Reference

### teams

```bash
appliku teams list
appliku teams get <team_path>
```

`teams get` returns the same fields whatever your membership level, but values you
are not entitled to read come back as `null`. Admins see `null` for the billing
fields (`paddle_*`), `can_invite_members` and `name_picked`; developers
additionally see `null` for the deploy keys (`public_key`, `public_key_ed25519`)
and the notification webhook URLs. So a `null` means either "not set" or "not
visible to you" — for billing values, check as an owner before concluding a team
has no subscription. Requires CLI 0.5.0 or newer; older versions crashed for any
non-owner.

### apps

```bash
appliku apps list --team <team_path>
appliku apps list --team <team_path> --output json

# Create from GitHub or GitLab. Use exactly one positive target ID.
appliku apps create github owner/repository -t <team_path> -n myapp -b main --server <id>
appliku apps create gitlab group/repository -t <team_path> -n myapp -b main --cluster <id>

# Create from custom Git. A URL file can contain HTTPS credentials.
appliku apps create custom -t <team_path> -n myapp -b main --server <id> \
  --git-url-file <path-or-> [--private-key-file <path>]

# Trigger a deployment
appliku apps deploy --team <team_path> --app <id>

# Restart WITHOUT a build: same running image, current env vars and process
# settings. No new code, no build, no release command. Build-time variables
# (NEXT_PUBLIC_*, anything the build command reads) keep their old values ->
# use `apps deploy` for those. Needs the backend with the restart endpoint; the
# first restart needs one full deploy made with it. Not for static sites.
appliku apps restart --team <team_path> --app <id>
appliku apps restart --team <team_path> --app <id> --check   # possible? exit 1 + reason if not

# Config vars: see the `config` group below. `apps delete-config-var` is
# DEPRECATED — use `appliku config unset KEY` instead.

# Application logs — one command, any deployment mode, one or more processes.
# With no --process it returns logs for ALL of the app's processes.
appliku apps logs --team <team_path> --app <id>
appliku apps logs --team <team_path> --app <id> --process web
appliku apps logs --team <team_path> --app <id> --process web --process celery --tail 200

# service-logs is DEPRECATED — use `apps logs -p <service>` instead.

# Nginx / load balancer logs
appliku apps nginx-logs         --team <team_path> --app <id> --domain example.com --tail 100
appliku apps load-balancer-logs --team <team_path> --app <id> --domain example.com --tail 100
```

Each create command accepts `--static-site` and `--output table|json`. Custom
Git also accepts a positional public URL, but exactly one positional URL or
`--git-url-file` is required. Use a file or standard input (`--git-url-file -`)
for a URL that contains credentials. Private keys are accepted only through
`--private-key-file`. Output is limited to safe application fields and does not
contain these secrets.

Create starts the asynchronous initial `appliku.yml` inspection. It does not
deploy the application. Deploy it separately when its setup is complete.

**The target must be ready.** A server or cluster that has not finished setup is
refused with HTTP 400. Both list commands report a `ready` column, so pick the
target from there rather than guessing an ID:

```bash
appliku servers list  --team <team_path>                 # Ready column
appliku clusters list --team <team_path>                 # Ready column
appliku servers list  --team <team_path> --output json | jq '[.[] | select(.ready)]'
```

A server is ready when it is active and its setup finished. A cluster is ready
when its primary manager is — a cluster with no primary manager can never host
an application. If nothing is ready, wait for setup to finish; retrying the
create immediately just returns the same 400.

### apps move

Move a server-mode application (with its standalone datastores and volumes) to
another server of the team. Team owners and admins only; `targets`, `plan` and
`confirm` answer 403 while moves are not enabled for the team. Needs CLI 0.10.0
(the move to a cluster, `--accept-rebuild` and `recover-without-stop-proof`
need CLI 0.11.0; `--dump-restore` needs CLI 0.12.0).

```bash
appliku apps move targets --team <team_path> --app <id>          # "ok" or the first refusal per server
appliku apps move plan    --team <team_path> --app <id> --to <server id or name> --wait \
  [--dump-restore <datastore id> ...]                            # Postgres/MySQL: dump and restore
appliku apps move confirm --team <team_path> --app <id> --name <app name> \
  [--accept-unmoved <key> ...] [--accept-rebuild]                # starts the move: downtime
appliku apps move status  --team <team_path> --app <id> --follow # until hold, end, or it waits for you
appliku apps move finish  --team <team_path> --app <id> --move <move_id>   # end the hold; no rollback after this
appliku apps move rollback --team <team_path> --app <id> --move <move_id> --accept-data-loss [--accept-new-domains]
# A server of the move is gone or does not answer (the move page lists the kept copies):
appliku apps move abandon          --team <team_path> --app <id> --move <move_id>  # end the move; no data is deleted
appliku apps move start-old-copy   --team <team_path> --app <id> --move <move_id>  # the kept copy runs as the app again
appliku apps move remove-kept-copy --team <team_path> --app <id> --move <move_id> [--name <app name>]  # cannot be undone
# also: list, cancel, retry, confirm-downtime, remove-forwarders, cleanup-now
# (each takes --team and --app; all but list take --move <move_id>; the prompts take --yes)
```

- The plan only reads both servers; nothing changes until `confirm`. A plan
  expires after 24 hours. Blockers in the plan stop the move; read them.
- `confirm` starts the move and the downtime. Show the plan (downtime,
  warnings, unmoved mounts) to the user and run `confirm` only after the user
  chose to start the move.
- `--accept-unmoved` must name exactly the keys that the plan lists as "Not
  moved". The data of these mounts stays on the old server; the application on
  the new server does not get it. Do not accept them for the user without asking.
- After the switch the move is in a hold (7 days): `rollback` moves the app back
  and loses the data written on the new server since the move; `finish` ends the
  hold early. Both are user decisions: ask first.
- `cancel`, `finish`, `rollback`, `remove-forwarders`, `cleanup-now`,
  `abandon`, `start-old-copy` and `remove-kept-copy` need `--move`; `status`,
  `retry` and `confirm-downtime` use the newest move without it. `status` prints
  the actions that are possible now; the API can still refuse, and the CLI then
  prints the reason (exit 1).
- `--accept-rebuild` (CLI 0.11.0): the plan says "Build on the new server" when
  the new server has another CPU architecture; the confirm then needs this flag.
  Show the plan item to the user and pass the flag only after the user accepted it.
  On such a move every Postgres-family and MySQL datastore moves by dump and
  restore (no choice; `--dump-restore` is not needed), and a Postgres 18, 19
  or pgvector datastore is built again on the new server (the same item and
  flag; its extensions can get a newer version). PostGIS types `postgis` and
  `postgis_16_34` cannot move to arm64 (blocker `datastore_image_arch`).
- `--dump-restore <datastore id>` (CLI 0.12.0, repeat; only with `--to`): a
  Postgres-family or MySQL datastore moves by dump and restore instead of a
  copy of its data folder. It is safer and the downtime is longer. The plan
  shows each datastore with its method and the flag for one that can use it.
  After `confirm` Appliku tests the dump and the restore while the
  application runs; a failed test ends the move with no downtime. When the
  measured downtime is more than 2 x the confirmed one + 5 minutes, the move
  waits for `confirm-downtime`. A change of the database after the plan
  cancels the move before the downtime; a change in the last minutes stops
  the switch (downtime, then the old server runs again). Settings outside the databases (for example
  `ALTER SYSTEM` values, a changed `pg_hba.conf`, MySQL `SET PERSIST`) are
  "Not moved" items. To change the method, make a new plan with the flag.
  SDK: `client.moves.create(team, app, server_id, dump_restore=[ids])`.
- SDK `client.apps.update(server=...)` does not move data and is refused (400)
  for an app with data when moves are enabled for the team: use `apps move`.
- When a server of a move is deleted or does not answer, Appliku keeps the
  copy of the application that is left (a "kept copy"). All three actions below
  are user decisions: ask first, and check `actions.abandon`,
  `actions.start_old_copy` and `actions.remove_kept_copy` (`allowed`, `reason`).
  - `abandon` (a move that has not ended: a server does not answer, a step
    waits too long, or a start of the old copy runs) ends the move now or at
    the latest in one hour. No data is deleted; the application stays where it
    runs now. If the server answers again later, the copy on it can start again
    by itself. Show the user the text of `abandon_preview` first.
  - An ended move lists its kept copies in `status --output json` under
    `kept_copies` (`kind`, `server_id`, `server_name`, `folders`, `may_run`,
    `removal_at`). Only a stopped old copy (`source_copy`) of a move that
    switched and was not rolled back is removed automatically, and not earlier
    than 30 days after the end of the move (`removal_at`; null when there is no
    automatic removal). `start-old-copy` and `remove-kept-copy` print the kept
    copies before they ask.
  - `start-old-copy` makes the kept copy the application again (the app, its
    datastores, DNS, nginx and cron jobs go to the server of that copy). The
    data written on the other server since then is lost. Today only a kept
    `source_copy` (the old copy on the old server, after the switch and with no
    rollback) can start; for the other kinds the refusal names the other ways
    (copy the files from the listed folders, or remove the copy). If the gone
    server still runs but does not answer, both copies can run at the same
    time: tell the user to stop or destroy that server at the provider first.
  - `remove-kept-copy` asks Appliku to remove the kept copies that it can
    remove; the log of the move names the copies that stay. It can delete the
    last copy of the application and its data. Never pass `--yes` or `--name`
    without the user's explicit decision.

#### Move to a cluster (CLI 0.11.0)

A server-mode application can also move to a cluster of the team. Only the
application moves; its datastores stay on the old server and the cluster
connects to them. This is available only when Appliku enables the move to a
cluster for the team (else `plan --to-cluster` answers 403, and `targets`
lists no clusters).

```bash
appliku apps move targets --team <team_path> --app <id>          # also lists the clusters, "ok" or the refusal
appliku apps move plan    --team <team_path> --app <id> --to-cluster <cluster id or name> \
  [--scale <process>=<n> ...] --wait                             # replicas 0-100 per process
# confirm, status, finish: as above
appliku apps move rollback --team <team_path> --app <id> --move <move_id> [--accept-new-domains]  # no --accept-data-loss
appliku apps move recover-without-stop-proof --team <team_path> --app <id> --move <move_id>
```

- Give `plan` exactly one of `--to` and `--to-cluster`; `--scale` only with
  `--to-cluster`. Without `--scale`, a process with 0 replicas stays at 0 and
  every other process gets 1. The plan shows the new address (the load balancer IP),
  "Database access", the replicas, the warnings and the refusals. To change
  the replicas, `cancel` the move and make a new plan.
- **Static site:** after a move to a cluster, a rollback is not possible. The
  plan, `status` and `confirm` print "No rollback: After the move, a rollback
  of this static site is not possible. You can only finish the move."
  (`rollback_supported` is false, the text is in `rollback_refusal`). Tell the
  user this before the confirm, and confirm only after the user agrees.
- The rollback of a move to a cluster needs no `--accept-data-loss`: the
  datastores stayed on the old server. `--accept-new-domains` is still needed
  when domains were added in the hold (they have no certificate on the old
  server): tell the user. The rollback is a user decision: ask first.
- `start-old-copy` of a move to a cluster also accepts the shared-database
  risk: "The copy on the cluster can still run and use the same databases
  until Appliku can stop it." The CLI shows this text and asks; the answer
  (or `--yes`) accepts it. The CLI has no separate flag; the SDK and the API
  need `accept_shared_database=True`. Never pass `--yes` without the user's
  explicit decision.
- `recover-without-stop-proof`: when a move to a cluster fails before its
  switch and Appliku cannot prove that the application stopped on the
  cluster, the move waits (`status --follow` stops there;
  `actions.recover_without_stop_proof.allowed` is true). This command starts
  the application on the old server without the proof. The same
  shared-database risk applies (the prompt or `--yes` accepts it). Ask the
  user first.

### config

Heroku-style environment variables (`heroku config` / `config:get` / `config:set` / `config:unset`).

```bash
appliku config list --team <team_path> --app <id>              # table: name, value, source, mode
appliku config list --team <team_path> --app <id> --output json   # {"NAME": "value", ...}
appliku config list --team <team_path> --app <id> --output shell  # NAME='value' (POSIX shell, not dotenv)
appliku config get KEY --team <team_path> --app <id>           # value only; exit 1 if not set
appliku config set A=1 B=2 --team <team_path> --app <id>       # merges; split on first '='
appliku config set A=1 --team <team_path> --app <id> --deploy  # save, then start a full deployment
appliku config set A=1 --team <team_path> --app <id> --restart # save, then restart WITHOUT a build
appliku config unset A B --team <team_path> --app <id>         # missing names: warning, exit 0
```

- Changes take effect on the next deployment or restart; nothing restarts on save.
  `--deploy` = full build; `--restart` = no build, runtime values only (build-time
  variables like NEXT_PUBLIC_* keep old values -> use `--deploy` for those). The two
  flags cannot be combined; neither runs when nothing was written. Saved-but-not-
  started exits 1 with "Config saved, but ..." (do NOT retry the write).
- `set` refuses `KEY=` (an empty value deletes on Appliku — use `unset`), the same
  name twice, and values with leading/trailing whitespace.
- Variables defined in `appliku.yml` cannot be set or unset here. Replacing a
  database- or domain-linked variable with a static value needs `--replace-links`.
- `set`/`unset` verify the API response and exit 1 naming any key not applied.
  The pre-write checks are advisory: a variable linked to a database between the
  check and the write is replaced without warning.
- `set`/`unset` need a token with `env_vars` read + update; `--deploy`/`--restart`
  also need `deployments` create.

### deployments

```bash
appliku deployments list   --team <team_path> --app <id>
appliku deployments latest --team <team_path> --app <id>
appliku deployments logs   --team <team_path> --id <deployment_id>
```

### domains

```bash
appliku domains list      --team <team_path> --app <id>
appliku domains create    --team <team_path> --app <id> --domain example.com
appliku domains delete    --team <team_path> --app <id> --id <domain_id>
appliku domains check-dns --team <team_path> --app <id> --domain example.com
```

### datastores

```bash
appliku datastores list    --team <team_path> --app <id>
appliku datastores start   --team <team_path> --app <id> --id <datastore_id>
appliku datastores stop    --team <team_path> --app <id> --id <datastore_id>
appliku datastores restart --team <team_path> --app <id> --id <datastore_id>
appliku datastores redeploy --team <team_path> --app <id> --id <datastore_id> [--yes]
appliku datastores delete  --team <team_path> --app <id> --id <datastore_id>
```

`datastores list` has a `problem` column: `failed` (the last deployment failed)
or `stuck` (no result after 15 minutes). `datastores redeploy` deploys the
datastore again with its saved settings (data, user, password and app env vars
do not change); it is the fix for these two problems, not for a slow database or
a wrong password. A 409 refuses the redeploy: show its text to the user (for
example another action runs on the datastore: wait; or contact support).

### volumes

```bash
appliku volumes list   --team <team_path> --app <id>
appliku volumes delete --team <team_path> --app <id> --id <volume_id>
```

### crons

```bash
appliku crons list   --team <team_path> --app <id>
appliku crons delete --team <team_path> --app <id> --id <cron_id>
```

### clusters

```bash
appliku clusters list   --team <team_path>
appliku clusters delete --team <team_path> --id <cluster_id>
```

### servers

```bash
appliku servers list --team <team_path>
appliku servers get  --team <team_path> --id <server_id>

# Hetzner Cloud server provisioning (requires the team to have a stored
# Hetzner Cloud token, configured in the dashboard under Team Settings ->
# Cloud Providers).
appliku servers hetzner-info   --team <team_path> --show locations
appliku servers hetzner-info   --team <team_path> --show server-types \
  --category "ARM" --location nbg1
appliku servers create-hetzner --team <team_path> \
  --location nbg1 --server-type cax11 \
  [--enable-backups] [--cluster <cluster_id>]
```

### invites

```bash
appliku invites list   --team <team_path>
appliku invites delete --team <team_path> --id <invite_id>
```

### migrations

```bash
appliku migrations list --team <team_path>
appliku migrations logs --team <team_path> --id <migration_id>
```

### ssh-keys

```bash
appliku ssh-keys list
appliku ssh-keys add    --key "ssh-rsa AAAA... user@host"
appliku ssh-keys add    --key "$(cat ~/.ssh/id_ed25519.pub)"
appliku ssh-keys delete --id <key_id>
```

**Only an OpenSSH public key is accepted.** The value must be one line starting
with a key type (`ssh-ed25519`, `ssh-rsa`, `ecdsa-sha2-nistp256`, ...) followed
by base64 key data; a comment after it is kept. Anything else is rejected with
HTTP 400.

Send the `.pub` file, never the private key. These are commonly pasted here by
mistake and all of them are refused:

- a private key (`-----BEGIN ... PRIVATE KEY-----`, or base64 starting
  `b3BlbnNzaC1rZXktdjEA`)
- a PuTTY/RFC4716 export (`---- BEGIN SSH2 PUBLIC KEY ----`) — convert it with
  `ssh-keygen -i -f key.pub` first
- a fingerprint (`SHA256:...`), a git remote URL, or an access token

Keys are also imported automatically when Git credentials are saved in the
dashboard. Keys the provider serves that are not valid OpenSSH public keys are
skipped, reported as `skipped_public_keys`, and summarized in one email.

## Python SDK Reference

```python
from appliku import Appliku

client = Appliku()                     # uses APPLIKU_TOKEN or config file
client = Appliku(token="YOUR_TOKEN")   # explicit token
```

### apps

```python
client.apps.list("my-team")
client.apps.get("my-team", app_id=42)
client.apps.create_from_github(
    "my-team", "owner/repository", name="myapp", branch="main", server=12
)
client.apps.create_from_gitlab(
    "my-team", "group/repository", name="myapp", branch="main", cluster=7
)
client.apps.create_from_custom_git(
    "my-team",
    "git@example.com:team/repository.git",
    name="myapp",
    branch="main",
    server=12,
    private_key=private_key_contents,
)
client.apps.update("my-team", app_id=42, branch="develop")
client.apps.delete("my-team", app_id=42)
client.apps.deploy("my-team", app_id=42)
client.apps.restart("my-team", app_id=42)               # no build; {success, deployment_id}
client.apps.restart_availability("my-team", app_id=42)  # {available, reason, source_number_id}

# Config vars. get returns {"env_vars": [...]}; set takes a flat mapping.
vars = client.apps.get_config_vars("my-team", app_id=42)

# set MERGES: variables not named in the call are left untouched.
client.apps.set_config_vars("my-team", app_id=42, vars={"DEBUG": "false"})

# An empty ("" or None) value DELETES the variable rather than blanking it.
client.apps.set_config_vars("my-team", app_id=42, vars={"DEBUG": ""})  # deletes DEBUG
client.apps.delete_config_var("my-team", app_id=42, key="OLD_VAR")  # returns remaining vars

# Logs — get_logs takes a list of processes (None = all), works on any mode.
logs = client.apps.get_logs("my-team", app_id=42)  # all processes
logs = client.apps.get_logs("my-team", app_id=42, processes=["web", "celery"], tail=100)
# get_service_logs is a deprecated shim that delegates to get_logs.
logs = client.apps.get_nginx_logs("my-team", app_id=42, domain="example.com", tail=100)
logs = client.apps.get_load_balancer_logs("my-team", app_id=42, domain="example.com", tail=100)
```

### deployments

```python
client.deployments.list("my-team", app_id=42)
client.deployments.get("my-team", app_id=42, deployment_id=1234)
client.deployments.latest("my-team", app_id=42)
client.deployments.logs("my-team", deployment_id=1234)
```

### domains

```python
client.domains.list("my-team", app_id=42)
client.domains.get("my-team", app_id=42, domain_id=7)
client.domains.create("my-team", app_id=42, domain="example.com")
client.domains.delete("my-team", app_id=42, domain_id=7)
result = client.domains.check_dns("my-team", app_id=42, domain="example.com")
```

### datastores

```python
client.datastores.list("my-team", app_id=42)
client.datastores.get("my-team", app_id=42, datastore_id=5)
client.datastores.create("my-team", app_id=42, name="mydb", kind="postgresql")
client.datastores.start("my-team", app_id=42, datastore_id=5)
client.datastores.stop("my-team", app_id=42, datastore_id=5)
client.datastores.restart("my-team", app_id=42, datastore_id=5)
client.datastores.redeploy("my-team", app_id=42, datastore_id=5)  # 0.10.0
client.datastores.delete("my-team", app_id=42, datastore_id=5)
```

### moves (0.10.0; cluster moves 0.11.0)

```python
from appliku.resources.moves import plan_ready, unmoved_keys

client.moves.targets("my-team", 42)                      # app_refusals, servers[].refusals
move = client.moves.create("my-team", 42, target_server=17)
# 0.11.0, a move to a cluster (only when Appliku enables it for the team; else 403).
# Give exactly one of target_server and target_cluster:
#   move = client.moves.create("my-team", 42, target_cluster=37, scales={"web": 2})
# move.rollback_supported is False (move.rollback_refusal): a static site has no
# rollback after the move. Tell the user before the confirm.
move = client.moves.wait_for("my-team", 42, move.id, until="plan_ready")
# move.preview: blockers, warnings, unmoved, estimate; move.actions.<name>.allowed/reason
# confirm starts the move and the downtime: show the plan to the user and
# confirm only after the user chose to start it.
if plan_ready(move) and (move.preview or {}).get("can_continue") and user_chose_to_start:
    # unmoved_keys(move): the mounts that do not move. Show them to the user and
    # pass only the keys that the user accepted.
    client.moves.confirm("my-team", 42, move.id, name="myapp", accept_unmoved=accepted_keys)
    # 0.11.0: add accept_rebuild=True only when the plan has preview.rebuild and the user accepted it.
move = client.moves.wait_for("my-team", 42, move.id, until="settled", timeout=6 * 3600)
client.moves.logs("my-team", 42, move.id, after=None)
# In the hold (move.status.value == "holding"), ASK THE USER first. finish ends
# the rollback option; rollback loses the data written on the new server since
# the move. Call one of them only after the user chose it:
#   client.moves.finish("my-team", 42, move.id)
#   client.moves.rollback("my-team", 42, move.id, accept_data_loss=True)
#   (a move to a cluster: the API ignores accept_data_loss, the datastores stayed;
#   the SDK argument is still required, False is fine)
# A server of a running move does not answer (show move.abandon_preview to the user):
#   client.moves.abandon("my-team", 42, move.id)
# An ended move with kept copies (move.kept_copies is not empty):
#   client.moves.start_old_copy("my-team", 42, move.id, accept_data_loss=True)
#   (a move to a cluster: also accept_shared_database=True; the copy on the cluster
#   can still run and use the same databases until Appliku can stop it)
# A move to a cluster waits for the stop-proof decision
# (move.actions.recover_without_stop_proof.allowed):
#   client.moves.recover_without_stop_proof("my-team", 42, move.id, accept_shared_database=True)
#   client.moves.remove_kept_copy("my-team", 42, move.id, confirm_name="myapp")
# User decisions only. Check move.actions.abandon / start_old_copy / remove_kept_copy /
# recover_without_stop_proof
# (.allowed, .reason) first.
# also: list, latest, get, confirm_downtime, cancel, retry, remove_forwarders, cleanup_now
```

Never accept unmoved mounts for the user: their data stays on the old server.

### volumes

```python
client.volumes.list("my-team", app_id=42)
client.volumes.create("my-team", app_id=42, name="uploads", mount_path="/app/uploads")
client.volumes.update("my-team", app_id=42, volume_id=3, mount_path="/app/media")
client.volumes.delete("my-team", app_id=42, volume_id=3)
```

### cron_jobs

```python
client.cron_jobs.list("my-team", app_id=42)
client.cron_jobs.create("my-team", app_id=42, schedule="0 * * * *", command="python manage.py clearsessions")
client.cron_jobs.update("my-team", app_id=42, cron_id=8, schedule="30 2 * * *")
client.cron_jobs.delete("my-team", app_id=42, cron_id=8)
```

### clusters

```python
client.clusters.list("my-team")
client.clusters.get("my-team", cluster_id=2)
client.clusters.create("my-team", name="prod-cluster")
client.clusters.delete("my-team", cluster_id=2)
```

### servers

```python
client.servers.list("my-team")
client.servers.get("my-team", server_id=10)

# Hetzner Cloud provisioning. Requires the team to have a stored Hetzner Cloud
# token. Server types and locations are pulled live from Hetzner.
info = client.servers.get_hetzner_cloud_info("my-team")
client.servers.create_hetzner_cloud(
    "my-team",
    location="nbg1",
    server_type="cax11",
    hetzner_backups_enabled=False,
    cluster=None,
)
```

### invites

```python
client.invites.list("my-team")
client.invites.create("my-team", email="colleague@example.com")
client.invites.delete("my-team", invite_id=4)
```

### migrations

```python
client.migrations.list("my-team")
client.migrations.run("my-team", app_id=42, command="python manage.py migrate")
client.migrations.logs("my-team", migration_id=99)
```

### public_keys

```python
client.public_keys.list()
client.public_keys.create("ssh-rsa AAAA... user@host")
client.public_keys.delete(key_id=12)
```

## Error Handling (SDK)

```python
from appliku import (
    Appliku,
    AuthenticationError,
    AuthorizationError,
    NotFoundError,
    ConflictError,
    ValidationError,
    RateLimitError,
    ServerError,
)

client = Appliku()
try:
    client.teams.get("missing-team")
except AuthenticationError:
    print("Invalid or missing token")
except NotFoundError as exc:
    print(exc)
```

| Exception | HTTP status |
|---|---|
| `AuthenticationError` | 401 |
| `AuthorizationError` | 403 |
| `NotFoundError` | 404 |
| `ConflictError` (0.10.0) | 409 |
| `ValidationError` | 400 |
| `RateLimitError` | 429 |
| `ServerError` | 5xx |

Every exception has `status_code`, `response` (the body), `code` (the body's
`code`, for example `operation_gate`, `datastore_busy`) and, for a refused move
plan, `refusals` and `warnings`.

## Common Workflows

### Find an app and check its latest deployment

```bash
appliku teams list
appliku apps list --team my-team --output json   # grab the numeric app id
appliku deployments latest --team my-team --app 42
appliku deployments logs --team my-team --id <deployment_id>
```

### Tail application logs

```bash
# All processes (default when no --process is given)
appliku apps logs --team my-team --app 42 --tail 200

# Specific processes
appliku apps logs --team my-team --app 42 --process web --process celery --tail 200
```

### Add a custom domain and verify DNS

```bash
appliku domains create    --team my-team --app 42 --domain example.com
appliku domains check-dns --team my-team --app 42 --domain example.com
```

### Restart a datastore

```bash
appliku datastores list    --team my-team --app 42   # get the datastore id
appliku datastores restart --team my-team --app 42 --id 5
```

## Gotchas

- **Create does not deploy**: Application creation starts initial configuration inspection only. Deploy in a separate command after setup is complete.
- **`apps logs` is the one logs command**: It works for both server-mode and cluster-mode apps and returns logs for one or more processes. It POSTs a request, then polls until logs are ready. (`apps service-logs` still exists as a deprecated alias for `apps logs -p <service>`.)
- **`--process` is repeatable and optional**: Pass it multiple times for multiple processes (`--process web --process celery`); omit it entirely to get logs for **all** of the app's processes.
- **Machine-readable output**: Add `--output json` to any list command when you need to parse IDs programmatically.
- **`APPLIKU_TOKEN` for CI**: Set this environment variable to avoid interactive login in automated contexts.
