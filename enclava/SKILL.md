---
name: enclava
description: >-
  Deploy, operate, and troubleshoot apps on Enclava (confidential workloads on
  Kubernetes with AMD SEV-SNP). Use for operational work such as deploying an OCI
  image or hosted template, CLI commands, status/logs/config, locked storage,
  recovery, lifecycle, signer identity, or a failed/stuck deploy. For conceptual
  trust questions use enclava-concepts; for independent evidence appraisal use
  enclava-verify.
compatibility: enclava CLI v0.1.x
---

# Running apps on Enclava

Enclava runs container images as **confidential workloads**: a Kata VM-backed guest
(AMD SEV-SNP) where the host can schedule and operate the workload but cannot read its
memory or silently alter its trusted startup path. Secrets and encrypted state are
released *inside* the guest only after image-signature, policy, and attestation checks
pass. Everything a user does goes through the `enclava` CLI (or the hosted console).

Use `enclava <command> --help` to discover the current surface, but treat help text as
a pointer rather than proof: when an executable or security claim matters, verify it
against observed behavior or the current implementation.

## The model in five facts

1. **Manual OCI apps use `enclava.toml`** (`enclava init` scaffolds it); hosted apps
   come from platform templates. `create` may resolve a tag, but `deploy` requires a
   **digest-pinned image** (`image@sha256:<digest>`).
2. **Custom images need portable cosign signature and bound provenance material**, and
   the signer identity is pinned per app (`--signer-subject` at create, or
   `enclava signer set` later). Keyless GitHub Actions signing is the intended path.
3. **Persistent storage is encrypted** (LUKS inside the guest). In password mode the
   owner holds a password and a BIP39 recovery mnemonic; the platform cannot read the
   volume.
4. **`enclava status` is the source of truth** — it reports runtime status, TEE unlock
   state, and `tee_error` (the in-guest startup failure reason) without needing cluster
   access. Read it before theorizing.
5. **Fail-closed is correct behavior.** When verification can't complete, startup
   blocks or the deploy fails rather than proceeding unverified. Explain this; never
   improvise workarounds around a verification failure.

## Secret-handling discipline (always)

- Storage passwords and recovery mnemonics are **owner secrets**. Never echo them into
  logs, shell history, or files beyond what the user asked for. Prefer
  `--storage-password-file` on a tight-permissions file over inline arguments.
- The mnemonic is shown **once** at claim time. By default the CLI persists it locally
  so `enclava key backup` covers it; export a backup outside the project immediately
  after the first deploy (`enclava key backup --out "$HOME/.enclava/my-app-recovery.json"`).
- If a password-mode app has neither a usable password nor a recovery mnemonic/backup,
  the platform cannot recover its data. Auto-unlock is not a substitute for an owner
  backup.

## Unlock state machine (core of the platform)

Password mode (the default):

- **First deploy** waits for the TEE bootstrap endpoint, then the owner **claims**
  ownership: sets a password; the BIP39 mnemonic is persisted to the protected
  local keystore (never printed). With `deploy --storage-password-file PATH` (or
  `claim --password-file`) this is fully unattended — run `enclava key backup` right
  after so the recovery material has a durable copy.
- **Every restart** boots to `locked`; unlock it with the password
  (`enclava unlock [--app NAME]`, prompted or `--password-file`). On the manual OCI path, a complete
  redeploy command can instead supply the same `--storage-password-file`.
- **`enclava recover --mnemonic-file M --new-password-file P`** re-wraps the seed under
  a new password after a lost password. It requires a **freshly booted, locked TEE** —
  against a long-running pod it fails `recovery_requires_locked_init_verifier`; restart
  the workload first. After a successful recover, storage is already unlocked — a
  follow-up `enclava unlock` reporting "not locked" is expected.
- **`enclava change-password`** re-wraps under a new password (needs the old one).

Auto-unlock (manual OCI apps only, for unattended restarts):

