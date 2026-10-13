# Hermes Email channel, multi-user access, and secret placement under the Dockhand restart caveat

> Resolves wayfinder ticket #49. Compiled 2026-10-13 by direct read of primary
> sources — official Hermes docs (Messaging Gateway, Email Setup, the `openmail`
> plugin catalog entry, Secrets, Docker Setup) and local decision #8 in
> `truenas/apps/README.md` (Dockhand-injected secrets do not survive a
> daemon-initiated restart).

## TL;DR

- **Email mechanism — use the native Email gateway adapter (IMAP/SMTP), not
  `openmail`.** Native Email is a first-class platform (threads + image/file
  attachments) that reads an existing mailbox over standard IMAP/SMTP with
  zero external dependencies; replies are threaded via `In-Reply-To` /
  `References`. `openmail` is a third-party hosted-inbox service that gives the
  agent a *new* address over the OpenMail API + a websocket. For a personal
  agent that sends/receives on the operator's own address, native IMAP/SMTP is
  the right pick; there is no Gmail API involved (Gmail is just IMAP/SMTP with
  an App Password).
- **Multi-user = allowlist (who may reach) + admin/user split (what they may
  do).** Reach gate is `EMAIL_ALLOWED_USERS` (comma-separated full addresses).
  Email pairing is **opt-in and should stay off** (use the allowlist). The
  admin/user split is `allow_admin_from` (operator) + `user_allowed_commands`
  (family are regular users). If `allow_admin_from` is unset, the tier split is
  disabled and every allowed user gets full access.
