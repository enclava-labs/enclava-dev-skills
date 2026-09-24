---
name: enclava-verify
description: Independently verify (appraise) a running Enclava app's trust properties from outside, or verify a saved proof bundle as historical evidence. Use when the user wants to verify/audit/attest an enclava origin or deployment, check a proof bundle, build or review a trust policy, ask "is this app really running in a TEE / is this the code I expect", or do third-party due diligence on an enclava-hosted service.
compatibility: enclava CLI v0.1.x (verify, describe)
---

# Independent verification with `enclava verify`

Verification answers an auditor's question: *should I trust this running origin?* It
fetches fresh cryptographic evidence from the target and appraises it against a
**policy you obtained independently of the target**. It is deliberately separate from
operating an app (that's the `enclava` skill).

## The two commands

```bash
enclava describe <https-origin-or-bundle-path> [--policy-skeleton PATH] [--save-bundle PATH] [--json]
enclava verify <https-origin> --policy trust-policy.json [--save-bundle PATH] [--json]
enclava verify --bundle saved-bundle.bin --policy trust-policy.json [--json]   # historical evidence
```

- **`describe`** observes what a target *contains* — no appraisal, no judgment. Safe
  to run against anything.
- **`verify`** appraises: every check in the policy must pass. `--json` emits the
  canonical result schema (for CI); `--save-bundle` captures the exact live bytes as
  historical evidence (the bundle format is the versioned media type
  `application/vnd.enclava.proof-bundle.v1`).

## What a live verification does

The CLI fetches `GET <origin>/.well-known/confidential/proof-bundle?nonce=<nonce>`
with a fresh 32-byte random nonce. The TEE-side service returns a proof bundle whose
attestation report is **bound to that nonce** — a replayed or prefabricated bundle
can't match it. The bundle then gets checked against the policy: report signature via
the AMD VCEK, endorsement chain up to the pinned ARK, launch measurement, TCB levels,
guest policy, TLS leaf binding to the origin, image digest and cosign signature
policy, platform release, runtime class and sidecar digests, deployment identity,
descriptor measurement, revocation freshness.

Key properties worth explaining to users:

- **Freshness by nonce**: each live verification is a new challenge; old evidence
  can't be served again. A saved `--bundle` is *historical* evidence of a past state,
  clearly distinct.
- **Identity, not correctness**: a passing verification identifies the guest launch
  state and separately validates the authorized image artifact, configuration, and
  deployment bindings. It does not show that the app is bug-free or honest.
- **Fail-closed**: if the chain can't be completed (bad signature, stale revocation,
  missing endorsement), verification fails rather than downgrades. That is the
  trustworthy behavior.

## Authoring a trust policy (TOFU done carefully)

There is no global list of "good" measurements — you pin what *you* expect. The
workflow:

1. `enclava describe <origin> --policy-skeleton draft.json` — emits a
   `enclava-trust-policy-v1` skeleton from observed values (trust-on-first-use).
2. **Edit the draft before trusting it.** The target supplies every observed value,
   so independently authenticate all of them rather than approving the generated file
   wholesale:
   - AMD/runtime claims: `amd.allowed_measurements`, `amd.minimum_tcb`,
     `amd.guest_policy_mask`/`guest_policy_value`, and
     `amd.trusted_ark_sha256` (compare the ARK with AMD's published root).
   - Expected target and deployment: origins, image digests, runtime classes,
     attestation-proxy and Caddy digests, platform-release versions, organization IDs,
     and application IDs. Compare them with release records, builds, and identities
     obtained outside the target.
   - Target-presented authority: `trusted_org_keyring_sha256` and
     `trusted_policy_signing_pubkeys`. Copying these unverified inherits whatever
     authority the target selected.
   The skeleton cannot infer your Sigstore trust policy: every Sigstore value is a
   `REPLACE_WITH_…` placeholder. Populate the Fulcio roots/hashes, Rekor key,
   certificate identity, OIDC issuer, source repository, workflow ref, and provenance
   builder ID from independent sources.
3. Record the final policy **via a channel the target cannot influence** (e.g. checked
   into *your* repo) — then verify with it. Note the limits of that step: it preserves
   the policy after selection but authenticates nothing; the values are only as
   trustworthy as the independent verification in step 2 made them.
4. Iterate: `enclava verify <origin> --policy p.json --json` reports each check;
   adjust pins consciously, never by blanket-copying observed values. TCB and
   measurement are **separate policy dimensions**: firmware updates can change the
   reported TCB (`minimum_tcb` is a component-wise lower bound), while guest/runtime
   changes change the launch measurement — a TCB bump does not by itself imply a new
   measurement (and vice versa).

A pinned policy encodes those launch, runtime, artifact, identity, and authority
expectations plus revocation limits and required checks. Use a known-good example only
for schema shape; source every trust value independently.

## Practical guidance

- Verify from a machine/network *you* control — the nonce protects freshness, but the
  TLS connection to the origin should also be genuinely yours.
- Some environments serve Let's Encrypt **staging** certs on tenant app origins (a
  platform toggle); browsers/CLIs with strict roots will flag them. If verification
  fails only at TLS-leaf chain-of-trust, check which LE environment the origin uses
  before concluding anything worse.
- For CI or repeat audits: `verify --json` and gate the step on the result (a failed
  appraisal must fail the check); keep the policy + saved bundles under version
  control as your evidence trail.
- `describe` first, always, when facing an unknown target — it tells you what there is
  to pin before you pin anything.
