# vibedecoding.io — Product Specification
**A Wiki for Your Coding Agents**

---

## The Problem

Coding agents are stateless. Every session starts fresh. When an agent discovers that your team has a shared logging library, fixes the issue in the current PR, and moves on — that knowledge dies. The next agent, next week, makes the same mistake.

The problem isn't that agents are dumb. It's that **institutional knowledge doesn't survive between sessions**.

Existing solutions fall short:
- **CLAUDE.md / agent.md**: Single-project scope. No cross-project knowledge.
- **Confluence / Notion**: Built for humans, not queryable by agents.
- **RAG over the codebase**: Finds what *is* — not what *should be* (conventions, decisions, lessons learned).

---

## The Solution

vibedecoding.io is a **persistent organizational memory layer for coding agents** — a wiki that agents can read from and write to, continuously enriched by both humans and automated agents.

### The One-Liner

> CLAUDE.md for your whole org, with a human-readable audit trail and agent tooling on top.

---

## Core Architecture

### Two-Layer Design

**Layer 1 — The Human Layer**
- Markdown wiki, navigable via a web portal
- Humans browse, edit, and review like any wiki (Notion/Confluence feel, but lightweight)
- Plain markdown is the canonical source of truth

**Layer 2 — The Agent Layer**
- A structured, indexed representation optimized for agent queries
- Rebuilt automatically whenever the wiki changes — on write, on git push
- No manual compile step; index maintenance is invisible to users
- No magic black box — the source is always auditable by humans

This mirrors the **event sourcing pattern**: the wiki is the event log, the fact store is the materialized view.

### Storage: Git-Backed

The wiki is a git repository. This gives us:
- Full provenance for free (every fact has author + timestamp)
- PR-based review flow for agent-proposed changes
- Familiar editing workflow for humans
- Trivial hosting — it's just a repo

**Future consideration**: Dolt (versioned SQL) for structured querying if the schema needs grow beyond what frontmatter can handle.

### Wiki Structure

```
vibedecoding-wiki/
  org/
    conventions/
      logging.md
      error-handling.md
    architecture/
      services.md
    decisions/
    skills/
      code-review.md      # org-wide code review skill
      onboarding.md       # org-wide onboarding skill
  projects/
    voice-tools/
      patterns.md
      decisions.md
      skills/
        code-review.md    # voice tools-specific additions (FHIR compliance, etc.)
    beads/
      skills/
        code-review.md
  users/
    ian/
      skills/
        code-review.md    # personal notes and preferences
  .vibe/
    config.yaml           # org, project, agent config
    agents.yaml           # agent definitions and triggers
```

---

## How It Relates to CLAUDE.md

| | CLAUDE.md / agent.md | vibedecoding |
|---|---|---|
| Scope | Single project | Org, project, and user layers |
| Author | Human, manually | Human + agent-assisted |
| Location | In the repo | External, centralized |
| Lifespan | Lives with the repo | Persistent, independent |
| Example fact | "Run tests with `npm test`" | "Always use lib/logger.ts for structured logging" |

**They complement each other.** vibedecoding can inject a relevant summary into CLAUDE.md at project init — "here's what the org knows that's relevant to this repo."

---

## The CLI

The primary interface for agents and humans alike. Agents use it because their `agent.md` tells them to — exactly how Beads works. The CLI is named `vibe`.

```bash
# Get the code-review skill (infers org + project from cwd)
vibe skill code-review

# Explicit project scope (preferred in agent.md)
vibe skill code-review --project voice-tools

# Fully explicit (org + project + user layers all merged)
vibe skill code-review --org mediform --project voice-tools --user ian

# Ad-hoc query when no skill exists yet
vibe query "logging patterns" --project voice-tools

# Write a discovered convention back to the wiki
vibe write "Use lib/logger.ts for all structured logging" \
  --project voice-tools \
  --tag logging \
  --context "discovered in code review"

# Initialize a new project (writes .vibe/config.yaml)
vibe init

# Create a named skill
vibe skill create code-review \
  --description "Facts relevant to reviewing pull requests"
```

There is no manual `vibe compile` step. Indexing happens automatically whenever the wiki changes.

### Context Inference

`vibe` infers context from the directory tree — walking up to find `.vibe/config.yaml`, exactly like git finds `.git`. Explicit flags override inference and take precedence.

```yaml
# .vibe/config.yaml (written by vibe init)
org: mediform
project: voice-tools
```

### Agent Bootstrap

The `agent.md` entry is the only integration required. Skills are specified explicitly so agent behavior is deterministic regardless of working directory:

```markdown
## Organizational Knowledge

This project uses vibedecoding for institutional knowledge.
- Before a code review, run `vibe skill code-review --project voice-tools`
- Before writing code on a new topic, run `vibe query <topic> --project voice-tools`
- When you discover a reusable pattern, run `vibe write --project voice-tools`
- Run `vibe --help` for full usage
```

The agent doesn't need to "know" about vibedecoding. It reads the instructions and uses the tool like any other CLI subprocess.

---

