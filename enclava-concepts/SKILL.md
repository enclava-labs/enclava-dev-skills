---
name: enclava-concepts
description: Explain how Enclava and confidential computing work — TEEs, SEV-SNP, remote attestation, proof bundles, encrypted storage, who can read what, and honest protection limits. Use for conceptual or trust questions ("can the operator see my data?", "what does attestation prove?", "what's a TEE/SEV-SNP/proof bundle?", "is my app encrypted?"). For running an appraisal use enclava-verify; for deployment and lifecycle commands use enclava.
compatibility: Enclava platform and CLI v0.1.x
---

# Enclava and confidential computing — the accurate mental model

Agents answering from general knowledge get the *specifics* wrong here (who holds
which key, what attestation covers, what "locked" means). This file carries the
load-bearing facts; the full treatments live in the developer docs
(https://docs.enclava.dev — concepts, threat model, glossary).

## The one-paragraph version

Enclava runs ordinary container images inside a **Kata confidential-containers guest**
backed by **AMD SEV-SNP**: a hardware-isolated VM whose memory is encrypted with keys
held by the CPU itself. The cluster can *schedule and operate* the workload — start
it, stop it, see its metadata — but cannot *read its memory* or silently alter the
trusted startup path without detection. Secrets, storage keys, and TLS material are
released to the guest only after the platform verifies what is actually booting.

## The facts that change answers from wrong to right

**1. Protection is against the infrastructure, not against the app.** In scope:
the operator/host reading workload memory, grabbing secrets by observing the pod,
substituting code via mutable tags, or releasing keys before checks pass. Out of
scope: bugs in the app's own code, data the app intentionally discloses, a malicious
but *properly signed* image, and denial of service by the infrastructure. A TEE is an
isolation layer around an app you still have to engineer and trust.

**2. Attestation proves *what* booted, not *that it's correct*.** The TEE produces
signed evidence (an SNP report bound to a fresh nonce) about the guest's identity,
launch measurement, and TCB. Verifiers compare it against expected values. The
launch measurement pins the *guest launch state*; the OCI image is appraised
separately — CAP checks the image digest and its cosign/Sigstore/provenance
bindings on their own. The complete appraisal combines both: guest measurement
plus signed-artifact checks establish that the exact code you authorized booted —
not that the code is bug-free or honest.

**3. The trust chain has two gates.** (a) *Image identity*: a custom image must carry
portable cosign signature and bound provenance material (typically keyless via GitHub
OIDC→Fulcio→Rekor), and the signer identity is pinned per app — so only authorized
identities can publish deployable code. (b) *Runtime measurement*: attestation binds
the booted guest to the expected measurement, policies, and configuration. Both must
pass before secrets or state are released; failure blocks startup (fail-closed by
design).

**4. Storage is encrypted with owner-controlled key material.** Persistent storage is
a LUKS volume inside the guest (app data and TLS state are separate encrypted volumes).
At password-mode claim, the TEE generates a random 32-byte **owner seed**. Argon2id and
HKDF derive a password wrapping key that encrypts that seed; the volume keys derive
from the seed itself. The control plane and KBS never receive the plaintext password or
seed. A BIP39 **recovery mnemonic** is an independent representation of the same seed,
not a copy of the password. Auto-unlock stores another encrypted seed envelope that KBS
releases only to a guest passing attestation (**KBS-attestation-gated wrapping**, not
VMPCK hardware sealing) for unattended restarts. A password-mode app with no usable
password or recovery copy has no platform backdoor; auto-unlock is not a substitute for
an owner backup.

**5. The live report is nonce-fresh; supporting material may be cached.** A live
verification fetches a proof bundle from
`/.well-known/confidential/proof-bundle?nonce=<random>` — the SNP report inside is
bound to that nonce, so an old report cannot satisfy the new challenge. Endorsements
such as the VCEK may be cached; their signatures, chain to the pinned AMD ARK, TCB
binding, and freshness policy are appraised separately. (See the `enclava-verify`
skill for the operational side.)

**6. The operator stays in the threat model for availability and metadata.** The
platform/host controls scheduling, can restart or destroy the workload, and can
observe traffic shape and metadata (sizes, timing, endpoints — mitigate with TLS
like anywhere else). What it cannot do is read content: memory, secrets, unlocked
storage, config values. "Encrypted and attested" removes *content* access, not
*authority* over the workload's existence.

**7. Fail-closed is a feature.** When any check can't complete — signature mismatch,
stale revocation, missing endorsement, unverified boot — startup or verification
fails rather than degrading. If a user reports a failure of this shape, the right
move is to read the named cause (e.g. `tee_error` via `enclava status`), not to hunt
for a bypass.

**8. Trusted computing base is named, not magic.** Trust ultimately rests on AMD's
attestation roots and SEV-SNP TCB, the measured guest stack, the pinned platform
release and policy authorities, KBS appraisal, the artifact identities you authorize,
and your owner credentials. Independent verification matters: target-presented keys,
measurements, and identities become trust anchors only after you authenticate them
through a channel the target cannot control.

## Vocabulary quick-reference

- **TEE** — trusted execution environment; isolated execution context that can produce
  attestation evidence.
- **SEV-SNP** — AMD Secure Encrypted Virtualization w/ Secure Nested Paging; the
  hardware memory-encryption + attestation feature Enclava targets.
- **Kata confidential containers** — the VM-backed container runtime model.
- **Remote attestation** — proving what code/config runs inside a TEE via signed
  evidence vs expected measurements.
- **Trustee/KBS** — key broker service releasing protected resources only after
  attestation and policy checks pass.
- **Deployment descriptor** — signed CAP document binding app, deploy ID, policy,
  image, expected runtime claims, and deployer identity.
- **`enclava-init`** — in-guest startup sidecar: verifies the chain, opens encrypted
  volumes, then writes `/run/enclava/init-ready`; `enclava-wait-exec` holds the app
  until then.
- **`tee_error`** — startup failure reason surfaced by `enclava status`.
- **Org keyring** — org-level trust material defining which deploy keys are authorized.

## Answering common doubts (calibrated)

- *"Can Enclava/the operator read my app's data?"* — In password mode, no: memory is
  hardware-encrypted; a random owner seed derives the storage keys and is stored only
  in password-wrapped form. Runtime config travels directly from the client to the
  attested TEE rather than being persisted as control-plane plaintext. Operators can
  still see metadata and control scheduling.
- *"What if AMD/Enclava pushes firmware?"* — Firmware TCB and launch measurement are
  separate policy dimensions. `minimum_tcb` is a component-wise lower bound, so a TCB
  increase can pass without changing the measurement; a downgrade below the minimum
  fails. Accept new values only after checking the release and your policy.
- *"Is my app safe if the image is signed?"* — Signing pins *who* published it, not
  whether it's good. A malicious image signed by an identity you authorized is out of
  scope — review what you pin.
- *"Where do secrets live?"* — Delivered directly from the client to the TEE after
  attestation; the control plane does not persist or return their plaintext.
  `enclava config get` lists key names only.
- *"What does the proof bundle prove?"* — Its nonce-bound SNP report identifies the
  guest launch measurement and TCB for that live challenge. Policy checks separately
  bind the origin/TLS channel, authorized image artifact and signer/provenance,
  platform release, and deployment identity. It does not prove code correctness.

When a question goes deeper than these, point at the docs' concepts and threat-model
pages rather than improvising — the honest edges (DoS, metadata, signed-but-malicious
code) are documented there deliberately.
