# Journey: deploy a custom OCI image (manual CAP path)

For teams running their own container image. If a hosted template already fits the
workload, that path (`hosted-template.md`) is much less work.

## Prerequisites

- An Enclava account and org (`enclava login`; hosted registration is browser-based,
  while standalone CAP exposes `signup`).
- The `enclava` CLI on PATH.
- Docker + a registry (examples use GHCR) and a `Dockerfile` in the repo root.

## 1. Scaffold

```bash
enclava init        # prompts for name/port when interactive; --app-name/--port run it
                    # non-interactively (piped stdin takes the directory name and
                    # EXPOSE/3000). Validates the app name before writing anything.
enclava prepare     # same scaffolding; --yes skips the update-existing-files
                    # confirmation (a non-TTY run without --yes still exits
                    # "not a terminal"). In CI: run it once, commit both files,
                    # and don't re-run over them.
```

Resulting `enclava.toml` (edit as needed):

```toml
[app]
name = "my-app"
port = 8080
command = ["/usr/local/bin/app"]

[storage]
paths = ["/data"]        # in-container paths needing persistent encrypted storage
size = "5Gi"
tls_size = "2Gi"

[unlock]
mode = "password"        # MUST be password for the first deploy (see spine)

[resources]
cpu = "1"
memory = "1Gi"

[health]
path = "/health"
interval = 30
timeout = 5
```

There is no "no storage" default: with the `[storage]` section absent, CAP still
defaults `paths = ["/data"]`. A stateless app must opt out explicitly:

```toml
[storage]
paths = []
```

## 2. Meet the image contract

The workload runs on a **read-only rootfs** inside the confidential guest. The
relevant image contract and optional scaffolding metadata are:

| Requirement | Why |
| --- | --- |
| `EXPOSE <port>` | Optional scaffolding metadata — lets `enclava init` suggest `app.port`. Not a runtime requirement: the container port and readiness probe use `app.port` from the signed config. |
| `VOLUME ["<path>", …]` for **every** `storage.paths` entry | `enclava-init` bind-mounts each path into the encrypted volume; without the volume declaration the bind fails with `Read-only file system (os error 30)`, surfaced as `tee_error` in `enclava status`. |
| Digest pinning at deploy | Tags are resolved at `create`; `deploy` requires `image@sha256:<digest>`. |

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY server.py .
EXPOSE 8080
VOLUME ["/data"]
CMD ["python3", "/app/server.py"]
```

## 3. Build and sign (cosign, keyless via GitHub Actions)

CAP requires both portable cosign signature material and a bound provenance manifest.
The intended path is **keyless signing with GitHub OIDC**: GitHub issues a short-lived
Fulcio certificate, cosign signs the digest, Rekor records it, and GitHub publishes
build provenance. No signing key to manage.

The starter workflow from `init`/`prepare` builds, pushes, signs, and attests on each
push to `main`. Older generated workflows may still pin `cosign-installer@v3`; replace
that step with the pinned v4.1.2 commit below. This is the minimum complete flow:

```yaml
name: Build signed image

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write
  attestations: write

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - uses: docker/setup-buildx-action@v3

      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}

      - name: Install cosign
        # v4.1.2 — immutable pin; installs cosign 3.x
        uses: sigstore/cosign-installer@6f9f17788090df1f26f669e9d70d6ae9567deba6

      - name: Sign image
        run: >-
          cosign sign --yes
          ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}@${{ steps.build.outputs.digest }}

      - name: Attest build provenance
        uses: actions/attest-build-provenance@v2
        with:
          subject-name: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          subject-digest: ${{ steps.build.outputs.digest }}
          push-to-registry: true
```

The workflow prints a ready-to-run `enclava deploy --image <image>@<digest>` line at
the end. The digest also appears on the GHCR package page, or via
`docker buildx imagetools inspect <image>:<tag>`. If the cluster pulls anonymously,
make the GHCR package public.

**The signer subject** (pin this next) is the GitHub Actions OIDC identity — note that
the **workflow filename is part of it**:

```
https://github.com/<org>/<repo>/.github/workflows/<workflow>.yml@refs/heads/<branch>
```

Confirm it locally before pinning:

```bash
cosign verify \
  --certificate-identity https://github.com/<org>/<repo>/.github/workflows/<workflow>.yml@refs/heads/main \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/<org>/<repo>@sha256:<digest>
```

(Keep the workflow filename stable afterwards — renaming it changes the identity.)

## 4. Create the app (pin identity + image)

```bash
enclava create \
  --image ghcr.io/<org>/<repo>:latest \
  --signer-subject https://github.com/<org>/<repo>/.github/workflows/<workflow>.yml@refs/heads/main
```

`create` registers the app from `enclava.toml`, resolves the tag to a digest, and pins
the signer identity. `--signer-issuer` defaults to the GitHub Actions OIDC issuer —
set it only for a different OIDC provider. If you skip `--signer-subject` here, set it
before deploying with `enclava signer set <subject>` (first set needs no confirmation
token; later changes go through `signer rotate`, which requires a platform-issued
confirmation token).

## 5. Deploy

```bash
enclava deploy \
  --image ghcr.io/<org>/<repo>@sha256:<digest> \
  --storage-password-file "$HOME/.enclava/my-app-storage-password"
```

- `--storage-password-file` makes the first deploy **claim ownership and unlock
  without a password prompt** (create the file with tight permissions; never pass the password
  inline) — the session must still be attended: fully unattended (no terminal stdin)
  claims are refused until response-loss recovery exists. Without the flag,
  `deploy` prompts for the password, or the user runs
  `enclava claim` separately.
- The recovery mnemonic is persisted to the protected local keystore so `enclava key backup`
  covers it; the mnemonic is never printed, so `--no-store-mnemonic` is rejected before
  the claim (an unpersisted mnemonic would be permanently lost).
- Runtime config: `--set KEY=VALUE`, or `--set-file KEY=PATH` to keep a secret out of
  process arguments.

## 6. Verify and back up

```bash
enclava status     # expect: Status: running, TEE: unlocked, Unlock: unlocked
enclava key backup --out "$HOME/.enclava/my-app-recovery.json"
```

If deploys will be unattended (restarts without a human), consider
`enclava auto-unlock enable --image <image>@sha256:<digest>` — but only after the
password-mode first deploy succeeded.

## Common failures on this path

| Symptom | Fix |
| --- | --- |
| `tee_error: … (os error 30)` read-only bind | Missing `VOLUME` for a `storage.paths` entry — add it, redeploy. |
| `portable_verification_material_unavailable` | CAP could not assemble portable signature/provenance material. One common cause is a cosign 2.x legacy `.sig` object without portable DSSE material; re-signing with the generated cosign 3.x workflow fixes that case. Registry/referrer errors, missing provenance, malformed bundles, and size limits share this code — inspect both signature and provenance objects. |
| Deploy rejected on signature/identity | Signer subject doesn't match what signed the image — check workflow filename/branch in the subject, re-run the `cosign verify` above. |
| First deploy in auto mode fails | Expected — current CLIs reject it up front at `create`; older CLIs fail at boot after ~a minute. Either way: `[unlock] mode` must be `password` for the first deploy. |
| `Status: drifted`, later deploys no-op | `enclava destroy --app <name> --force`, then `create` + `deploy` from clean (destroy first — the app name and escrow interact; see the day-2 reference before recreating with the same name). |
