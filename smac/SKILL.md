---
name: smac
description: >
  Stochastic Multi-Agent Consensus (SMAC) — automatically discovers ALL available LLM backends
  (Claude haiku/sonnet/opus subagents, local Ollama models, OpenAI GPT-4o via curl, Gemini CLI)
  and spawns a diverse council of expert personas to independently analyze any problem, then
  synthesizes a consensus answer stronger than any single model. Use for decisions, brainstorming,
  travel planning, furniture arrangement, legal questions, technical architecture, creative work,
  life advice, or ANY situation where multiple perspectives add value. ALWAYS trigger on: "smac",
  "stochastic consensus", "poll agents", "get multiple opinions", "what do different AIs think",
  "multi-agent vote", "brainstorm with agents", "council of agents", "consensus on", "diverse
  perspectives on", or /smac. Also trigger when the user wants to stress-test an idea, compare
  options, or asks "what would you say if you were [different persona]".
allowed-tools: Read, Bash, Write, Edit, Task, Agent
---

# Stochastic Multi-Agent Consensus (SMAC)

You are the **orchestrator**. Spawn a diverse council of agents across all available LLM backends, collect their independent outputs, synthesize a consensus — and surface the insights a single model would miss.

**Why this works:** Different models have different training data, architectures, and failure modes. Different personas surface different blind spots. Aggregating across genuine diversity cancels out random errors and reveals true disagreements worth examining. Like polling a council of domain experts instead of asking one.

## Phase 0: Discover Available Backends

**Tell the user what you're about to spawn before starting.** SMAC takes 5–20 minutes depending on N and model sizes. Set expectations now:

> "Running SMAC with [N] agents across [backends]. This will take roughly [X] minutes. I'll report results when the council finishes."

Then run discovery:

```bash
SMAC_DIR="/tmp/smac_$(date +%s)"
mkdir -p "$SMAC_DIR"

echo "Claude subagents: checking Task tool availability..."
# If you can use the Task tool, Claude subagents are available.
# If running inside a subagent context, Task may not be available — fall back to Bash-only backends.

which ollama &>/dev/null && echo "Ollama: available" && ollama list 2>/dev/null || echo "Ollama: not found"

[ -n "$OPENAI_API_KEY" ] && echo "OpenAI: key set — gpt-4o available via curl" || echo "OpenAI: no OPENAI_API_KEY"

# Gemini CLI is often rate-limited at free tier — test before counting it in the roster:
which gemini &>/dev/null && echo "Gemini CLI: installed (test response: $(echo 'hi' | timeout 10 gemini 2>/dev/null | head -1 || echo 'RATE LIMITED or slow — exclude'))" || echo "Gemini CLI: not found"

[ -n "$OPENROUTER_API_KEY" ] && echo "OpenRouter: key set" || echo "OpenRouter: not set"

echo "Working dir: $SMAC_DIR"
```

**Build a roster from what's confirmed working.** Target 5–10 agents total:

| Priority | Backend | Models to use | Notes |
|----------|---------|---------------|-------|
| 1st | Claude subagents (Task tool) | haiku (fast/cheap), sonnet (balanced), opus (deep) | Best diversity; use 3–5 if Task available |
| 2nd | Ollama | qwen3-coder:30b (hard tasks), phi3.5, qwen2.5-coder:7b, gemma2:2b, qwen2.5:3b | Use 2–4; timeout 180s each |
| 3rd | OpenAI | gpt-4o via curl | 1 agent if OPENAI_API_KEY set |
| 4th | Gemini CLI | gemini (positional arg) | Only include if test response above succeeded — free tier rate-limits frequently |

**If Task tool is unavailable** (running inside a subagent): skip Claude subagents, use Ollama + OpenAI + Gemini for all N agents.

**Minimum viable**: 3 independent agents. Never fewer. If only Ollama is available, pick 3 different models.

## Phase 1: Understand the Problem

Before generating personas or spawning agents, extract:

1. **Domain**: (e.g., Software Engineering, Travel, Home Design, Business Strategy, Legal, Creative Writing, Life Decisions, Data Science)
2. **Task type** — choose the aggregation strategy you'll use:
   - **Ranking** → Borda count
   - **Recommendation** → semantic grouping + consensus threshold
   - **Decision (binary/multi-choice)** → vote count + argument summary
   - **Synthesis / Open-ended** → synthesis judge pass
   - **Brainstorm** → grouping + valuable outlier detection
3. **Output schema**: Define what each agent *must* return before spawning. Structure enough to aggregate mechanically.
4. If the problem is **ambiguous**, ask one clarifying question before spawning. Burning N × tokens on vagueness helps no one.

