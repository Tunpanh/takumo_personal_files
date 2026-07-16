# MODEL_ORCHESTRATION.md

How to split work between models: Opus plans and synthesizes, Sonnet executes.

## Role Split

- **Opus — planner + synthesizer.** Owns planning, architecture decisions, troubleshooting, and final synthesis. Runs on the main loop.
- **Sonnet — executor.** Follows the plan and does the hands-on work.

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
