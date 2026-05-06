# Architectural Decisions

Six load-bearing decisions made in v1 of the bundle. Each is recorded with
context, decision, and consequence so future porters and maintainers can
distinguish deliberate constraints from accidents.

---

## 1. Pure-prompt orchestration (no compiled component)

**Context.** The source framework had a Node.js orchestrator, HMAC audit
logging, TOCTOU defenses, and pipeline state. None of these are available
in a portable Agent Skills bundle that must work in two host CLIs.

**Decision.** The bundle ships zero executable code. The `aor-review`
router skill describes fan-out in plain English and relies on the host
CLI's sub-agent mechanism. Aggregation logic lives in the prompt.

**Consequence.** Trivially portable; loses programmatic guarantees (e.g.,
parallel execution, audit-grade timing, JSON-schema validation by code).
Not appropriate for regulated-pipeline use cases.

## 2. Single-source-of-truth SME files

**Context.** Reviewer logic could live in the router, the agents, or
both. Splitting it creates drift; merging it creates a monolith.

**Decision.** The 10 SMEs in `.claude/agents/aor-sme-*.md` are the **only**
place reviewer logic lives. The router auto-discovers them via Glob
(`.claude/agents/aor-sme-*.md`). The router contains no SME-specific
knowledge.

**Consequence.** Adding a new SME is one file drop, no router edit.
Removing or renaming an SME just removes the file.

## 3. Markdown internally; host LLM does Excel translation

**Context.** The user receives requirements in Excel and must return them
in Excel. The bundle could ship Excel parsing or push it to the host.

**Decision.** All bundle internal artifacts are markdown. The host LLM is
responsible for Excel ↔ markdown translation at the user-facing boundary.
The bundle stays format-agnostic.

**Consequence.** Bundle has no `xlsx` dependency. Translation quality
varies with the host's tabular-handling capability. Excel-specific edge
cases (merged cells, formulas, formatting) are out-of-bundle concerns.

## 4. Compliance reviewer intentionally absent in v1

**Context.** The source framework had an IEC 62304 / GDPR / licensing
compliance reviewer. Including it would couple the bundle to the medical
device domain.

**Decision.** v1 ships 9 domain-neutral SMEs. Compliance is added on
demand via `/aor-review-adhoc --save compliance` with a role description
matching the user's regulatory context.

**Consequence.** Bundle works for unregulated and regulated domains. Each
regulated team gets a compliance reviewer tuned to their actual standard
(FDA QMSR, EU MDR, IEC 62304, IEC 81001-5-1, GDPR, HIPAA, etc.).

## 5. Persisted vs. ephemeral ad-hoc SME spawning

**Context.** A one-off SME role might be reused or might be truly one-off.

**Decision.** `aor-review-adhoc` defaults to ephemeral (no file written).
With `--save <slug>` it persists to `.claude/agents/aor-sme-<slug>.md`,
where the router will then auto-discover it.

**Consequence.** Cheap to try out an SME role; cheap to keep one that
proved useful. The persistence path uses the same file format as the
canonical SMEs, which means the persisted file is indistinguishable from
a hand-written one.

## 6. Reflection loop is open at v1

**Context.** Each new host (Copilot CLI, Claude Code, future hosts) will
have small adaptation needs. We need to capture those without committing
to a closed feedback ingestion mechanism.

**Decision.** `HANDOVER.md` requires the host to write a per-host
reflection artifact at `feedback/<date>-<platform>.md` on first use. The
loop terminates outside the bundle — accumulated artifacts feed a future
shared cross-platform reusable-component spec, not an automated bundle
update mechanism.

**Consequence.** Reflections accumulate as data, not as code paths. v1
does not auto-apply suggestions; v2+ may add a curation/distillation
step.

---

## Known v1 limitations (accepted trade-offs)

These are surfaced by an architecture review and held over for v2:

- **JSON output contract is duplicated across 11 files** (10 SMEs +
  `aor-review-adhoc`). A future `aor-sme-contract.md` referenced from
  each agent's frontmatter would single-source it. Until then, schema
  changes require synchronized edits.
- **`aor-review-adhoc` inlines the SME frontmatter shape**, which means
  if the canonical frontmatter ever gains a field, the spawner emits
  stale files. A future agent-template reference file would invert this
  dependency cleanly.
- **Router extensibility is weak**: a second orchestration mode (e.g.,
  `aor-review-batch` reading a config file) would re-implement the
  router monolithically. Aggregation and report formatting could be
  factored into shared reference files to make routers substitutable.
- **Router → SME path-passing protocol is informal**: the router tells
  each SME to review "the following targets" in prose. There is no
  explicit contract for working directory, allowed file traversal beyond
  listed targets (e.g., for cross-reference checks), or argument
  serialization. Host-LLM inference closes the gap today.

These are real architectural smells; they are not blocking for v1's goal
of getting a working bundle into the user's Copilot CLI.