- A **first deploy must be password mode** because no owner seed exists before claim;
  current `create` rejects `unlock.mode = "auto"` up front. Deploy password-first,
  then run `enclava auto-unlock enable --image <image>@sha256:<digest>`. This wraps
  the owner seed for **KBS-attestation-gated release**: KBS returns the encrypted
  envelope only to a guest that passes attestation; it is not TEE-hardware sealing.
  The command also binds the digest into a signed redeploy descriptor.
  `enclava auto-unlock disable --image <image>@sha256:<digest>` returns to
  password-on-restart. Both transitions currently require local `enclava.toml`, so
  hosted template apps cannot use them.

Known failure mode — **stale owner seed after a waived destroy**: if
`destroy --abandon-teardown` finishes a destroy whose workload teardown could not
complete, the previous owner seed may remain in KBS and recreating the app with
the **same name** is refused `stale_owner_seed` (the message carries the full
remediation). Remedies: the platform operator erases the stale seed and clears
the recorded waiver, or choose a different app name. If the workload was already
gone (or its teardown had completed), no waiver is recorded and recreation is
unaffected. A destroy whose teardown cannot complete otherwise stays retryable by
design — the app remains in the deleting state and stays wired.

## Command map (verified surface)

| Area | Commands |
| --- | --- |
| Auth | Hosted/device flow: `login [--api-url] [--no-browser] [--org] [--approve-logs] [--replace-session]`, `whoami`, `logout`. Standalone CAP only: `signup`, `login --nostr`, `login --email` (their credential prompts have flag forms — `--email-address`/`--password-file`, `--npub`/`--nsec-file`; `--email` and `--nostr` are mutually exclusive). |
| Scaffold | `init [--app-name NAME] [--port PORT]` (prompts when interactive; validates the name — leading letter, `[a-z0-9-]`, no consecutive hyphens, ≤63 — before writing anything), `prepare [--yes]` (`--yes` skips the update-existing-files confirmation; without it a non-interactive run refuses and names `--yes`) |
| Deploy | `create [--image] [--signer-subject] [--signer-issuer]`, `deploy --image IMG@DIGEST [--set K=V] [--set-file K=PATH] [--storage-password-file PATH]` |
| Observe | `status [--app] [--json]`, `logs [--app] [-f] [--log-private-key-file PATH]`, `log-key generate/list/select/revoke` |
| Config | `config set K=V…` (local `enclava.toml`; no `--app`), `config get` (names only), `config unset K` |
| Templates | `template list`, `template deploy [slug] --name N [--ssh-public-key-file PATH]`, `template ssh-command --name N --wait [--json]` |
| Ownership | `claim`, `unlock`, `recover`, `change-password`, `auto-unlock enable/disable` |
| Lifecycle | `rollback [--to ID]` (without `--to` it confirms the default target), `destroy [--app] [--force] [--abandon-teardown]` |
| Global | `--no-input` (env `ENCLAVA_NO_INPUT`) on any command: never prompt — take defaults or fail naming the flag that replaces the prompt |
| Identity | `signer set/rotate`, `org create/switch/invite/members/keyring`, `key status/backup/restore/setup/rotate-owner` |
| Domains | `domains add/verify/list/remove` |

Commands that expose `--app` can address a named hosted or manual app. Commands such
as `create`, generic `deploy`, `config set`, and `auto-unlock` instead require local
`enclava.toml`; check the selected journey before assuming `--app` exists.

## Automation contract (agents)

- **Run unattended commands with `--no-input`** (or `ENCLAVA_NO_INPUT=1` in the
  environment): nothing may stop at a prompt. A prompted command either takes
  deterministic defaults (`init`) or fails naming the flag that replaces the prompt
  (e.g. `pass --force to destroy without confirmation`) — read that message and use
  the flag it names.
- **Every interactive input has a flag alternative**; secrets come from `--*-file`
  paths, never process arguments.
- **Read state with `enclava status [--app] --json`** (stable shape; `null` means
  "not verified", never a verified-looking value).
- **`destroy --force` only on explicit user instruction.** `destroy` deletes the
  app's data volumes; its default double confirmation (yes/no, then type the app
  name) exists to stop accidents. Never add `--force` on your own initiative.

## Attended steps and handoffs

