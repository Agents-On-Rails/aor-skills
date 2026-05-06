# AOR Bundle — CI Signing Contract

Publisher-side commitments for producing signed AOR bundle releases. Read
alongside the [Verifying this release](./README.md#verifying-this-release)
section in the bundle README — that section covers the *consumer* side; this
document covers the *publisher* side. The two sides intersect at the
`exporterPublicKey` field embedded in each `<tarball>.attestation.json`.

This contract is the artifact a regulated-industry consumer's QMS auditor
or a downstream maintainer can cite when evidencing how the publisher's
signing process is structured.

## Contents

- [§1. Versioning policy](#1-versioning-policy)
- [§2. Key custody policy](#2-key-custody-policy)
- [§3. CI signing contract — workflow step requirements](#3-ci-signing-contract--workflow-step-requirements)
  - [§3.1 Canonical secret name](#31-canonical-secret-name)
  - [§3.2 Write-temp-and-export pattern](#32-write-temp-and-export-pattern)
  - [§3.3 What the workflow MUST NOT do](#33-what-the-workflow-must-not-do)
  - [§3.4 What the workflow MUST verify before signing](#34-what-the-workflow-must-verify-before-signing)
  - [§3.5 Self-hosted runner considerations](#35-self-hosted-runner-considerations)
- [§4. GitHub Releases publish-step contract](#4-github-releases-publish-step-contract)
- [§5. What this contract does NOT cover](#5-what-this-contract-does-not-cover)
- [§6. Cross-references](#6-cross-references)

> **Scope:** publisher commitments for the canonical AOR-bundle release
> lane operated from this repository. Forks producing their own signed
> bundles MUST replace this entire document (different secret name,
> different key custody, different repo conventions) before re-distributing
> a forked bundle — the contract herein names the canonical repository's
> AOR_EXPORTER_PRIVATE_KEY_PEM secret and policies, which a fork has not
> adopted. Shipping the canonical contract unchanged inside a forked
> bundle is misleading to downstream consumers (ARCH-003 from the
> 9-SME convergence panel).

---

## §1. Versioning policy

- **Tag pattern.** Each release is tagged `aor-bundle-vX.Y.Z`, where
  `X.Y.Z` is a semver triple optionally followed by a prerelease label
  (`-rc.1`, `-beta.2`, etc.). The validation regex below is enforced
  by `release.mjs` against the `--bundle-version` argument AFTER the
  canonical `aor-bundle-` prefix has been stripped from the tag —
  e.g. tag `aor-bundle-v1.0.0-rc.1` produces bundle-version `v1.0.0-rc.1`
  which is what the regex matches. The release-lane workflow performs
  this prefix-strip; see §3.2.

  ```
  /^v\d+\.\d+\.\d+(-[A-Za-z0-9.-]+)?$/
  ```

- **Tag immutability.** Once pushed to the canonical repo, a release tag
  is never re-pointed. A defective release is superseded by a new
  patch-version tag (e.g., `aor-bundle-v1.0.1`) with a row appended to
  `EXPORT-LOG.md` noting the supersession reason.

  This is a **publisher commitment backed by GitHub tag-protection
  rules** on the canonical repository (force-push and deletion of
  `aor-bundle-v*` tags are blocked at the repository-settings layer).
  GitHub permits force-push of tags by default, so the immutability
  property is *not* a property of git itself — it is enforced by the
  protection rule. A consumer evidencing the chain SHOULD additionally
  rely on the per-release `tarballSha256` in attestation, which would
  detect any successful re-point at verification time even if the
  protection rule were ever circumvented.

- **Tarball / attestation naming.** The release lane produces two
  artifacts per tag:

  | Artifact | Filename pattern |
  |---|---|
  | Tarball | `aor-bundle-vX.Y.Z.tar.gz` |
  | Attestation | `aor-bundle-vX.Y.Z.tar.gz.attestation.json` |

- **`frameworkVersion` and `frameworkSha` in attestation.**
  `frameworkVersion` is the framework `package.json` version field at the
  pinned `parentSha`. `frameworkSha` is the framework commit SHA the
  bundle was built from. Both fields are part of the signed payload.

  In the canonical release lane, `parentSha == frameworkSha` by
  construction (the workflow does not pass `--parent-sha`, so
  `release.mjs` defaults to the framework HEAD). Divergence is only
  possible in operator-driven local builds and is observable in the
  release body (both fields shown separately).

  The signed `frameworkSha` **transitively authenticates
  `framework-manifest.json`** at that commit: `framework-manifest.json`
  is itself committed to the framework repo at `frameworkSha`, so the
  consumer's verification of the signed `frameworkSha` plus a
  `git show <frameworkSha>:framework-manifest.json` (or an equivalent
  source-repo lookup) yields a verified manifest of all 270+
  hash-tracked framework files, including `bundles/aor/tools/**` and
  `bundles/aor/export.yml` (per ADR-011 D7, closed in this arc).

- **`exportedAt` is derived, not wall-clock.** The `exportedAt` field in
  attestation and `.bundle-metadata.json` is the commit timestamp of
  `parentSha`, NOT `Date.now()`. This is what makes ADR-011 D5
  reproducibility work — building the same `parentSha` twice yields
  byte-identical tarballs, including identical attestation payloads when
  signed with the same key.

- **Prerelease semantics.** `-rc.X` and `-beta.X` tags ship signed by
  the canonical key (the contract does not weaken for prereleases) but
  consumers SHOULD treat them as testing-only artifacts and not deploy
  them in regulated environments without their own validation.

---

## §2. Key custody policy

- **Algorithm.** The canonical exporter key is an **ED25519** private key
  in **PKCS#8 PEM** format. ED25519 was selected for: deterministic
  signatures (no nonce randomness), small signatures (64 bytes), wide
  library support (Node `crypto` natively, OpenSSL ≥1.1.1, Python
  `cryptography`).

- **Storage.** The private key is stored as a GitHub Actions
  **repository secret** on the canonical AOR repository. It is never
  committed to the repository, never written to a persistent filesystem
  on a runner, and never echoed in workflow logs.

- **Access scope.** The secret is readable only by GitHub Actions
  workflows configured on the canonical repository. Among humans, only
  repository administrators can rotate or revoke the secret. The
  publisher commits to maintaining access on a least-privilege basis.

- **Rotation cadence.** The canonical key is rotated **at least
  annually**, AND immediately on any of:
  - Suspected compromise (e.g., a secret-leak alert).
  - Departure of an administrator who had administrator-level access.
  - Algorithm or library deprecation guidance from upstream
    (Node `crypto`, OpenSSL).

- **Public-key carry-forward.** When the key rotates, the **previous
  public key is preserved** in a publisher-maintained `PUBLIC-KEYS.md`
  (or equivalent) so consumers can continue to verify older releases.
  `PUBLIC-KEYS.md` will be created at first key rotation, with a row
  per historical key (fingerprint, valid-from / valid-until window,
  rotation reason). It is NOT shipped pre-populated because no
  rotation has occurred yet — the canonical first release is signed
  by a single key whose public form is embedded in every release's
  `<tarball>.attestation.json` `exporterPublicKey` field. The contract
  is: a release signed at time `T` remains verifiable as long as the
  corresponding public key remains published, even after the private
  key has been rotated out of active use.

- **No private-key recovery.** There is no escrow, no recovery, no
  social-key-recovery scheme. If the canonical key is lost (not just
  compromised), the publisher rotates to a new key and notes the
  break in `EXPORT-LOG.md`. Releases between the loss and the next
  signed release MUST be rebuilt and re-signed with the new key.

- **Compromise response.** On confirmed compromise the publisher
  commits to the following timeline:
  1. **Within 24 hours of confirmation:** revoke the affected secret
     `AOR_EXPORTER_PRIVATE_KEY_PEM` in GitHub repository settings.
  2. **Within 72 hours of confirmation:** rotate to a new ED25519 key,
     update the GitHub Actions secret, sign and publish a fresh
     advisory release.
  3. **Within 5 business days of confirmation:** append a row to
     `PUBLIC-KEYS.md` marking the compromised key with the date of
     revocation; append annotations to `EXPORT-LOG.md` for any release
     signed within the suspected-compromise window; publish an advisory
     referencing the affected release range.
  4. **Consumer notification channels:** the GitHub release notes for
     the next release after the compromise (pinned for ≥30 days), the
     repository's `SECURITY.md` (if present), and any operator-published
     security mailing list. Regulated-industry consumers SHOULD subscribe
     to GitHub release notifications on the canonical repo so they are
     alerted within minutes of the advisory release going live.

---

## §3. CI signing contract — workflow step requirements

The release-lane workflow (delivered by task #9 as
`.github/workflows/bundle-release.yml`) MUST follow these rules. They are
the minimum-acceptable surface for the canonical signing lane.

### §3.1 Canonical secret name

The GitHub Actions secret containing the PEM-encoded private key is
named:

```
AOR_EXPORTER_PRIVATE_KEY_PEM
```

The secret value is the **PEM contents**, not a file path. The workflow
materializes the temp file at runtime and exports its path via
`AOR_EXPORTER_KEY_PATH` (the env var consumed by `release.mjs`).

### §3.2 Write-temp-and-export pattern

The release step MUST follow this canonical pattern. Note that
`RELEASE_TAG` (e.g. `aor-bundle-v1.0.0`) carries the canonical
`aor-bundle-` prefix, while `release.mjs --bundle-version` validates
against the bare-semver regex (`/^v\d+\.\d+\.\d+(-[A-Za-z0-9.-]+)?$/`,
§1) — the prefix MUST be stripped before passing. The canonical
release lane also passes `--no-log` because EXPORT-LOG.md is updated
by a separate post-release maintainer commit (auto-commit-back is a
tracked follow-up; see §6).

```yaml
- name: Sign and release
  env:
    AOR_EXPORTER_PRIVATE_KEY_PEM: ${{ secrets.AOR_EXPORTER_PRIVATE_KEY_PEM }}
    RELEASE_TAG: ${{ github.ref_name }}
  run: |
    set -euo pipefail
    BUNDLE_VERSION="${RELEASE_TAG#aor-bundle-}"
    KEY_FILE="$(mktemp -p "${RUNNER_TEMP:-/tmp}" aor-key.XXXXXXXX.pem)"
    chmod 0600 "$KEY_FILE"
    printf '%s' "$AOR_EXPORTER_PRIVATE_KEY_PEM" > "$KEY_FILE"
    trap 'shred -u "$KEY_FILE" 2>/dev/null || rm -f "$KEY_FILE"' EXIT
    export AOR_EXPORTER_KEY_PATH="$KEY_FILE"
    npm run bundle:release -- --bundle-version="${BUNDLE_VERSION}" --no-log
```

The actual lane in `.github/workflows/bundle-release.yml` adds a
preceding fail-closed check that `AOR_EXPORTER_PRIVATE_KEY_PEM` is
non-empty (CI-SIGNING-CONTRACT MUST: refuse to release without
signing). That check is part of the contract — it is omitted from the
snippet above only for brevity.

Mandatory properties of this pattern:
- **`mktemp` with explicit dir.** Use `$RUNNER_TEMP` (set by GitHub
  Actions to a per-job ephemeral directory). Fallback `/tmp` is
  acceptable on hosted runners; self-hosted runners SHOULD verify
  `$RUNNER_TEMP` is on tmpfs (see §3.5).
- **`chmod 0600` before write.** Ensures the file is unreadable to other
  users on a multi-tenant runner during the brief window before the
  process exits and `trap` cleans up.
- **`printf '%s'` not `echo`.** `echo` may interpret `\` escapes
  depending on shell; `printf '%s'` writes the secret literally.
- **`trap … EXIT`.** Ensures the temp file is removed even if the
  release step fails. `shred -u` first (defense in depth on disks that
  do not honor `unlink` immediately), `rm -f` fallback for environments
  without `shred`.

### §3.3 What the workflow MUST NOT do

**Process visibility** (no leak via running-process state):
- MUST NOT pass the PEM content as a CLI argument to any tool. CLI args
  are visible in `ps` output to all users on the runner.
- MUST NOT echo `$AOR_EXPORTER_PRIVATE_KEY_PEM`, `$AOR_EXPORTER_KEY_PATH`,
  the temp `$KEY_FILE` path, or the file contents in any step. GitHub
  Actions auto-redacts secret values in logs but redaction is best-effort
  and can be defeated by encoding; the path itself is not redacted at
  all and should be treated as defense-in-depth sensitive.
- MUST NOT enable `set -x` (shell trace) in any step that has the secret
  in environment.

**Filesystem persistence** (no leak via durable storage):
- MUST NOT write the PEM content to any path under `$GITHUB_WORKSPACE`,
  `$HOME`, or any other directory that is preserved between jobs or
  cached.
- MUST NOT use `actions/upload-artifact` to upload the temp file. The
  signed artifacts (tarball + attestation) do not contain the private
  key; only those are uploaded.
- MUST NOT cache the temp directory across jobs (e.g., via
  `actions/cache`).

**Trigger surface** (no leak via untrusted-event execution):
- MUST NOT use `pull_request_target` triggers on any workflow that has
  access to `AOR_EXPORTER_PRIVATE_KEY_PEM`. `pull_request_target` runs
  against the base ref with full secret access — the canonical
  poison-PR attack vector.
- MUST NOT use `workflow_run` triggers that consume artifacts from PR
  workflows (artifact-injection cross-event leak).
- MUST NOT permit `repository_dispatch` or `workflow_dispatch` on the
  release workflow without explicit input validation that asserts the
  triggering ref is a canonical `aor-bundle-v*` tag reachable from the
  default branch.

**Cross-fork hygiene**:
- MUST NOT enable the secret on forks via repository settings ("Send
  secrets to workflows from forks"). The canonical key MUST be scoped
  to canonical-repo workflows only.

### §3.4 What the workflow MUST verify before signing

Before invoking `bundle:release`, the workflow MUST:
- Run `npm run bundle:verify-manifest` (manifest-completeness +
  presence-check). `release.mjs` already runs these as a pre-flight, but
  doing them as a separate workflow step gives clearer error reporting
  and fails faster.
- Run `npm run bundle:smoketest` (structural + behavioral when the
  task #9 host-CLI lane is wired).
- Run `npm run bundle:test:reproducibility` against a pinned
  `parentSha`.

If any of these fail, the workflow MUST NOT proceed to the signing step.
This is a defense-in-depth contract: the publisher commits to never
signing a release that has not passed the bundle invariants.

### §3.5 Self-hosted runner considerations

If the canonical signing lane runs on a self-hosted runner:
- `$RUNNER_TEMP` MUST be on **tmpfs** (RAM-backed). Verify with `mount`
  in a pre-flight step.
- Disk-backed temp directories MUST NOT be used; they survive runner
  restarts and may be backed up by the runner host's snapshot policy.
- Runner host MUST have `shred` available (or the cleanup step MUST be
  adapted to whatever secure-delete primitive is available on the host).

GitHub-hosted runners satisfy all of these by construction.

---

## §4. GitHub Releases publish-step contract

After signing succeeds, the release lane publishes to GitHub Releases.

### §4.1 Trigger

Tag push matching `aor-bundle-v*` on the canonical repository's default
branch. Tag-push from a non-default branch is NOT a release trigger.

### §4.2 Artifacts attached to each release

| Artifact | Required? | Notes |
|---|---|---|
| `aor-bundle-vX.Y.Z.tar.gz` | YES | The bundle tarball |
| `aor-bundle-vX.Y.Z.tar.gz.attestation.json` | YES | Detached attestation |
| `aor-bundle-vX.Y.Z.tar.gz.sha256` | NO (optional convenience) | Plain `<sha256>  <filename>` text file |

Attaching anything else (e.g., expanded bundle contents, debug logs)
is a contract violation — consumers expect exactly these two files
under canonical names.

### §4.3 Release body

The release body MUST include, at minimum:

- `frameworkSha` (full 40-char SHA).
- `parentSha` (typically equal to `frameworkSha`; differs only when
  built from a non-HEAD ref).
- `tarballSha256` (full 64-char hex; the prefix-form used in
  `EXPORT-LOG.md` is for readability, not for the Releases page).
- `exportedAt` (ISO-8601 UTC).
- Probe versions (Copilot CLI version sourced from the canonical probe
  fixture).
- A link to the bundle README's
  [Verifying this release](./README.md#verifying-this-release) section.

The release body **MUST NOT include** any field that resolves to a
natural person — no `${{ github.actor }}`, no `triggering_actor`, no
maintainer email, no commit author name. The release-body composition
in `bundle-release.yml` derives all values from the attestation JSON
(deterministic, builder-anonymous) and the immutable git context
(repository name, tag name, framework SHA), none of which are PII.
This invariant prevents a future workflow edit from silently
introducing builder identity into a publicly-archived release page.

### §4.4 Release name

Release name and tag both equal `aor-bundle-vX.Y.Z`. Do not customize.

---

## §5. What this contract does NOT cover

- **Key generation procedure.** The canonical key is generated by an
  administrator outside CI, on an air-gapped or otherwise trusted
  workstation. The procedure is operator-internal.
- **Trust bootstrap.** Consumers obtain the publisher's authoritative
  ED25519 public key fingerprint out-of-band — from the source repo's
  README, a signed git tag, the publisher's website over HTTPS, or
  another trusted channel. The bundle and the attestation alone DO NOT
  bootstrap trust (see README §"Verifying this release"). The publisher
  commits to maintaining one or more trusted publication channels for
  the public key.
- **Verifier-side procedure.** Documented in the bundle README (
  [Verifying this release](./README.md#verifying-this-release)). The
  publisher commits to keeping that section accurate against any
  changes to `attest.mjs` or the canonical algorithm.

---

## §6. Cross-references

- `bundles/aor/README.md` §"Verifying this release" — the consumer-side
  counterpart to this document.
- `bundles/aor/tools/release.mjs` — the canonical release entry point.
  `--help` text references this contract for the `AOR_EXPORTER_KEY_PATH`
  environment variable.
- `bundles/aor/tools/lib/attest.mjs` — signing implementation,
  `canonicalAlgorithm` field semantics.
- `bundles/aor/EXPORT-LOG.md` — append-only release log; on key rotation
  or compromise, the log carries the corresponding annotations.
- `docs/adr/011-bundle-export-pipeline-architecture.md` — bundle export
  pipeline architecture; D5 reproducibility, D6 framework-canonical
  export, D7 hash-tracking (closed in commit `d6a5c55` per task #22).
- `docs/adr/009-aor-bundle-non-divergence.md` — bundle/framework
  non-divergence governance.
