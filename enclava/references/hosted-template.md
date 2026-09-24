# Journey: deploy a hosted template (fastest path)

Hosted templates are curated workload definitions managed by the Enclava PaaS — it
handles the runtime shape, sidecars, stable endpoint allocation, and config plumbing.
If a template matches the need, this is less work than the manual OCI path
(`deploy-oci.md`) and requires no image building or signing.

## 1. Log in and list

```bash
enclava login            # browser device flow; whoami to confirm org context
enclava template list
```

## 2. Deploy `debian-ssh-frp` (confidential SSH box)

```bash
enclava template deploy debian-ssh-frp --name shell \
  --ssh-public-key-file ~/.ssh/id_ed25519.pub
```

What the platform does: reserves a stable `host:port` endpoint on the Enclava relay,
stores that non-secret metadata, and injects the FRP relay credentials itself — the
user never handles relay credentials. The SSH public key is delivered to the workload
through a short-lived config-token path straight to the TEE.

Useful flags:

| Flag | Purpose |
| --- | --- |
| `--ssh-public-key KEY` | Key as an argument; repeatable. |
| `--no-wait` / `--ssh-timeout-seconds N` | Don't block on endpoint readiness (default waits up to 600s). |
| `--storage-password-file PATH` | Password-mode storage, no-prompt claim/unlock (same semantics as the OCI path). |
| `--log-key KEY_ID` / `--generate-log-key KEY_ID` | Bind org log-encryption to this deploy (see day-2 for the logs ceremony). |

## 3. Get the SSH command

```bash
enclava template ssh-command --name shell --wait
# ssh -p 20051 user@relay.enclava.me   (port allocated by PaaS)
```

The command is **fail-closed validated**: CLI and console compare the rendered command
against the stored stable endpoint, and a malformed command, wrong host, padded value,
or unexpected username is rejected rather than shown. Use `--json` when scripting.

## 4. Check state

```bash
enclava status --app shell
```

Hosted apps have no local `enclava.toml`. For commands that support `--app` — such as
`status`, `logs`, `unlock`, `recover`, and `destroy` — pass the hosted name explicitly.
Not every app operation supports that flag: `config set` and `auto-unlock` always load
local `enclava.toml`, so they cannot address hosted apps; generic `deploy` is not a
hosted restart command.

## Notes and limits

- The user supplies the SSH public key plus owner choices such as the storage password
  or log key. Platform-managed endpoint and relay credentials stay server-side.
- Password-mode template apps follow the same ownership and recovery state machine as
  manual apps. After a restart, run `enclava unlock --app shell`; back up the mnemonic
  after the first claim.
- Unlock mode is **fixed by the template definition** — `debian-ssh-frp` is password
  mode, with no deploy-time or current post-deploy switch to auto-unlock.
- SSH lands in the workload's `web` container — a minimal device namespace. Absent
  `/dev/sev*` or sparse `lsblk` output there is a namespace artifact, not a fault;
  `enclava status` is the authoritative state.
