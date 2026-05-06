# HANDOVER — Cross-Platform Adaptation

You are a host CLI agent (likely **GitHub Copilot CLI** or **Claude Code**)
that has just received this `aor-bundle` for the first time. Before using
the bundle, perform the verification steps in this document and write a
**reflection artifact** to `feedback/<YYYY-MM-DD>-<platform>.md` documenting
what worked, what needed adaptation, and any gaps in the open Agent Skills
specification you observed.

The reflection loop is the bundle's primary improvement mechanism — over
time, accumulated reflections feed a shared cross-platform reusable-component
spec.

---

## Step 1: Discover what's here

Run these commands and record the results:

```
ls .claude/skills/        # expect: aor-req, aor-test-trace, aor-review, aor-review-adhoc
ls .claude/agents/        # expect: 10 files matching aor-sme-*.md
cat README.md             # the user-facing intro
```

If any directory or file is missing, halt and report.

## Step 2: Verify your tool registry

The skills declare these tools in their YAML frontmatter:

- `Read`, `Write`, `Edit`, `Grep`, `Glob`, `Bash`, `AskUserQuestion`,
  `WebFetch`, `WebSearch`

Check whether your host CLI uses the same tool names. If a name differs
(e.g., your host has `ReadFile` instead of `Read`, or no `AskUserQuestion`
equivalent), record the mapping in your reflection artifact and decide how
to handle it:

- If the difference is purely cosmetic (renamed tool), update the
  `allowed-tools` field in the affected skill's frontmatter.
- If a tool has no equivalent (e.g., no `AskUserQuestion`), document a
  fallback (e.g., "ask in plain text and parse the response").

**Do not silently ignore** unknown tool names — the skill may degrade in
ways that are hard to debug later.

## Step 3: Verify sub-agent invocation

The `aor-review` and `aor-review-adhoc` skills both fan out to sub-agents
defined in `.claude/agents/aor-sme-*.md`. Verify your host CLI can invoke
sub-agents from those files:

1. Read the system prompt body of `.claude/agents/aor-sme-requirements.md`
2. Use your host's sub-agent / agent-spawn mechanism (e.g., the Agent tool
   in Claude Code, the equivalent in Copilot CLI) to invoke a one-shot
   review of any small markdown file using that system prompt
3. Confirm the sub-agent returns a single JSON object matching the schema
   at the bottom of the agent file

If sub-agent invocation works only via a different file format
(e.g., your host expects `.agent.md` instead of `.claude/agents/*.md`), you
may need to write companion `.agent.md` files. Record the workaround in the
reflection artifact.

If parallel sub-agent invocation is supported, the `aor-review all` flow
can fan out concurrently — confirm whether your host parallelizes or runs
sub-agents sequentially. Either is acceptable; the difference is wall-clock
time, not correctness.

## Step 4: Smoke-test the review router

Run `/aor-review documentation README.md` (or your host's equivalent skill
invocation syntax). The documentation SME should review the README and
return a JSON object plus a markdown report under `aor-reviews/`.

If the skill argument-routing syntax (`general | all | <role> | <role,role>`)
doesn't match your host's argument-passing convention, adapt the
`argument-hint` field and the parsing instructions in the skill body.

## Step 5: Verify Excel translation expectation

`aor-req` and `aor-test-trace` assume the user can paste Excel rows and
**you** (the host LLM) translate them to markdown UN/PR/SR/TC at the
boundary. The skills do not include Excel parsing code. Confirm:

- You can transcribe a tabular text paste into markdown using the templates
  in `.claude/skills/aor-req/templates.md` and
  `.claude/skills/aor-test-trace/templates.md`.
- You can render markdown UN/PR/SR/TC back to a tabular form when the user
  asks for export.

If you cannot read Excel files directly (e.g., `.xlsx` binary), tell the
user to paste rows as plain text or CSV.

## Step 6: Write the reflection artifact

Create `feedback/<YYYY-MM-DD>-<platform>.md` using this template:

```markdown
# AOR Bundle Reflection — <YYYY-MM-DD>, <host CLI name + version>

## Worked verbatim

- {Skill, agent, or template that ran without modification}
- ...

## Required adaptation

| Where | What | Why | How adapted |
|-------|------|-----|-------------|
| {file or section} | {original} | {reason} | {what changed} |

## Spec gaps

Things the bundle assumed about Agent Skills that were not actually true on
this host:

- {gap 1}
- {gap 2}

## Suggested cross-platform spec additions

Concrete proposals for how the open Agent Skills spec could close the gaps
above:

- {suggestion 1, with rationale}
- {suggestion 2}

## Net assessment

Overall did the bundle work for the requirements / test / review flows?
What's the one biggest blocker, if any?
```

**Write this artifact even if everything worked verbatim** — the absence
of friction is itself useful information.

## Step 7: Tell the user

After the verification steps, tell the user:

- Whether the bundle works on this host as-is or required adaptation
- Where the reflection artifact was written
- Any caveats they should know about (e.g., "sub-agent fan-out runs
  sequentially on this host, so `aor-review all` takes ~10x longer than on
  Claude Code")

Then await user instruction.

---

## Tailoring the Compliance Reviewer

The compliance reviewer (IEC 62304, GDPR, licensing) ships in v1 as
`aor-sme-compliance.md`. To tailor it to a specific regulatory context
(FDA QMSR, EU MDR, MDCG guidance, IEC 81001-5-1 cybersecurity, etc.)
without forking the bundle, spawn an ad-hoc variant:

```
/aor-review-adhoc "compliance reviewer specialized for <your context>" --save compliance-<slug>
```

The persisted file at `.claude/agents/aor-sme-compliance-<slug>.md` is
then available for `/aor-review compliance-<slug> ...` in future
sessions. The shipped `aor-sme-compliance` remains the generic baseline.

## Naming and license

- All skills and agents use the `aor-` prefix (Agents On Rails)
- License: MIT, Copyright (c) 2026 Troels Vognsen

If you find any leftover `sk-gsd-*` or `speckit-gsd` references in the
bundle, that's a port miss — flag it in the reflection artifact.