## Skills

Skills are the primary interface between the wiki and agents. A skill is a **named, layered, living context bundle** — everything an agent needs to know for a specific task, assembled from multiple scopes and maintained automatically as the wiki grows.

```bash
vibe skill code-review    # everything relevant to reviewing code
vibe skill onboarding     # everything a new team member needs
vibe skill architecture   # system design decisions and constraints
```

### Why Skills, Not Raw Queries

A raw query like `vibe query logging` requires the agent to know the right keyword. A skill like `vibe skill code-review` just works — it contains everything the agent should know before reviewing code, including logging conventions, error handling patterns, security rules, and anything else relevant that's been added over time.

The agent doesn't need to know what questions to ask. The skill knows.

When you write `use the logging tool in the commons package`, the skill maintenance agent notices logging is relevant to code reviews and updates the `code-review` skill automatically. The connection between facts and skills is maintained without human intervention.

### Layered Scopes

Skills are composed from three layers, merged in order:

```
org/skills/code-review.md          # "always check for structured logging"
  └── projects/voice-tools/skills/code-review.md   # "voice-tools uses FHIR — check compliance"
        └── users/ian/skills/code-review.md  # "I care especially about error boundaries"
```

When an agent calls `vibe skill code-review --project voice-tools --user ian`, it gets all three layers merged into one coherent output. The agent sees a single skill tailored exactly to its context.

**Scope rules:**
- **Org layer** — maintained by the team and agents. Applies everywhere.
- **Project layer** — project-specific additions and overrides. Written by project agents and team members.
- **User layer** — personal notes and preferences. Low friction, no PR review needed.

**On conflicts between layers:** layers annotate rather than override. If org says "always use structured logging" and the project says "this service uses stdout only for now," both facts surface with their context. The agent reasons about the tension rather than getting a silently wrong answer.

### Declared Skills

Users name the intent. The system populates and maintains the skill as the wiki grows.

```bash
vibe skill create code-review \
  --description "Facts relevant to reviewing pull requests"
```

### Discovered Skills

The system notices patterns across writes and surfaces suggestions unprompted:

- "You've written 8 facts about authentication across 3 projects — want me to create an `auth` skill?"
- "There's enough onboarding-related content to build an `onboarding` skill."

A discovery agent runs periodically, clusters related facts, identifies gaps in existing skills, and surfaces suggestions. The user accepts, rejects, or renames them. Accepted skills become declared skills and are maintained going forward.

This solves the onboarding problem: new users don't need to design a skill taxonomy on day one. They just start writing facts, and within days the system surfaces what it thinks they care about.

---

## Agents

Agents are the primary growth vector and the key differentiator. They transform vibedecoding from a passive wiki into a **living organizational nervous system**.

### Agent Model

- **You own**: the agent code, VM runtime, tool integrations, execution guarantees
- **Customer owns**: the prompts, configuration, trigger rules

Customers write prompts in `.vibe/agents.yaml`. You provide reliable, isolated execution.

```yaml
agents:
  - name: release-summarizer
    trigger: github.release
    prompt: |
      You are summarizing releases for a medical scheduling SaaS.
      Focus on API changes, breaking changes, and workflow impacts.
      Our users are clinic administrators, not developers.

  - name: linear-milestone
    trigger: linear.milestone.completed
    prompt: |
      Summarize this milestone focusing on what we decided NOT to build
      and why. That's what we always forget.
```

### Two Types of Agent Runs

**1. Curation** — keeping the wiki itself clean. Wiki-internal. Proposes changes, human approves.
- Detects duplicate facts
- Flags contradictions between entries
- Proposes merges for related content
- Runs on a daily or weekly schedule — not per write

**2. Skill Maintenance** — keeping skills relevant and current across all layers.
- Watches for new wiki entries and determines which skills each entry belongs in
- Updates org, project, and user skill layers as facts are added or changed
- Discovers new skill candidates from emerging clusters
- Runs after writes and on a periodic schedule

These are separate concerns. Curation keeps the source clean. Skill maintenance keeps the agent interface smart.

### Event Sources → Agents → Wiki

```
GitHub release       ──► Release Summarizer    ──► org/releases/
Linear milestone     ──► Milestone Summarizer  ──► projects/{name}/milestones/
Meeting transcript   ──► Meeting Summarizer    ──► org/decisions/
PR merged            ──► Pattern Extractor     ──► org/conventions/
Daily cron           ──► Curator               ──► (deduplicate, flag contradictions)
On write / cron      ──► Skill Maintainer      ──► org|project|user skills/
Weekly cron          ──► Skill Discoverer      ──► surfaces new skill suggestions
```

### Execution Runtime: Firecracker VMs

Agent execution runs in Firecracker microVMs:
- **Isolation**: each run is a clean VM, no cross-customer contamination
- **Fast boot**: ~125ms, so webhook-triggered agents feel responsive
- **Reproducible**: same VM image every run, no dependency drift
- **Resource bounded**: CPU/memory per agent controlled, predictable cost

