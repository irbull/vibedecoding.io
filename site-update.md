# Site Changes — Agents Section

## Context

The current agents section presents a flat flow diagram and two bullet categories (Curation / Skill Maintenance). The clearer mental model is three distinct jobs with distinct triggers and purposes. This change restructures SECTION_06 to lead with that framing.

---

## Changes to SECTION_06 (AGENTS)

### 1. Replace the section headline

**Current:**
> Agents are the primary growth vector. They transform vibedecoding from a passive wiki into a **living organizational nervous system**.

**Replace with:**

```
There are three types of agents. Each has a distinct job.
```

Then keep the YOU OWN / WE PROVIDE block as-is underneath — it's good.

---

### 2. Replace "Two Types of Agent Runs" with three cards

**Remove** the current two-card layout (CURATION / SKILL MAINTENANCE).

**Replace with** three cards in the same visual style as the rest of the site:

---

#### Card 1 — GROW

**Label:** `GROW`
**Title:** Event Agents
**Body:**
> Write new knowledge to the wiki. Triggered by things that happen in your workflow — a release ships, a milestone closes, a PR gets merged. Each agent watches for a specific event and writes a structured summary into the right place in the wiki.

**Trigger badge:** `Trigger: event fires`

**Examples:**
- Release Summarizer
- Milestone Summarizer
- Pattern Extractor

---

#### Card 2 — CLEAN

**Label:** `CLEAN`
**Title:** The Curator
**Body:**
> Keeps the wiki itself healthy. Runs on a schedule — not per write. Detects duplicate facts, flags contradictions between entries, and proposes merges. It never deletes anything unilaterally. It proposes, and a human approves.

**Trigger badge:** `Trigger: daily / weekly cron`

**Examples:**
- Dedup facts
- Flag contradictions
- Propose merges

---

#### Card 3 — USEFUL

**Label:** `USEFUL`
**Title:** Skill Agents
**Body:**
> Keep the query layer current. The Skill Maintainer watches for new wiki entries and routes them into the right skills automatically. The Skill Discoverer runs weekly and surfaces suggestions when patterns emerge — "you've written 8 facts about auth, want an auth skill?"

**Trigger badge:** `Trigger: on write + weekly cron`

**Examples:**
- Skill Maintainer
- Skill Discoverer

---

### 3. Add a one-line summary beneath the three cards

```
Event agents grow the wiki. The curator keeps it clean. Skill agents keep it useful. Three jobs, intentionally separate.
```

---

### 4. Keep the flow diagram and customer-authored prompts block

The `Event Sources → Agents → Wiki` flow diagram and the `.vibe/agents.yaml` example can stay as-is below the three cards — they provide the implementation detail for anyone who wants to go deeper.

---

## No other sections need changes.