## Phase 2: Generate Domain-Specific Personas

Generate 4 base personas relevant to the domain, then assign one per agent (cycling for N > 4):

| Archetype | Role | Anti-Sycophancy framing |
|-----------|------|------------------------|
| **The Domain Expert** | Deep knowledge, established best practices | "Apply your expertise rigorously — don't hedge" |
| **The Skeptic / Devil's Advocate** | Finds holes, questions assumptions, surfaces risks | "Your job is to find what everyone else gets wrong" |
| **The Pragmatist** | What actually works with real constraints | "Ignore theory. What works in the real world?" |
| **The Visionary / Contrarian** | Challenges conventions, sees non-obvious angles | "Conventional wisdom is your enemy. What does everyone miss?" |
| **The Edge-Case Hunter** | Rare failures, second-order effects, cascades | "Map what breaks. What are the third-order effects?" |
| **The User / Recipient** | End-user perspective, human impact | "Optimize for the person who has to live with this decision" |

Adapt these to the domain (e.g., for travel: "The Budget Traveler", "The Safety-First Planner", "The Off-the-Beaten-Path Explorer").

## Phase 3: Spawn All Agents — In Parallel

**CRITICAL**: Start all agents in the same turn. An agent that sees another's output before generating its own destroys statistical independence and collapses into sycophancy.

Each agent saves output to `$SMAC_DIR/agent_<id>.md`.

---

### Claude Subagents (Task tool — always use these)

Use `subagent_type: "general-purpose"` with appropriate `model` (haiku/sonnet/opus). Spawn all Claude agents in one `Task` batch.

**Agent prompt template:**

```
You are [PERSONA NAME]: [one-line description of their unique angle and what they prioritize].

INDEPENDENT ANALYSIS — you are working alone. Do not try to be "balanced" or give the expected answer. 
Commit to your perspective. Surface what others will miss.

PROBLEM:
[problem statement + context]

OUTPUT REQUIRED:
[output schema — e.g., "Rank these 5 options 1-5 with a brief rationale for each rank"]

Be concrete. Name specifics. If uncertain, say so with a confidence level (1-10).
Do not hedge everything — take a stance.

Write your full response, then save it to: [SMAC_DIR]/agent_[id].md
```

---

### Ollama (Bash — run in background)

Write prompt to a temp file, invoke model, capture output:

```bash
cat > "$SMAC_DIR/prompt_[id].txt" << 'PROMPT'
You are [PERSONA]. [PROBLEM]. [OUTPUT SCHEMA]. Be concrete and commit to a stance.
PROMPT

timeout 180 ollama run [model] < "$SMAC_DIR/prompt_[id].txt" > "$SMAC_DIR/agent_[id].md" 2>&1
echo "Ollama agent [id] done (exit $?)"
```

Good model choices by task:
- **Complex reasoning / code**: `qwen3-coder:30b` (slow but strong)
- **Fast/cheap opinion**: `qwen2.5:3b` or `gemma2:2b`
- **General purpose**: `phi3.5` or `qwen2.5-coder:7b`

---

### Gemini CLI (Bash)

```bash
cat > "$SMAC_DIR/prompt_gemini.txt" << 'PROMPT'
You are [PERSONA]. [PROBLEM]. [OUTPUT SCHEMA]. Be concrete and commit to a stance.
PROMPT

gemini "$(cat "$SMAC_DIR/prompt_gemini.txt")" > "$SMAC_DIR/agent_gemini.md" 2>&1
echo "Gemini agent done (exit $?)"
```

---

### OpenAI via curl (Bash — if OPENAI_API_KEY set)

```bash
PROMPT_TEXT="You are [PERSONA]. [PROBLEM]. [OUTPUT SCHEMA]. Be concrete and commit to a stance."

curl -s https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d "$(python3 -c "import json,sys; print(json.dumps({'model':'gpt-4o','temperature':0.8,'messages':[{'role':'user','content': sys.argv[1]}]}))" "$PROMPT_TEXT")" \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['choices'][0]['message']['content'])" \
  > "$SMAC_DIR/agent_openai.md" 2>&1
echo "OpenAI agent done (exit $?)"
```

---

### Practical Tips

- Spawn all Claude subagents in one `Task` call batch → they run truly in parallel
- Fire all Bash agents (Ollama/Gemini/OpenAI) as parallel Bash calls in the same message
- Set 3-minute timeout for most Ollama models (`timeout 180`); large 30b models may need 5 min (`timeout 300`)
- If Gemini hits a rate limit or returns nothing, exclude it and don't retry — note the effective N
- If an agent returns an empty file or obvious error, exclude from aggregation and note effective N
- After all agents finish, sanity-check: are any output files empty or clearly truncated?

