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
- **Name the specimen.** A result states what it was measured on — which file,
  which record, which device. Citing the source shows *how* you measured; naming
  the specimen shows *what* you measured, and no citation can catch a wrong
  specimen. "It works" without the artifact is an impression, not a result.
- **A negative result needs a positive control.** Before reporting "X is absent",
  prove the same check finds an X you know is there. Otherwise "not found" and
  "my detector is broken" look identical.
- **A proxy is not the thing.** When work ends at hardware, an external service,
  or someone else's environment, exercise it there once — or say plainly which
  layer you verified and what stays unverified.
- **Agreeing is asserting.** The bar does not drop when you are supporting
  someone else's call. A reason invented to justify a decision already made is a
  guess wearing a citation's clothes.

A recorded conclusion (memory, plan, doc) carries its specimen and the condition
that would refute it — "green because X; if Y ever appears, this is wrong."
Stored without both, it hardens into background truth nobody re-checks.

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
- **Specimen:** every result names what it was measured on; a negative result carries a positive control; hardware/external work is proven there, not on a proxy; agreeing with someone is still asserting