- **Secrets live in the mounted `/opt/data/.env`, never only as Dockhand env
  vars.** Hermes's Docker image reads API keys from `/opt/data/.env` (the
  bind-mounted data volume); the Email password is documented as stored in
  `~/.hermes/.env` (= `/opt/data/.env`). Dockhand key-icon secrets vanish on a
  daemon-initiated restart (decision #8), so the Email credentials — and any
  Strata `API_KEY`, if one is later added — must be written into that mounted
  `.env` file, not injected only via the Dockhand env-var panel.
- **Voice Mode and Home Assistant are both "door open, no config now."** Voice
  Mode is a separate feature and Home Assistant is a plugin platform; neither
  is required for the email + multi-user slice. They can be added in a later
  slice without touching this config.

## 1. Native Email platform vs the `openmail` plugin

**Native Email gateway adapter (recommended for this slice).** Hermes has a
first-class Email platform. From the official [Email Setup] page:

- It uses **standard IMAP and SMTP** via Python's built-in `imaplib`,
  `smtplib`, and `email` modules — **no external packages, no bot API, no
  third-party service**. "Works with Gmail, Outlook, Yahoo, Fastmail, or any
  provider that supports IMAP/SMTP." There is **no Gmail API path**; Gmail is
  reached over IMAP/SMTP with a 16-character App Password (2FA required).
- **Receiving:** the adapter polls IMAP for `UNSEEN` messages every
  `EMAIL_POLL_INTERVAL` seconds (default 15), caches attachments (images →
  vision tool; PDF/ZIP → file access), strips HTML tags, filters self-messages
  and `noreply@`/`mailer-daemon@`/`bounce@`/`no-reply@` + `Auto-Submitted` /
  `Precedence: bulk` / `List-Unsubscribe` senders.
- **Replying:** replies go out over SMTP with `In-Reply-To` + `References`
  headers (threading), a `Re:` subject prefix, a generated `Message-ID`, plain
  UTF-8 text. The agent attaches files by emitting `MEDIA:/path/to/file`.
- **Config keys** (in `~/.hermes/.env`, i.e. `/opt/data/.env` in Docker):
  `EMAIL_ADDRESS`, `EMAIL_PASSWORD`, `EMAIL_IMAP_HOST`, `EMAIL_SMTP_HOST`
  (required); `EMAIL_ALLOWED_USERS`, `EMAIL_AUTHSERV_ID` (required unless
  sender-auth is off), `EMAIL_IMAP_PORT` (993), `EMAIL_SMTP_PORT` (587),
  `EMAIL_POLL_INTERVAL` (15), `EMAIL_HOME_ADDRESS`,
  `EMAIL_ALLOW_ALL_USERS` (false).

**`openmail` plugin (a different thing).** From the [openmail] catalog entry:
it "gives your Hermes agent its own email address" via the **OpenMail hosted
service** — inbound mail wakes the agent over a **websocket** (no webhook), and
replies go out in-thread. It needs `OPENMAIL_API_KEY` (an OpenMail API key) and
adds 9 tools (`openmail_send`, `openmail_reply`, `openmail_list_threads`, etc.).
Its sender gate is `OPENMAIL_ALLOWED_USERS` / `OPENMAIL_ALLOW_ALL_USERS`, with
modes `channel` / `notify` / `tool` (`OPENMAIL_MODE`).

**Decision:** for a threaded, send+receive **personal-agent inbox on the
operator's own mailbox**, the native Email adapter is correct — it reuses an
existing IMAP/SMTP account with no new external account or API key beyond the
mailbox's own App Password. `openmail` only makes sense if you specifically
want a separately-provisioned OpenMail address and are willing to depend on a
third-party inbox service.

## 2. Multi-user: operator + family

Three independent layers, all doc-backed.

**Reach gate (who may even reach the agent).** From [Email Setup] "Access
Control" and [Messaging Gateway] "Security": with `EMAIL_ALLOWED_USERS` set,
only those addresses (plus `GATEWAY_ALLOWED_USERS` or an approved pairing) are
processed; with no allowlist, unknown senders are **silently ignored**. Syntax
is a comma-separated list of **whole addresses** (a bare `alice` never matches
`alice@…`). `EMAIL_ALLOW_ALL_USERS=true` opens it to everyone (not recommended).

**Email pairing — leave off.** [Messaging Gateway] "DM Pairing": "Email is the
exception: unknown email senders are ignored unless email pairing is explicitly
enabled." [Email Setup] repeats it with a warning: "Use a dedicated inbox and
configure `EMAIL_ALLOWED_USERS` for normal operation. Email pairing is opt-in
because shared inboxes often contain unrelated unread messages." So: **do not
enable pairing; use the allowlist.**

**Admin vs regular user (what allowed users may do).** From [Messaging Gateway]
"Admins vs Regular Users": the allowlist answers "can this person reach the bot
at all?"; the admin/user split answers "what are they allowed to do?" Two tiers
per scope:

- **Admin** — full access to every slash command and gated capability.
- **Regular user** — chats normally, but can run only the slash commands in
  `user_allowed_commands`; the always-allowed floor is `/help` and `/whoami`.

The split is configured via `allow_admin_from` (admins) +
`user_allowed_commands` (what non-admins may run). **Backward-compat caveat
that matters here:** "if `allow_admin_from` is not set for a scope, the tier
split is disabled for that scope and every allowed user has full access." So to
make family members regular users you **must** set `allow_admin_from` to the
operator and provide `user_allowed_commands`.

**Concrete block (for ticket #50):**

```yaml
# /opt/data/config.yaml
gateway:
  platforms:
    email:
      enabled: true
      extra:
        # Operator is the only admin → full slash-command access.
        allow_admin_from: ["operator@absurdlab.dev"]
        # Everyone else who is allowlisted (family) is a regular user.
        user_allowed_commands: [status, model, usage]
```

**Undetermined (to verify at deploy):** the tier-split example in [Messaging
Gateway] is written for `discord` and nests under `gateway.platforms.<platform>.extra`,
whereas [Email Setup]'s own config examples (e.g. `skip_attachments`,
`unauthorized_dm_behavior`, `authserv_id`) nest under top-level
`platforms.email`. The docs do not show the email-specific nesting for
`allow_admin_from`/`user_allowed_commands`. The reach gate (`EMAIL_ALLOWED_USERS`)
is unambiguous and is the load-bearing control; confirm the tier-split keys land
via `/whoami` (shows `admin` / `user` / `unrestricted` + allowed commands) at
first deploy and adjust the nesting if needed.

The reach allowlist goes in the `.env` (see §3), not `config.yaml`:

```dotenv
EMAIL_ALLOWED_USERS=operator@absurdlab.dev,spouse@absurdlab.dev,kid@absurdlab.dev
```

## 3. Secrets placement under the Dockhand restart caveat

**Hermes's storage model (doc-backed).** The [Docker Setup] page: "The container
stores all user data (config, API keys, sessions, skills, memories) in a single
directory mounted from the host at `/opt/data`," and `.env` is listed as "API
keys and secrets" inside that volume. "API keys are read from `/opt/data/.env`
inside the container. You can also pass environment variables directly … Direct
`-e` flags override values from `.env`." The [Email Setup] page: "the password
is stored in `~/.hermes/.env` — protect this file (`chmod 600`)." (In the
Docker layout `~/.hermes` maps to the mounted `/opt/data`.) The [Secrets] page
adds that `.env` is also where the bootstrap token for *optional* external
secret managers (Bitwarden/1Password) lives; those managers are not needed for
this slice.

