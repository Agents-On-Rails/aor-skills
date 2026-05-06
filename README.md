# aor-bundle

Portable Agent Skills bundle for requirements work — User Needs (UN), Product
Requirements (PR), Software Requirements (SR), Test Cases (TC), and end-to-end
traceability. Part of the **Agents On Rails** family.

Works with **GitHub Copilot CLI** and **Claude Code** out of the box (Agent
Skills open standard). Other hosts may need adaptation — see `HANDOVER.md`.

> **For regulated-industry consumers (medical devices, in-vitro diagnostics,
> safety-critical software):** this bundle is engineering tooling, not a
> medical device. Read [Intended use and disclaimer](#intended-use-and-disclaimer)
> below before using outputs in a regulated SDLC, and read
> [Verifying this release](#verifying-this-release) before trusting any
> tarball you did not build yourself.

## What's in the box

| Skill | Purpose |
|---|---|
| `aor-req` | Author UN/PR/SR with EARS-12 patterns and INCOSE quality rules |
| `aor-test-trace` | Author Test Cases (Given/When/Then or procedural) and the trace matrix |
| `aor-review` | Multi-SME review router: `general` \| `all` \| `<role>` \| `<role,role>` |
| `aor-review-adhoc` | Spawn an SME on demand for one-off perspectives |

**EARS-12** means the 12 patterns of the *Easy Approach to Requirements
Syntax* (5 base + 6 extension + NFR decomposition). Full reference:
[`.claude/skills/aor-req/ears-guide.md`](./.claude/skills/aor-req/ears-guide.md).

The review router fans out to specialist agents in `.claude/agents/aor-sme-*.md`:

- `requirements` — EARS-12, INCOSE C1-C9, ISTQB T1-T10, traceability
- `test` — Test case quality, coverage gaps
- `architecture` — Architectural concerns in requirements
- `code` — Code review (when requirements drive code changes)
- `devops` — CI/CD, deployment-relevant requirements
- `documentation` — Doc accuracy and completeness
- `performance` — Performance NFRs and bottlenecks
- `security` — Security requirements, threat coverage
- `ux` — User experience, accessibility, error messaging
- `compliance` — IEC 62304, GDPR, licensing, data privacy

The compliance reviewer is included in v1 of the bundle (delivered via the
framework-canonical export per ADR-009 D6). Earlier prototypes shipped 9
SMEs and listed compliance as "out-of-band"; that gap is closed.

<!-- aor:skills-vs-agents -->

### Skills vs SME agents — what gets invoked when

The bundle ships **two distinct artifact types** at different paths:

- **Skills** at `.claude/skills/aor-*/SKILL.md` — invoked **directly** as
  slash commands (`/aor-req`, `/aor-test-trace`, `/aor-review`,
  `/aor-review-adhoc`). The host loads the SKILL.md frontmatter and the
  user can type the slash command at any prompt.
- **SME agents** at `.claude/agents/aor-sme-*.md` — **never** invoked as
  skills. They are sub-agent system prompts spawned **only** by the
  review router (`/aor-review`) or the ad-hoc spawner
  (`/aor-review-adhoc`). Typing `/aor-sme-requirements` will fail with
  "skill not found" — that is intentional.

If a user wants a one-off SME perspective without committing it to the
permanent panel, use `/aor-review-adhoc "<persona description>"` — the
spawner returns the SME's findings and offers to persist with `--save`.

<!-- aor:operating-discipline -->

## Operating discipline (read before first use)

Skills declare their operating discipline at the top of each SKILL.md and
state it at startup so the user can confirm the mode. The bundle's shared
contract:

- **Mode declaration.** Each session is *exploratory* (single requirement,
  conversational, no files written) or *document* (full specification,
  files written). The skill asks at startup; default is exploratory.
- **Q&A discipline.** Skills ask narrow clarifying questions one at a
  time during exploratory work; in document mode they batch closely
  related questions to avoid serial-prompting fatigue. Multi-choice menus
  are presented via the host's structured-question facility where
  available (`AskUserQuestion` on Claude, `ask_user` on Copilot CLI).
- **Failure disclosure.** If a tool call fails, the skill halts and
  reports rather than silently retrying. A single retry is permitted only
  when the failure is transient and the skill can describe what was
  retried; persistent failures are surfaced to the user.
- **Skills vs agents distinction.** See the section above — never invoke
  an SME agent as a skill.

<!-- aor:cross-host-mapping -->

## Cross-host tool-name mapping

Skills declare tools in their frontmatter using **Claude Code** tool names.
Other hosts use different vocabularies but support the same operations.
This mapping is sourced from the 2026-05-05 GitHub Copilot CLI 1.0.39
host probe (`tools/copilot-runner/probes/fixtures/copilot-1.0.39-host.yaml`).

| Claude Code | Copilot CLI 1.0.39 |
|---|---|
| `Read` | `view` |
| `Grep` | `grep` |
| `Glob` | `glob` |
| `Edit` | `edit` |
| `Write` | `create` |
| `Bash` | `powershell` |
| `AskUserQuestion` | `ask_user` |
| `WebFetch` | `web_fetch` |
| `WebSearch` | `web_search` |

On Copilot CLI 1.0.39 the frontmatter `tools:` field is **decorative** —
accepted syntactically but not enforced at runtime. The same SKILL.md
ships to both hosts. Re-probe before any release tag where the host
version differs from the captured fixture; the mapping silently drifts
otherwise.

<!-- aor:disclaimer -->

## Intended use and disclaimer

This bundle is **engineering tooling** for authoring requirements and test
artifacts (UN, PR, SR, TC) and for running specialist-perspective reviews.

- It is **not** a medical device and not safety-classified under any
  regulatory framework. It does not perform clinical decision support.
- It is **not** a substitute for the consumer's quality management system
  (QMS), software-development lifecycle (SDLC), or regulatory affairs
  function. Outputs are drafts that **must** be reviewed and approved by
  qualified personnel under the consumer's QMS before being treated as
  controlled documents.
- The compliance SME's output is an engineering review input — gap list,
  traceability completeness, testable-clause inventory — **not** a
  declaration of conformity. Conformity assessment under the applicable
  regulation (e.g., MDR Article 19, IEC 62304 §4.3) remains the
  consumer's responsibility, performed by qualified personnel under
  their QMS.
- The AOR bundle is **engineering tooling / software-of-tools** under
  IEC 62304 §5.1.10. It is **not** a medical-device-software item and
  has no IEC 62304 §4.3 safety class (Class A/B/C) — those classes
  apply to device software, not to development tools. Tool-validation
  activities scaled to risk are the consumer's responsibility under
  their tool-validation procedure (typically derived from ISO 13485
  §7.6 and, for the US market, 21 CFR §820.70(i)).
- Each release tarball ships with validation-evidence inputs the
  consumer can fold into their own tool-validation record: SME-panel
  reports captured in the source repository, structural smoke-test
  fixtures (`bundles/aor/tools/smoketest/fixtures/`), reproducibility
  test results (byte-identical tarball from a given parent SHA), and
  the detached signed attestation file that pins the configuration
  identity of each release.

By using this bundle, the consumer accepts that the bundle authors make
no warranty of fitness for any regulated purpose; see `LICENSE` for the
full warranty disclaimer.

<!-- aor:verifying-release -->

## Verifying this release

> Publisher-side commitments — what we promise about how we sign — are
> documented in [`CI-SIGNING-CONTRACT.md`](./CI-SIGNING-CONTRACT.md). For
> evidence-chaining the publisher's signing process during a QMS audit,
> read that document alongside this section.

Each release tarball is shipped with a detached attestation file:
`aor-bundle-vX.Y.Z.tar.gz.attestation.json`. The attestation embeds the
exporter's ED25519 public key alongside a signature over the canonical
form of the attestation payload (excluding the `signature` and
`exporterPublicKey` fields themselves).

**The attestation alone does not prove authenticity** — anyone can
re-tarball and re-sign with their own key. Establish trust by verifying
the embedded `exporterPublicKey` against the publisher's authoritative
key obtained out-of-band:

1. Obtain the publisher's canonical ED25519 public key fingerprint from
   a trusted channel (project README on the source repository, a signed
   git tag, the publisher's website over HTTPS, a key-signing event,
   etc.). The bundle does NOT bootstrap trust on its own.
2. Compute SHA-256 of the `exporterPublicKey` PEM bytes from the
   attestation file. Compare against the published fingerprint.
3. If the fingerprints match, recompute the canonical message
   (attestation JSON with `signature` and `exporterPublicKey` keys
   deleted, all mapping keys recursively lex-sorted, UTF-8 encoded —
   the algorithm is named in the attestation's `canonicalAlgorithm`
   field, currently `"lex-sort-utf8-1"`) and verify the ED25519
   signature.
4. Recompute SHA-256 of the tarball and compare to the attestation's
   `tarballSha256`.
5. If you have the framework source, you can also reproduce the
   tarball: clone the framework at the attestation's `parentSha`,
   pin Node to the version in `.nvmrc`, and run
   `npm run bundle:release -- --bundle-version=<X.Y.Z>
   --parent-sha=<the_pinned_sha> --no-log` — the resulting tarball
   should match the attestation's `tarballSha256` byte-for-byte (per
   ADR-011 D5 reproducibility).

**Unsigned ("DEV BUILD") tarballs** are dev artifacts. The attestation
will have `signature: null` and `exporterPublicKey: null`. Do NOT
distribute these as releases or use them in regulated environments.
Production releases set `AOR_EXPORTER_KEY_PATH` in the release lane;
the resulting attestations carry a real signature.

A reference verifier script is intentionally NOT shipped in this
bundle (it would be self-attesting and therefore meaningless from a
trust standpoint). Implement the four steps above in your verification
tooling of choice, with the publisher's public key managed under your
own key-distribution policy.

The publisher-side counterpart to this section is
[`CI-SIGNING-CONTRACT.md`](./CI-SIGNING-CONTRACT.md), shipped in the
bundle alongside this README. It documents how the canonical release
lane signs (key custody, rotation cadence, GitHub Actions workflow
contract). Read it if you need to evidence-chain the publisher's
signing process for a regulated SDLC.

## Install

Drop the bundle into the host's recognized skills location:

- **Copilot CLI**: `.github/skills/`, `.claude/skills/`, or `~/.copilot/skills/`
- **Claude Code**: `.claude/skills/` and `.claude/agents/` (already matches the bundle layout)

For a project-local install, copy the contents of `aor-bundle/.claude/` into your repo's `.claude/` directory:

**Unix / macOS / Linux (bash, zsh):**

```bash
cp -r aor-bundle/.claude/skills/aor-* your-repo/.claude/skills/
cp aor-bundle/.claude/agents/aor-sme-*.md your-repo/.claude/agents/
```

**Windows (PowerShell):**

```powershell
Copy-Item -Recurse aor-bundle\.claude\skills\aor-* your-repo\.claude\skills\
Copy-Item aor-bundle\.claude\agents\aor-sme-*.md your-repo\.claude\agents\
```

## Usage (typical session)

1. Paste Excel rows into chat: *"translate these requirements into our markdown format"*
2. `/aor-req` — drafts UN/PR/SR following EARS-12 and INCOSE rules
3. `/aor-test-trace` — drafts TCs and the traceability matrix
4. `/aor-review all` — runs the full SME panel, returns aggregated findings
5. `/aor-review-adhoc "regulatory affairs SME for EU MDR"` — on-demand perspective
6. Iterate; export back to Excel by asking the host LLM to translate

Internal format is Markdown. Translation to/from Excel is the host LLM's job —
the skills don't ship Excel parsing.

## Cross-reference syntax

```
relation_type::SOURCE->TARGET [ID-list]
```

Five relation types:

- `traces_to` — forward allocation (UN→PR, PR→SR, SR→TC, UN→Persona)
- `derived_from` — backward satisfaction (PR→UN, SR→PR, TC→SR)
- `validates` — verification link (TC→SR)
- `mitigates` — risk control link (SR→RISK)
- `conflicts_with` — contradiction marker

ID format (regex): `^(UN|PR|SR|TC)-\d{3}(\.[A-Z])?$` — base IDs
(`UN-001`) or sub-IDs for tabular EARS-P patterns (`PR-010.A`,
`PR-010.B`).

**Clause references** (used inside cross-references and trace links to
identify specific behavioural clauses within a single SR): `<ID>.c<N>` —
e.g., `SR-001.c1`, `SR-001.c2`. Clause references are *citations*, not
new IDs; the underlying ID still matches the base regex above.

## Architecture & known limitations

Six load-bearing architectural decisions are recorded in `DESIGN.md`:
pure-prompt orchestration, single-source SMEs, markdown-internal format,
compliance-reviewer-on-demand, ad-hoc SME spawning modes, and the open
reflection loop.

`DESIGN.md` also lists v1 trade-offs the team accepted: JSON contract
duplication, ad-hoc spawner inlining, weak router extensibility, and
informal router→SME path-passing. None block v1; all are candidates for v2.

## License

MIT — see `LICENSE`.

## Feedback / cross-platform spec

This bundle is part of an effort to develop a shared specification for
cross-platform reusable Agent Skills components. **First-time use on a new
host should leave a feedback artifact** at `feedback/<date>-<platform>.md`
following the template in `HANDOVER.md`.
