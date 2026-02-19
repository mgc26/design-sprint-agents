# Simulating a Design Sprint with 13 AI Agents

**Technical overview of an experiment in multi-agent orchestration for healthcare product design**

---

## What is this?

We ran a full [Jake Knapp Design Sprint](https://www.thesprintbook.com/). Monday through Friday. 10 phases. 13 AI agents. The challenge: design an AI-powered care navigation tool for newly diagnosed cancer patients. The agents debate, synthesize, vote, sketch, prototype, and test with simulated users. ~27 minutes. $4.37.

This repo documents the technical architecture. The code itself is not open-sourced, but we're sharing the methodology, orchestration patterns, and representative examples so others can understand (and critique) the approach.

## The short answer to "are the agents orchestrated?"

**Both.** The agents are orchestrated through a structured 5-day workflow, but within that structure they alternate between independent work and collaborative synthesis. Exactly like a real Sprint.

Some phases are **"together alone"**: agents work independently and cannot see each other's outputs. This prevents groupthink and forces genuine divergence. Other phases are **collaborative**: agents see everything that came before and build on each other's work. A Facilitator agent synthesizes team outputs at key junctures, and a Decider agent has authority to override consensus.

The information architecture (who can see what, when) is the core design decision. Not just "agents talking to each other."

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        SprintEngine                             │
│                   (5-day orchestrator)                           │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────┐  ┌────┐ │
│  │  Monday   │→│ Tuesday  │→│Wednesday │→│Thursday│→│Fri │ │
│  │ Understand│  │  Sketch  │  │  Decide  │  │  Build │  │Test│ │
│  └──────────┘  └──────────┘  └──────────┘  └────────┘  └────┘ │
│       │              │              │             │          │   │
│  3 phases       2 phases       2 phases     1 phase    2 phases │
│  (goal, HMW,   (demos,        (critique,   (role-     (user    │
│   target)       sketches)      storyboard)  based)     tests)  │
└────────────────────────────────┬────────────────────────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
     ┌────────┴────────┐ ┌──────┴──────┐  ┌────────┴────────┐
     │  8 Team Agents   │ │  Facilitator │  │  5 User Agents  │
     │  (persona-driven │ │  (synthesis   │  │  (Friday only,  │
     │   LLM workers)   │ │   via Opus)   │  │   test users)   │
     │                  │ │              │  │                 │
     │ Sonnet: Decider, │ │ Assembles:   │  │ Each is a newly │
     │  Designer, Eng,  │ │  goals,      │  │ diagnosed cancer│
     │  Facilitator     │ │  storyboard, │  │ patient with    │
     │                  │ │  prototype   │  │ distinct life   │
     │ Haiku: Oncologist│ │              │  │ circumstances   │
     │  Navigator,      │ └──────────────┘  └─────────────────┘
     │  Advocate,       │
     │  Troublemaker    │
     └──────────────────┘
```

### Agent-to-agent communication is mediated, not direct

Agents never talk to each other directly. The SprintEngine controls what each agent can see at each phase:

| Phase | What agents can see | What's hidden |
|-------|-------------------|---------------|
| Sprint Goal | Platform context, challenge definition | Nothing. Everyone sees everything |
| HMW Notes | Goal, customer journey map | **Other agents' HMW notes** |
| Target Vote | All HMW notes revealed | Nothing |
| Lightning Demos | Goal, map, target | Nothing |
| Solution Sketches | Goal, map, target, all demos | **Other agents' sketches** |
| Critique & Vote | All sketches **anonymously** | **Who wrote which sketch** |
| Storyboard | Winning sketches, supervote decision | Nothing |
| Prototype | Full storyboard, all prior context | Nothing |
| User Testing | Full prototype, interview script | Other users' reactions (sequential) |
| Pattern Debrief | All 5 user test results | Nothing |

This information asymmetry is the key design pattern. It mirrors the real Sprint methodology and produces meaningfully different outputs than if all agents could see everything at all times.

---

## The 13 Agents

### 8 Team Members

Each agent has a detailed persona (background, decision framework, personality traits, known biases, relationships with other agents) and a specific set of research tools.

| Agent | Role | Model | Sprint Authority |
|-------|------|-------|-----------------|
| Dr. Elaine Sato | Chief Patient Experience Officer | Sonnet | **The Decider.** 4 HMW dots (vs 2), 3 supervotes, final verdict |
| Jordan Whitaker | Design Sprint Facilitator | Sonnet | Synthesizes team outputs, assembles storyboard & prototype |
| Priya Anand | UX Lead / Product Designer | Sonnet | Solution sketches, prototype screens |
| Marcus Webb | Clinical AI Architect | Sonnet | Technical feasibility, cost modeling |
| Dr. Ravi Chandrasekaran | Medical Oncologist | Haiku | Clinical domain grounding, plain-language medical copy |
| Carmen Delgado | Oncology Nurse Navigator | Haiku | Workflow reality, real timelines and resources |
| James Osei | Cancer Survivor & Advocate | Haiku | Patient experience, emotional truth, interview script |
| Dr. Zara Malik | Health AI Ethics Researcher | Haiku | Equity critique, regulatory risk, safety guardrails |

### 5 User Testers (Friday only)

Each user is a newly diagnosed cancer patient with distinct barriers. They react to the prototype authentically. They are not polite.

| User | Profile | What They Reveal |
|------|---------|-----------------|
| Maria Santos | 52, housekeeper, Spanish-dominant, low health literacy, prepaid Android | Whether the tool works on a small screen with limited data and shaky medical English |
| Robert Fitzgerald | 68, retired engineer, widowed, iPad native but AI-skeptical | Whether elderly tech-literate users trust AI and whether family features work |
| Destiny Johnson | 34, single mother of 2, works retail, financial stress | Whether the tool accounts for childcare, work schedules, financial assistance |
| Wei Zhang | 55, limited English proficiency (Mandarin), restaurant worker | Whether the tool works for non-English, non-Spanish LEP patients |
| Kevin O'Brien | 42, respiratory therapist, health-literate, power user | Whether health-literate patients find it too shallow or patronizing |

---

## Model Mixing Strategy

We don't use the same model for every agent. The model assignment reflects the cognitive demand of each role:

- **Sonnet** (Claude Sonnet 4.5): Complex reasoning. Synthesis, design, financial modeling, technical architecture.
- **Haiku** (Claude Haiku 4.5): Grounded, practical, direct perspectives. Clinical workflow, patient experience, ethical critique.
- **Opus** (Claude Opus 4.5): Facilitator synthesis calls only (goal consolidation, storyboard assembly, prototype assembly). Used where output quality is critical.

Haiku is ~4x cheaper per token than Sonnet, so running 9 of 13 agents on Haiku meaningfully reduces cost. But the Haiku agents aren't just cheaper. They're different. The Nurse Navigator's blunt, practical perspective is actually better served by a model that doesn't over-elaborate.

---

## Prompt Caching Architecture

13 agents making 5-6 calls each. ~82 total API calls. Prompt caching is critical for cost:

```
System prompt structure (ordered for cache efficiency):
┌───────────────────────────────────────────┐
│ 1. Platform context (identical for all    │ ← Cached once, read by all agents
│    agents in all phases)                  │
├───────────────────────────────────────────┤
│ 2. Shared context (identical for all      │ ← Cached once per phase,
│    agents within a phase, includes        │    read by remaining agents
│    prior phase outputs)                   │
├───────────────────────────────────────────┤
│ 3. Persona (unique per agent, ~500 tokens)│ ← Not cached (different each time)
└───────────────────────────────────────────┘
```

We also use **staggered execution**: within each phase, we fire one Sonnet agent and one Haiku agent first (as "cache primers"), wait for them to complete, then run all remaining agents in parallel. The remaining agents hit warm caches on both the platform context and the shared context blocks.

**Result:** 607K cache-read tokens vs 82K fresh input tokens. ~7:1 cache hit ratio.

---

## Research Tools

Agents don't just generate opinions from training data. Each agent has access to domain-specific research tools that return structured evidence. Tools are assigned by expertise:

| Tool | What It Returns | Assigned To |
|------|----------------|-------------|
| Patient Journey Research | Barriers, emotional trajectory, digital adoption rates | Designer, Oncologist, Navigator, Advocate |
| Competitive Landscape | Existing products (Jasper Health, Belong.life, etc.) | Designer, Engineer |
| AI Capability Assessment | What LLMs can/can't do reliably for clinical tasks | Engineer, Troublemaker |
| Health Literacy Data | Readability guidelines, WCAG requirements | Designer |
| Clinical Workflow | Current navigator workflow, bottlenecks, timelines | Decider, Oncologist, Navigator |
| Regulatory Landscape | FDA CDS guidance, HIPAA, state privacy laws | Troublemaker |
| Health Equity Evidence | Digital divide data, mitigation strategies | Advocate, Troublemaker |
| Implementation Cost | Build vs. buy models, per-patient cost at scale | Decider, Engineer |

Tools are called during evidence-gathering phases (Sprint Goal, HMW Notes, Lightning Demos). The tool outputs become part of the agent's reasoning context. They cite specific data points in their proposals.

---

## Structured Output & Parsing

Every phase prompt specifies an exact response format. This enables reliable artifact assembly and JSON export:

```
RESPOND WITH THIS EXACT STRUCTURE:
LONG_TERM_GOAL: [1-2 sentence aspirational goal]

SPRINT_QUESTIONS:
1. [Question framed as "Will..." — biggest fear]
2. [Second biggest fear]
...
```

This isn't just prompt engineering hygiene. It's necessary for the Facilitator to synthesize 8 independent proposals into a single consolidated goal, or for the Friday debrief to map user reactions back to Monday's sprint questions.

---

## Temperature as a Design Tool

Different phases use different temperatures to match the cognitive mode:

| Cognitive Mode | Temperature | Phases |
|---------------|-------------|--------|
| Divergent / Creative | 0.8-0.9 | HMW Notes, Solution Sketches, User Testing |
| Balanced | 0.5-0.6 | Storyboard, Prototype, Interview Script |
| Analytical / Decisive | 0.3-0.4 | Target Vote, Supervote, Pattern Debrief, Verdict |

Solution Sketches (the most creative phase) run at 0.9. The Decider's Verdict runs at 0.3. This matters. Running creative phases at low temperature produces bland, convergent outputs. Running voting phases at high temperature produces inconsistent reasoning.

---

## What Happened: Results from One Run

**Challenge:** Design AI-powered care navigation for newly diagnosed cancer patients at an NCI-designated cancer center. 34-day diagnosis-to-treatment gap. $1.5M budget. 14 overwhelmed nurse navigators.

**The Decider's Verdict: Flawed Success**

The prototype validated three core hypotheses:
1. **AI transparency builds trust** (4/5 users positive). "I like that she says right away that it's not a person. Because if you don't tell me that, and I find out later, I'm going to feel tricked."
2. **Proactive outreach within 2 hours closes the psychological void** (5/5 users validated). Every user contrasted the prototype's immediate callback with their real experience of waiting 3+ days.
3. **Concrete timeline structure reduces cognitive load in acute crisis** (4/5 users positive). "NOW we're talking. This is what I needed. Not 'how are you feeling,' just: here's what happens Saturday, here's what happens Monday."

But it also revealed critical failures:
1. **Language accessibility is a blocker.** The prototype was built in Spanish to demonstrate equity, but accidentally excluded Mandarin speakers. Demonstrated the exact two-tiered system the team was trying to avoid.
2. **Only solves the first 72 hours.** The 34-day gap has three distinct phases (acute crisis, workup, treatment decision). The prototype nailed phase 1 but was incomplete for phases 2 and 3.
3. **Generic emotional validation alienates some patients.** "If this thing called me and started talking about my feelings, I'd hang up."

### Cost

| Metric | Value |
|--------|-------|
| Total cost | $4.37 |
| API calls | 82 |
| Input tokens | 82,403 |
| Output tokens | 116,907 |
| Cache read tokens | 607,687 |
| Wall clock time | 26 min 53 sec |

The most expensive agent was the Facilitator ($2.62) because its synthesis calls use Opus. The five user testers cost $0.05 combined.

---

## Key Design Patterns

### 1. "Together Alone": Controlled Information Asymmetry

The most important pattern. In real Sprints, participants sketch alone so they aren't anchored by the loudest voice. We enforce this computationally: during HMW Notes and Solution Sketches, agents see shared context (goal, map, target) but **never** see other agents' outputs for that phase.

This produces genuine divergence. Without it, agents converge on the first idea generated. A well-documented failure mode in multi-agent systems.

### 2. Decider Authority with Transparent Override

The Decider has more voting power (4 dots vs 2, 3 supervotes vs straw poll) and can override team consensus. But must explain reasoning. This mirrors real organizational decision-making: the person accountable for the budget has final say, but the team's input is visible and shapes the decision.

### 3. Anonymous Critique

Solution Sketches are presented without attribution during Wednesday's critique. Agents vote on ideas, not people. This prevents status effects (the Decider's sketch winning because they're the boss, not because it's best).

### 4. Sequential User Testing

Friday's user tests run sequentially, not in parallel. Each user reacts to the prototype without seeing other users' reactions. Just like real usability testing. The pattern debrief afterward is where the team synthesizes across all five tests.

### 5. Role-Based Prototype Building

On Thursday, agents aren't all doing the same thing. They're assigned roles:
- **Makers** (Designer + Engineer + Troublemaker): Build screens and interactions
- **Writer** (Oncologist): Clinical accuracy at a 5th-grade reading level
- **Asset Collector** (Navigator): Realistic timelines, appointments, resources
- **Interviewer** (Advocate): Writes Friday's interview script
- **Stitcher** (Facilitator): Assembles everything into a coherent prototype

---

## What This Is and Isn't

**This is:**
- An experiment in structured multi-agent orchestration for design thinking
- A demonstration that information architecture (who sees what, when) matters more than agent count
- A tool for rapid, cheap exploration of complex design spaces (~$4, ~27 min for a full Sprint)
- A way to stress-test ideas against diverse user archetypes before spending months on real user research

**This is not:**
- A replacement for real design sprints with real humans
- A claim that AI agents "think" or "feel" or "have opinions"
- A general-purpose multi-agent framework (purpose-built for Sprint methodology)
- Production software (research experiment)

---

## Example Artifacts

This repo includes:
- [`examples/persona-team.yaml`](examples/persona-team.yaml): Example team persona (the Decider)
- [`examples/persona-user.yaml`](examples/persona-user.yaml): Example user test persona (Maria Santos)
- [`examples/sprint-excerpt.md`](examples/sprint-excerpt.md): Excerpts from a full sprint report (Monday goal, Friday verdict)

---

## Stack

- **Language:** Python 3.11+, fully async (`asyncio`)
- **LLM:** Anthropic Claude API (Sonnet 4.5, Haiku 4.5, Opus 4.5)
- **Personas:** YAML configuration files (no persona logic in code)
- **Output:** Markdown report + structured JSON export
- **Concurrency:** `asyncio.gather()` within phases, sequential across phases

---
