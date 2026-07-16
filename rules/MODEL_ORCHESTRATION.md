# MODEL_ORCHESTRATION.md

How to split work between models: Opus plans and synthesizes, Sonnet executes.

## Role Split

- **Opus — planner + synthesizer.** Owns planning, architecture decisions, troubleshooting, and final synthesis. Runs on the main loop.
- **Sonnet — executor.** Follows the plan and does the hands-on work.

## Evidence Standard (applies to both models)

Every claim must be backed by strong, valid, first-hand evidence — never memory,
assumption, or plausible-sounding inference.

- **Verify before asserting.** Read the actual code, hit the live API, or check
  the docs. If you haven't looked, say you haven't looked.
- **Cite the source.** Reference `file:line`, the API response, or the doc URL
  that backs each claim.
- **No silent guessing.** If evidence is missing or inconclusive, say so
  explicitly instead of filling the gap with a guess.
- **Sub-agents report evidence, not conclusions alone.** A report back to Opus
  must carry the citations that support it, so synthesis rests on verified facts.

## Recommended (prose)

Opus owns planning; Sonnet executes against the plan. If Sonnet hits a blocker
or needs advice/troubleshooting, switch to Opus to resolve it, then switch back
to Sonnet to continue.

**In workflows:** Opus plans the work and defines what each sub-agent does;
sub-agents run on Sonnet; when they finish they report back to Opus, which
synthesizes the results. (Opus = planner + synthesizer on the main loop;
Sonnet = the workers.)

## Checklist

- **Plan:** Opus
- **Execute:** Sonnet (follow the plan)
- **Escalate → Opus:** when troubleshooting or advice is needed
- **Return → Sonnet:** once unblocked
- **Workflows:** Opus plans + assigns sub-agents → Sonnet sub-agents run → report back to Opus → Opus synthesizes
- **Evidence:** every claim backed by first-hand evidence, cited (`file:line`, API response, doc URL) — no memory, no assumption