Most commands run fully unattended (`init` takes deterministic defaults without a
TTY, `prepare --yes`, `create`, `status`, first `deploy`/`claim` with
`--storage-password-file`, `unlock --password-file`, `recover --mnemonic-file
--new-password-file`, `rollback --to`, `destroy --force`). Three steps involve a
human:

- **`key setup`** (fresh org, before the first `create`) — passphrase prompt is
  scriptable (`--passphrase-file`/`--new-passphrase-file`), but the `--backup-out`
  file is owner signing material: have the operator choose where it goes.
- **`login`** — run `login --no-browser`, print the URL and code, wait for the
  operator to approve in the browser. The device code is short-lived and
  single-use, so showing it is fine — don't redact it or warn about it.
- **Recovery-material storage** — the first deploy/claim runs fully unattended with
  `--storage-password-file`, but it *creates* owner secrets: the one-time recovery
  mnemonic lands in the protected local keystore. Right after the first deploy, run
  `enclava key backup --out <file>` and have the operator keep that file durable.

Rule of thumb: any step that *creates owner secrets* (the claim mnemonic, a key
setup backup, a recovery export) ends with a human storing them; do not invent
unattended paths around that.

## Pick the journey

Read exactly one reference (they carry the detail; this file is the shared spine):

- **Deploying your own container image** → read `references/deploy-oci.md`
- **Deploying a hosted template (fastest path, e.g. a confidential SSH box)** → read `references/hosted-template.md`
- **App already exists — logs, config, domains, unlock/recovery, rollback, destroy, or anything failed/stuck** → read `references/day-2.md`

For *understanding* the platform (what a TEE proves, who can read what, threat model),
the `enclava-concepts` skill is the right tool. For *independently verifying* a running
app's trust properties, use `enclava-verify`.

## Triage quick table

| Symptom | Meaning / first move |
| --- | --- |
| Deploy stalls; `status` shows `tee_error: … Read-only file system (os error 30)` | A `storage.paths` entry isn't a `VOLUME` in the image → add `VOLUME`, redeploy. |
| First deploy in auto mode is rejected | Expected — deploy password-mode first. |
| `portable_verification_material_unavailable` | CAP couldn't assemble portable signature/provenance material. One common cause is a cosign 2.x legacy `.sig` object without portable DSSE material; re-signing with the generated cosign 3.x workflow fixes that case. Registry/referrer errors, missing provenance, malformed bundles, and size limits share this code — inspect the signature and provenance objects before concluding. |
| `status` shows `Status: drifted` after a failed deploy | Follow-up deploys won't roll — `destroy --app <name> --force`, then `create` + `deploy` clean. |
| Same-name `create` refused `stale_owner_seed` (create-time; `already_claimed` can still surface as a template bootstrap-wait fallback) | Stale owner seed after a waived destroy (see above) — operator erases the seed + clears the waiver, or pick a different name. |
| `destroy` keeps failing its workload teardown | By design it stays retryable (app remains in `deleting`, still wired). Retry when the workload is reachable; use `destroy --abandon-teardown` only when recovery is ruled out (waiver). |
| `destroy` returns `app mutation already in progress` (409) | An in-flight deployment holds the mutation lease — wait for it to finish (check `status`), then retry. |
| Browser rejects the app cert (`ERR_CERT_AUTHORITY_INVALID`) | Some environments serve a Let's Encrypt staging cert — a platform toggle, not an app fault. |
| Unlock "did not complete" right after a deploy | The background unlock may still be running — trust `enclava status` over the CLI exit message. |
| `error: this step would prompt for input in a non-interactive session` | A prompt was needed but prompting is off — pass the flag the message names (and `--no-input` on every automated run). |
| `error: an interactive terminal or --password-file <PATH> is required to provide the … non-interactively` | A secret was needed without its file flag — pass the `--*-file` flag the message names. |

## Going deeper

Concepts, architecture, and full reference live in the Enclava developer docs
(https://docs.enclava.dev — also in this repo under `docs/`). Prefer them for *what*
something is; prefer this skill for *how* to operate it.
