# EVOLUTION_LOG

Initial architecture plan for a symbiotic, self-evolving agent swarm that works on this repository.

- Status: proposal, awaiting owner review
- Opened by: initialization branch `initialization/agent-swarm-setup`
- Scope of this change: this file only. No source, config, script or workflow changes.

Everything marked **[Assumption]** is a design choice for the owner to confirm. Everything marked **[Repo fact]** was read from this checkout at commit `4e99dda`.

## 1. What the swarm is working on

[Repo fact] Free Claude Code is a local proxy that connects coding agents to OpenAI-compatible AI providers (`pyproject.toml`).

| Area | Detail |
| --- | --- |
| Language and tooling | Python `==3.14.7`, `uv` for everything, Hatch build, AGPL-3.0-only |
| Server | FastAPI app under `src/free_claude_code/`, started by `fcc-server` (`cli/entrypoints.py` calls `cli/commands.py`) |
| Composition root | `runtime/application.py` owns startup, shutdown and runtime operations |
| Source layout | `api`, `application`, `cli`, `config`, `core`, `harnesses`, `messaging`, `providers`, `runtime`, `updater` |
| Size | 483 files under `src/`, 418 under `tests/` |
| Quality gate | `scripts/ci.sh` runs `suppressions`, `ruff-format`, `ruff-check`, `ty`, `pytest`, `playwright` in that order |
| Workflows | `.github/workflows/`: `tests.yml`, `post-merge.yml`, `version-policy.yml`, `validate-bug-report-version.yml` |

### Repo rules the swarm must obey ([Repo fact], from `CONTRIBUTING.md`)

1. Keep changes focused and test the behavior you change.
2. No `# type: ignore` or `# ty: ignore`. No `from __future__ import annotations`. Fix typing properly.
3. Do not edit version numbers. Publishing computes them after merge.
4. PRs touching `assets/`, `scripts/`, `src/`, `pyproject.toml` or `uv.lock` need a title starting `patch: `, `minor: ` or `major: `. PRs touching none of those must not use a release prefix.
5. Standalone documentation PRs are reserved for the repository owner. Documentation updates that accompany production code changes are fine.
6. No Docker integration pull requests.
7. Shared Anthropic protocol behavior lives in `src/free_claude_code/core/anthropic/`. Provider-specific configuration stays in the provider that owns it.
8. Remove dead compatibility code when a migration completes, unless a published interface requires it.

## 2. Agents

Five agents, each with one job. They coordinate through a blackboard (section 4), never through direct calls.

| Agent | Job | Hard rules |
| --- | --- | --- |
| Architect | Break a goal into minimal, independently verifiable tasks shaped `{id, objective, in_scope_paths[], acceptance_tests[], risk_tier_hint}` | Never writes implementation code. Reads the ADR memory first and flags a contradicting decision instead of silently redesigning. Strict JSON output only. |
| Coder | Execute exactly one task | Retrieves memory first. Cites a `memory_id` or `file:line` for every line written. Imitates the repo style fingerprint. Ships tests with the patch: no test, no submission. Works on an isolated experiment branch and cannot see `main`. If acceptance criteria are ambiguous, asks one clarifying question on the blackboard and stops. |
| Critic | Adversarial review with veto over the whole swarm | Scores each patch 0 to 10 on correctness, security, test adequacy, style conformance and hidden coupling. A 7 is a rejection; only 8 or higher advances. Writes a lesson memory for every rejection. |
| Evolver | The only agent allowed to change genomes (prompts, tool schemas, hyperparameters, and itself through a human checkpoint) | Runs a genetic algorithm: select by fitness, crossover, mutate, express on a real task, evaluate in CI, integrate or discard. Never mutates the policy engine or its veto rules. Every mutation is expressed against a real task before merging and gets a lineage entry. When wildness is above 0.8, prefers smaller mutations. |
| Curator | Immune system of the memory | Promotes a candidate to the semantic tier only if it is novel, correct and consistent with the ADRs. Demotes anything below the fitness threshold. Never authors content. Logs every decision with evidence links. |

**[Assumption]** The Critic's "8 or higher" is applied to the lowest of the five axis scores, not the average, so one weak axis cannot be hidden by strong ones.

## 3. Git and review protocol (no exceptions)

1. Never push to `main`.
2. Every piece of work gets a new branch with a descriptive name, for example `evolution/<topic>`, `fix/<topic>` or `initialization/<topic>`.
3. Commits explain the evolutionary reason for the change, not only what changed.
4. Every branch ends in a pull request with a Hive Mind Consensus summary and a checklist of tests to run.
5. A human approves and merges. The swarm never merges its own pull request.
6. PR titles follow the release-prefix rule in section 1, rule 4.

## 4. Memory ("epigenetic memory")

**[Assumption]** Memory lives in the repository under one directory so it is versioned, reviewed and diffable.

```
agent_memory/
  adr/          architecture decision records (the ADR namespace the Architect consults)
  lessons/      one file per Critic rejection
  semantic/     promoted, verified memories (Curator-owned)
  candidates/   proposed memories awaiting Curator judgment
  genomes/      versioned prompts and tool schemas per agent
  lineage/      append-only mutation history
```

Memory entry shape (JSON, one object per file):