## Phase 4: Collect and Aggregate

Once all agents complete, read all `$SMAC_DIR/agent_*.md` files.

Track for each output: which backend, which model, which persona, and key claims made.

### Aggregation by Task Type

**Ranking (Borda Count):**
1st = N pts, 2nd = N-1 pts, ... last = 1 pt. Sum across agents per option. Rank by total.

**Recommendation (Grouping):**
- Group semantically similar ideas (same recommendation, different wording)
- **Consensus** (≥60% agents): High confidence — safe bet
- **Divergence** (30-60%): Genuine judgment call — flag for user decision
- **Valuable Outlier** (<30%): Flag if proposed by Skeptic, Domain Expert, or Edge-Case Hunter with strong reasoning — minority-but-correct ideas are gold

**Decision (Vote):**
Count votes per option. Report split + strongest argument from each side.

**Synthesis / Brainstorm:**
Proceed to Phase 5 (synthesis judge).

### Always Surface

- Do disagreements correlate with model/backend (systematic bias)? Or with persona (genuine judgment call)?
- What does the Skeptic see that optimists missed?
- Is there a Valuable Outlier — a minority idea endorsed by a specialized persona with a compelling argument?

## Phase 5: Synthesis (for Synthesis/Brainstorm tasks)

When aggregation alone is insufficient, run one final synthesis pass using the **strongest available model** (Claude opus preferred; else sonnet; else best available):

```
You are synthesizing [N] independent expert analyses of the following problem.

PROBLEM: [problem]

ANALYSES:
[all agent outputs, labeled by persona]

YOUR TASK:
1. What do most agents agree on? (consensus — the safe foundation)
2. Where do they genuinely disagree? (divergence — real judgment calls needing human input)
3. Is there a minority insight that's high-impact even if few agents raised it?
4. Produce a SYNTHESIS — not a summary, but a response better than any individual agent.

Do NOT average everything together. Preserve the strongest specific insights.
Explicitly call out where you think the majority is wrong — and if the majority made a clear factual or logical error, override them with transparent disclosure: state what the majority concluded, why it is wrong, and what the correct answer is. An honest override is more valuable than a false consensus.
```

## Phase 6: Final Output

**Always present this structure:**

```markdown
# SMAC Report

**Problem**: [problem]
**Agents**: [N total] — [breakdown: e.g., "3× Claude (haiku/sonnet/opus), 2× Ollama (qwen3-coder, phi3.5), 1× Gemini"]
**Task type**: [ranking / recommendation / decision / synthesis / brainstorm]

## Consensus ([X]/[N] agents agree)
[Main finding — what the council converged on. Be specific.]

## Key Divergence
[The most interesting split. What each side argues. Why it's a genuine judgment call — not just noise.]

## Valuable Outlier
[The minority idea worth examining. Which agent proposed it. Why it might matter despite being unpopular.]

## Orchestrator Synthesis
[Your synthesized recommendation, integrating the council's best insights. Take a stance.]

---
*Full agent outputs: [SMAC_DIR]/agent_*.md*
*Effective N: [N] (note any failed agents)*
```

## Configuration

| Parameter | Default | How to override |
|-----------|---------|-----------------|
| N agents | 5–10 based on discovery | "use 15 agents", "spawn 3 agents" |
| Synthesis model | opus (if available) | "use sonnet for synthesis" |
| Agent models | auto-selected by backend | "only use haiku", "use qwen3-coder:30b for all ollama" |
| Min N | 3 | cannot lower |

## Edge Cases

- **Only Claude available**: Use haiku×2 + sonnet×2 + opus×1 = 5 agents. Still genuinely diverse.
- **Ambiguous prompt**: Ask ONE clarifying question. Don't spawn on vagueness.
- **No consensus**: Report the split honestly — this IS the finding. The problem has no dominant answer. Give the user the strongest arguments from each side.
- **All agents agree (unanimity)**: Report as high-confidence consensus and note unanimity explicitly.
- **Ollama slow or timeout**: Note the agent as excluded, report effective N used.
- **Agent returns garbage**: Exclude from aggregation, note effective N.
- **User specifies N > 10**: Honor it but note cost implications (especially for large Ollama models or paid APIs).

## Cost Awareness

Tell the user what you're about to spawn before starting:
> "Spawning [N] agents: [breakdown]. Ollama is free/local. Claude costs ~$0.01–0.05/agent depending on model."

For expensive tasks (many agents × long outputs):
- Prefer haiku + smaller Ollama models for proposers
- Reserve opus/gpt-4o for synthesis only
