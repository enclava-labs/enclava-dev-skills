# Journey: day-2 — status, logs, config, domains, recovery, lifecycle

For apps that already exist (either on-ramp). The unlock state machine and
secret-handling rules live in the skill spine (`../SKILL.md`) — this file assumes them.

## Status and triage

```bash
enclava status [--app NAME]
```

Reads, in order of interest: `Status` (running / drifted / …), `TEE` (unlocked /
locked), `Unlock`, and `tee_error`. `tee_error` is the in-guest startup failure reason
from `enclava-init` — when a deploy stalls or an app won't come up, read it first; it
states the cause without any cluster access. The spine's triage table maps the common
values.

## Logs (tenant-encrypted; needs a one-time ceremony)

Tenant logs are end-to-end encrypted — the platform cannot read them. To stream:

```bash
enclava login --approve-logs                          # grant this CLI session log access
enclava log-key generate --app NAME --key-id logs-laptop-2026q3
# tenant-held X25519 key: private key stays local; only the public key is registered
# (re)deploy so the app is bound to the key — or select an existing one at deploy time
enclava logs --app NAME -f --log-private-key-file <private-key-file>
```

`log-key list/select/revoke` manage the registered keys. Losing the private key means
losing log access (data itself is unaffected).

## Config secrets

```bash
enclava config set FEATURE_FLAG=enabled  # multiple KEY=VALUE pairs; no --app
                                          # resolves local enclava.toml only
enclava config get [--app NAME]           # lists key NAMES only
enclava config unset FEATURE_FLAG [--app NAME]
```

`config set` has no file-input form: its values appear in the local process arguments
and may enter shell history, so do not use it for sensitive plaintext. For a manual OCI
app, prefer `deploy --set-file KEY=PATH`; hosted apps currently have no `config set`
path because they have no local `enclava.toml`. Values sent through either supported
path travel directly from the CLI to the attested TEE; the control plane does not
persist or return their plaintext.

## Custom domains

```bash
enclava domains add app.example.com --app NAME   # returns a TXT challenge
# publish the TXT record at your DNS, then:
enclava domains verify app.example.com --app NAME
enclava domains list --app NAME
enclava domains remove app.example.com --app NAME
```

## Rollback and destroy

```bash
enclava rollback --app NAME [--to DEPLOYMENT_ID]   # defaults to previous deployment
enclava destroy --app NAME [--force]
```

- `destroy` asks for confirmation unless `--force`.
- **Busy 409** (`app mutation already in progress`): an in-flight deployment holds the
  app's mutation lease. Not an error to fix — wait for it to settle (watch `status`),
  then retry. On hosted, a queued destroy may also complete on its own after the
  in-flight operation finishes.
- Destroy deletes data volumes (PVCs) regardless of pod health. It clears the key
  escrow only **best-effort via the live pod** — see the stale-escrow rules below
  before recreating an app with the same name.

## Ownership, recovery, key hygiene (password mode)

The mnemonic is an independent recovery representation of the owner seed — not a
copy of the password. Practical rules:

1. **After the first deploy/claim**:
   `enclava key backup --out "$HOME/.enclava/my-app-recovery.json"`. Verify with
   `enclava key status` what's covered locally.
2. **Lost password, have mnemonic**:
   ```bash
   enclava recover --app NAME --mnemonic-file M --new-password-file P
   ```
   Requires a **freshly booted, locked TEE** — against a long-running pod it returns
   `recovery_requires_locked_init_verifier`; restart the workload (platform-side pod
   restart / redeploy) and retry. Recover leaves storage unlocked; a follow-up
   `enclava unlock` reporting "not locked" is expected, not a bug.
3. **Rotation**: `enclava change-password --app NAME` (old → new password; both can
   come from files via `--current-password-file`/`--new-password-file` for scripted runs).
   `enclava key restore <backup>` re-derives local key material from a backup (e.g.
   new laptop) — after which `deploy`/`claim` work again.
4. **Stale escrow** (destroy-while-dead): the escrow survives, and a same-named new
   app *inherits* it — claim fails `already_claimed`, unlocks silently land
   wrong-password (`TEE: locked`, no error). Fix: `recover` with the **old** app's
   mnemonic. Prevent: prefer destroying while the app is healthy.
5. **Unattended restarts (manual OCI apps only)**:
   `enclava auto-unlock enable --image <img>@sha256:<digest>` wraps the seed for
   KBS-attestation-gated release; `disable` returns to password-on-restart. Both need
   the digest-pinned image and local `enclava.toml` to build a signed redeploy
   descriptor, and take the unlock password via prompt or `--password-file`.
   Hosted template apps cannot switch modes with the current CLI.

If both password and mnemonic are gone, the encrypted volume is unrecoverable by
design — the platform cannot help; say so plainly rather than suggesting tricks.

## What not to reach for

- There is no CLI path to read config plaintext back; the control plane does not store
  it, and there is no platform-side "reset" for owner secrets. Anything promising
  otherwise is a misunderstanding of the trust model.
- A verification/attestation failure that blocks startup is the system working as
  designed — diagnose the named cause (`tee_error`, signature mismatch, policy), don't
  hunt for a bypass.