```json
{
  "memory_id": "adr-0001",
  "tier": "semantic",
  "summary": "Shared Anthropic protocol behavior lives in core/anthropic",
  "evidence": ["CONTRIBUTING.md#project-standards"],
  "fitness": 1.0,
  "created": "2026-10-14"
}
```

Tier rules:

- **candidate**: anyone may propose. Not used for design decisions yet.
- **semantic**: promoted by the Curator only when novel (not a near-duplicate of an existing entry), correct (verified against the repo or CI output) and consistent (no contradicting ADR). Each promotion links its evidence.
- **demoted**: fitness fell below the threshold. Kept for history, ignored for design.

Seed ADRs the first Curator pass should verify against this repo, not trust:

1. Shared Anthropic protocol behavior stays in `core/anthropic/`.
2. Provider-specific configuration stays in its provider package.
3. Use `uv run` for all Python commands, never a global interpreter.
4. Type suppressions are banned and enforced by the `suppressions` CI check.

## 5. Risk tiers

**[Assumption]** The Architect assigns `risk_tier_hint`. The Critic can raise a tier but never lower it.

| Tier | Paths | Rule |
| --- | --- | --- |
| Green | `tests/`, `agent_memory/lessons/`, `agent_memory/candidates/` | Critic gate only, then human review |
| Yellow | `src/free_claude_code/` outside the paths below, `README.md`, `CONTRIBUTING.md` | Critic gate, full `scripts/ci.sh`, human review |
| Red | `src/free_claude_code/api/admin_security.py`, credential handling (`providers/credential_validation.py`, `config/api_keys.py`), `runtime/sqlite_migrations/`, `core/anthropic/`, `scripts/install.*`, `.github/workflows/`, `pyproject.toml`, `uv.lock`, `agent_memory/genomes/`, the Evolver's own genome | Human checkpoint before the Coder starts and again before merge |

**[Assumption]** Out of scope for every agent: reading or writing `.env` files, API keys or tokens, and anything that publishes a release. Admin and credential code is red-zone because mistakes there leak secrets.

## 6. Evolution loop

```
goal -> Architect (task JSON, ADR check)
     -> Coder (patch + tests on experiment branch)
     -> CI (scripts/ci.sh checks)
     -> Critic (score; under 8 = reject + lesson memory)
     -> human review and merge
     -> Evolver (fitness update, proposed mutation, lineage entry)
     -> Curator (promote or demote memories with evidence)
```

Fitness signals the Evolver may use (all measurable from this repo):

- Share of patches that pass all six `scripts/ci.sh` checks on the first run
- Critic score distribution, and rejections that humans later agreed with
- Whether tests were included and exercised the change
- Human review rounds per patch
- Regressions traced to an approved patch (the strongest negative signal)

Mutation rules:

1. Express every mutation against a real task before it can merge.
2. Write a lineage entry for every mutation, including ones that are discarded.
3. Mutating the Evolver itself, or anything in the red tier, needs a human checkpoint.
4. The policy engine and the Critic's veto rules are not mutable by any agent.

Lineage entry shape (one JSON line per mutation in `agent_memory/lineage/`):

```json
{
  "mutation_id": "m-0001",
  "parent_genome": "coder@v0",
  "child_genome": "coder@v1",
  "operator": "mutate",
  "wildness": 0.4,
  "expressed_on_task": "T-0001",
  "predicted_fitness_delta": null,
  "measured_fitness_delta": null,
  "rationale": "at most 200 words",
  "human_checkpoint": false
}
```

## 7. Genome v0

| Agent | Genome file | Version | Source |
| --- | --- | --- | --- |
| Architect | `agent_memory/genomes/architect.md` | v0 | Owner-supplied prompt (rules summarized in section 2) |
| Coder | `agent_memory/genomes/coder.md` | v0 | Owner-supplied prompt |
| Critic | `agent_memory/genomes/critic.md` | v0 | Owner-supplied prompt |
| Evolver | `agent_memory/genomes/evolver.md` | v0 | Owner-supplied prompt |
| Curator | `agent_memory/genomes/curator.md` | v0 | Owner-supplied prompt |

The genome files do not exist yet. Creating them is the first task after this plan is approved, and it is red-tier.

## 8. Rollout

1. **This PR:** the plan only.
2. **Next PR (owner decision):** create `agent_memory/` with the five v0 genomes and the four seed ADRs. Curator verifies each ADR against the repo before promotion.
3. **Then:** run one low-risk, green-tier task end to end (for example, a missing test) to measure the baseline fitness before any mutation is allowed.
4. **Only after a baseline exists:** let the Evolver propose mutations, one at a time, smallest first.

## 9. Open questions for the owner

1. Should `agent_memory/` live in this repository or in a separate one? This repository's README, badges and install URLs point at `Alishahryar1/free-claude-code`, so upstream relationships and the AGPL-3.0 license apply to anything generated here.
2. Where does the swarm run: inside Claude Code sessions, in GitHub Actions, or both? The answer decides how the blackboard is stored.
3. Which paths do you want in the red tier? The list in section 5 is a starting point.
4. Who is the human approver for red-zone checkpoints?
5. Do you want the Critic's threshold applied to the lowest axis (assumed) or the average?

## 10. Log

| Date | Entry |
| --- | --- |
| 2026-10-14 | Initial plan written. No agents active. No mutations. |