The VM receives a scoped API token, runs the agent, writes to the wiki via the `vibe` CLI, and exits. Clean and auditable.

Underlying cost is EC2 (Firecracker is the isolation layer). EC2 cost per run is negligible — LLM token cost dominates.

---

## Unit Economics

### LLM Cost Per Run (GPT-4.1: $2.50/1M in, $15/1M out)

| Agent Type | Input Tokens | Output Tokens | Cost/Run |
|---|---|---|---|
| Lightweight (release summary, skill update) | ~3,700 | ~500 | ~$0.017 |
| Heavy (curator, meeting summary, discovery) | ~20,000 | ~2,000 | ~$0.08 |

EC2/Firecracker cost per run: ~$0.001–0.003 (negligible)

**Fully loaded cost: $0.02–$0.10 per run**

### What Uses Runs

- Daily curator: ~30 runs/month
- Skill maintenance (triggered by writes): ~20–50 runs/month for active team
- Weekly skill discovery: ~4 runs/month
- Webhook agents (releases, PRs, milestones): ~50–100 runs/month for active team
- **Total typical active team: ~100–180 runs/month** — well within 300

### Pricing Model

| Tier | Price | Users | Runs/month | Overage |
|---|---|---|---|---|
| **Free** | $0 | 1 | 50 | — |
| **Team** | $39/mo | Up to 15 | 300 | $0.15/run |
| **Business** | $99/mo | Up to 50 | 1,000 | $0.10/run |
| **Enterprise** | Talk to us | Unlimited | Custom | Negotiated |

**Pricing rationale:**
- Gate by run volume, not by agent type or feature access
- All tiers get the same agents — differentiated by how often they run and how many users
- $99 stays under the "just expense it" threshold for most senior engineers
- Flat team tiers avoid per-seat friction
- Free tier is meaningful from day one — skill maintenance runs start immediately on first write

### Fixed Monthly Infrastructure
- EC2 for webhook receiver + agent scheduler: ~$50–100/month
- Covered by 3–5 paying Team customers

---

## MVP Scope

### Phase 1: CLI + Skills
- `vibe init`, `vibe write`, `vibe skill`, `vibe query`, `vibe skill create`
- Git-backed wiki repo with org/project/user layer structure
- Context inference from `.vibe/config.yaml` + explicit flag override
- Declared skills (org layer to start)
- Daily curation agent (dedup, contradiction detection)
- Skill maintenance agent (keeps skills current on writes)
- agent.md bootstrap snippet with explicit `--project` flags

### Phase 2: Webhooks + Event Agents + Full Layering
- GitHub release webhook
- Release summarizer agent (Firecracker runtime)
- Customer-authored prompts via `.vibe/agents.yaml`
- Project-layer skill support
- Skill discovery agent (weekly, surfaces suggestions)

### Phase 3: Portal + User Layer + More Integrations
- Web UI for browsing and editing the wiki
- User-layer skill support (personal notes, no PR review)
- Linear milestone integration
- Team management + billing
- Skill suggestion UI (accept/reject/rename discovered skills)

### Future
- Meeting transcript ingestion
- Jira integration
- Custom agent marketplace / prompt library
- SSO, audit logs (Enterprise)

---

## The Product Story in One Sentence Per Tier

- **Free** — build the wiki habit with the CLI
- **Team** — agents writing to your wiki automatically
- **Business** — agents maintaining themselves
- **Enterprise** — your legal team calls our legal team

---

## Key Differentiators

1. **Standalone CLI** — works with Claude Code, Cursor, Aider, any agent framework. No API integrations to maintain.
2. **Skills, not queries** — agents call `vibe skill code-review` and get everything they need. The skill gets smarter over time without any agent changes.
3. **Layered context** — org, project, and user layers merge into one tailored skill output. The same skill name returns different context depending on where you are.
4. **Skill discovery** — the system tells you what skills you should have, based on what you've already written.
5. **Git-backed trust model** — agent-proposed changes go through PR review. Full provenance. Human-auditable at all times.
6. **Firecracker isolation** — real VM-level isolation for agent execution. Genuinely safe to give write access to a wiki that matters.
7. **Prompt-owned, infrastructure-provided** — customers control the voice and focus of agents; you guarantee reliable execution.
8. **Cross-project scope** — the thing CLAUDE.md can't do.

---

## Open Questions for Planning

- What's the trust model for agent writes? Auto-commit vs. PR draft vs. confidence threshold?
- How does staleness get flagged? (Last-modified date in frontmatter? Curator agent?)
- How does `vibe query` work under the hood for ad-hoc queries — embedding similarity, keyword, or both?
- How are layer conflicts surfaced in skill output — inline annotations, separate sections, or flagged warnings?
- What's the onboarding flow for a new team? (`vibe init` should output a ready-to-paste agent.md snippet)
- Do we support private wiki repos (self-hosted git) from day one, or Enterprise-only?
- How does skill discovery surface suggestions — in the CLI, portal, or a digest email?
- What's the right cadence for skill maintenance — on every write, batched daily, or both?