**The Dockhand caveat (doc-backed, local).** `truenas/apps/README.md` decision
#8: "secret-bearing stacks come back up **missing their secrets** after a
daemon-initiated restart (host reboot, dockerd crash) and must be redeployed
from Dockhand to re-inject." Dockhand secret values are set via the Stack-editor
env-var panel **key-icon toggle** and are injected at Deploy time, so a Docker
daemon restart re-creates the container without re-injecting them.

**Exact mechanism for this slice:**

1. Mount a persistent host directory into the container at `/opt/data` (the
   image's single source of truth). In Dockhand terms this is a normal bind
   mount on the ZFS pool — it is **not** a Dockhand secret and therefore
   survives daemon restarts.
2. Write a `.env` file in that directory containing the Email credentials (and
   any Strata key). Hermes reads `/opt/data/.env` at startup.
3. Do **not** put `EMAIL_PASSWORD` (or the Strata key) only in Dockhand's
   key-icon env-var panel — those values vanish on a daemon-initiated restart,
   and the stack would come back unable to authenticate.

```dotenv
# /opt/data/.env  (host: the bind-mounted Hermes data dir; chmod 600)
EMAIL_ADDRESS=hermes@absurdlab.dev
EMAIL_PASSWORD=<gmail app password>      # NOT the account password
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_IMAP_PORT=993
EMAIL_SMTP_PORT=587
EMAIL_POLL_INTERVAL=15
EMAIL_ALLOWED_USERS=operator@absurdlab.dev,spouse@absurdlab.dev,kid@absurdlab.dev
EMAIL_AUTHSERV_ID=mx.google.com          # Gmail; read the raw header to find yours
EMAIL_HOME_ADDRESS=operator@absurdlab.dev
```

**Strata key.** The deployed Strata stack currently **ships without an API key**
(its access control is LAN placement + `STRATA_ALLOWED_HOSTS`), per
`truenas/apps/strata/README.md` §"Reaching it by hostname". If a key is added
later it is the Strata-side `API_KEY=<key>`, passed as `Authorization: Bearer`.
The same rule applies: put the value in Hermes's mounted `/opt/data/.env` (or
its custom-endpoint provider config), **not** only as a Dockhand env var. Note
the Strata runbook already applies the analogous guidance for the Strata stack
itself: "`API_KEY=<key>` as a **plain** env var in the Dockhand panel — never
the key-icon secret toggle, since this stack is `restart: unless-stopped` and
secrets do not survive a daemon-initiated restart." **Undetermined:** the exact
Hermes-side env var name / provider field that carries the Strata key into the
custom-endpoint model config — that lives in Hermes's model-provider docs,
which are outside the sources fetched for this ticket.

## 4. Voice Mode and Home Assistant — defer, door is open

**Doc-backed:** the [Messaging Gateway] platform table lists **Home Assistant
(plugin)** as a platform and the page cross-references a separate **Voice Mode**
feature ("For the full voice feature set … see Voice Mode …"). [Email Setup]
requires neither.

**For this slice:** neither is needed. Email + multi-user config (above) works
without them. Both are additive later:

- **Voice Mode** is a feature enabled via the gateway (`/voice` and TTS) — no
  config needed now; the door is open because the gateway already "delivers
  voice messages" and `hermes gateway setup` / the platform list are unchanged.
- **Home Assistant** is a plugin platform installed when that slice is reached;
  it does not affect the Email allowlist, admin/user split, or secrets layout
  described here.

No configuration for either is required in ticket #50.

## Concrete config for ticket #50 (copy target)

`.env` in the mounted data volume (`/opt/data/.env`, `chmod 600`):

```dotenv
# --- Email channel (native IMAP/SMTP) ---
EMAIL_ADDRESS=hermes@absurdlab.dev
EMAIL_PASSWORD=<gmail app password>
EMAIL_IMAP_HOST=imap.gmail.com
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_IMAP_PORT=993
EMAIL_SMTP_PORT=587
EMAIL_POLL_INTERVAL=15
EMAIL_ALLOWED_USERS=operator@absurdlab.dev,spouse@absurdlab.dev,kid@absurdlab.dev
EMAIL_AUTHSERV_ID=mx.google.com
EMAIL_HOME_ADDRESS=operator@absurdlab.dev
# --- Strata (only if a key is later added; see §3) ---
# STRATA/API key value here, not in the Dockhand key-icon panel
```

`config.yaml` in the mounted data volume (`/opt/data/config.yaml`):

```yaml
gateway:
  platforms:
    email:
      enabled: true
      extra:
        allow_admin_from: ["operator@absurdlab.dev"]
        user_allowed_commands: [status, model, usage]
```

Deployment note: mount the persistent data dir at `/opt/data` and do **not**
inject `EMAIL_PASSWORD` (or the Strata key) through Dockhand's key-icon secret
toggle — write them to `/opt/data/.env` so they survive a daemon-initiated
restart (decision #8).

## Sources

- [Hermes Messaging Gateway](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/) — Email is a first-class platform (threads ✅, images ✅, files ✅); `EMAIL_ALLOWED_USERS` syntax; DM-pairing Email exception (off unless enabled); admin vs regular-user tier split (`allow_admin_from`, `user_allowed_commands`) and its "no `allow_admin_from` → split disabled" backward-compat rule; Home Assistant listed as a plugin platform; Voice Mode referenced.
- [Hermes Email Setup](https://hermes-agent.nousresearch.com/docs/user-guide/messaging/email) — IMAP/SMTP (no Gmail API), full `EMAIL_*` env reference, threading (`In-Reply-To`/`References`), attachment handling, access control incl. pairing warning, `EMAIL_AUTHSERV_ID`, password stored in `~/.hermes/.env` (`chmod 600`).
- [openmail plugin](https://hermes-agent.nousresearch.com/docs/plugins/openmail) — OpenMail hosted-inbox service, websocket inbound, `OPENMAIL_API_KEY`, `OPENMAIL_ALLOWED_USERS`, `OPENMAIL_MODE`.
- [Hermes Secrets](https://hermes-agent.nousresearch.com/docs/user-guide/secrets/) — secrets live in `~/.hermes/.env`; optional external managers (Bitwarden/1Password) with their bootstrap token also in `.env`.
- [Hermes Docker Setup](https://hermes-agent.nousresearch.com/docs/user-guide/docker) — `/opt/data` is the single source of truth; `.env` holds "API keys and secrets"; "API keys are read from `/opt/data/.env`"; `-e` flags override `.env`.
- Local: `truenas/apps/README.md` (decision #8 — Dockhand key-icon secrets vanish on daemon-initiated restart); `truenas/apps/strata/README.md` (Strata ships keyless; `API_KEY` placement guidance).

*Doc-backed vs undetermined:* the email mechanism, allowlist syntax, pairing
default, admin/user semantics, and `/opt/data/.env` storage are all directly
doc-backed. The exact `config.yaml` nesting for the email tier split
(`gateway.platforms.email.extra` vs `platforms.email.extra`) and the exact
Hermes-side env var that carries the Strata key are **undetermined** from the
sources fetched here and flagged inline in §2 and §3.
