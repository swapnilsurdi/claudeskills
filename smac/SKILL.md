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

**Why this works:** Different models have different training data, architectures, and failure modes. Different personas surface different blind spots. Aggregating across genuine diversity cancels out random errors and reveals true disagreements worth examining.

**The cardinal rule:** A false consensus is worse than no answer. If agents fail to produce valid output, abort loudly — never synthesize from silence.

---

## Phase 0: Mode + Backend Discovery

### Step 0a — Pick a mode

| Mode | Agents | When to use |
|------|--------|-------------|
| **quick** | 3 agents | Fast sanity check, casual opinion, time-sensitive |
| **standard** | 5–7 agents | Default — most decisions, brainstorming, analysis |
| **deep** | 10+ agents | High-stakes, irreversible, or genuinely contested decisions |

Default to **standard** unless the user specifies otherwise or the task is clearly quick/trivial.

### Step 0b — Pre-test every backend before counting it

Detection ≠ dispatch. Probe each backend with a tiny request *before* adding it to the roster. A backend that appears available but fails silently will produce empty output and corrupt your effective N.

```bash
SMAC_DIR="/tmp/smac_$(date +%s)"
mkdir -p "$SMAC_DIR"

echo "=== Backend Pre-Tests ==="

# Claude subagents: available if you can invoke the Task tool right now.
# If you are running inside a subagent context, Task may not be available.
# Test by checking whether you have it — do NOT assume.

# Ollama: test a real small-model round-trip (use the smallest available model)
if which ollama &>/dev/null; then
  OLLAMA_TEST=$(ollama run qwen2.5:3b "Reply with the single word: OK" 2>/dev/null | tr -d '[:space:]')
  [ "$OLLAMA_TEST" = "OK" ] || echo "$OLLAMA_TEST" | grep -qi "ok" \
    && echo "Ollama: CONFIRMED" \
    || echo "Ollama: PROBE FAILED — excluding (got: $OLLAMA_TEST)"
else
  echo "Ollama: not installed"
fi

# OpenAI: test key validity with a minimal call
if [ -n "$OPENAI_API_KEY" ]; then
  OAI_TEST=$(curl -s -o /dev/null -w "%{http_code}" \
    https://api.openai.com/v1/models \
    -H "Authorization: Bearer $OPENAI_API_KEY")
  [ "$OAI_TEST" = "200" ] && echo "OpenAI: CONFIRMED (key valid)" \
    || echo "OpenAI: PROBE FAILED (HTTP $OAI_TEST) — excluding"
else
  echo "OpenAI: no OPENAI_API_KEY"
fi

# Gemini CLI: free tier rate-limits aggressively — test before counting
if which gemini &>/dev/null; then
  GEMINI_TEST=$(timeout 15 gemini "Reply with the single word: OK" 2>/dev/null | head -1)
  echo "$GEMINI_TEST" | grep -qi "ok\|sure\|yes" \
    && echo "Gemini: CONFIRMED" \
    || echo "Gemini: PROBE FAILED (rate-limited or slow) — excluding"
else
  echo "Gemini: not installed"
fi

echo "Working dir: $SMAC_DIR"
```

**Build the roster only from backends that CONFIRMED.** Then tell the user before spawning:

> "Running SMAC [mode] with [N] agents: [confirmed breakdown]. Estimated time: [X] min. Ollama is free/local; Claude costs ~$0.01–0.05/agent. Starting now."

If no backends confirm and Task tool is also unavailable: **stop here** and tell the user — do not fake a SMAC run with a single-thread analysis. That defeats the purpose entirely.

### Step 0c — Load Prior Learnings

SMAC improves over time by remembering which models made which kinds of errors. Load the learnings file before building agent prompts:

```bash
LEARNINGS_FILE="$HOME/.claude/smac/learnings.jsonl"
mkdir -p "$HOME/.claude/smac"

if [ -f "$LEARNINGS_FILE" ]; then
  echo "=== Prior Learnings ==="
  # Show recent entries (last 30 days) relevant to confirmed backends
  python3 - "$LEARNINGS_FILE" << 'PY'
import json, sys
from datetime import datetime, timezone, timedelta

cutoff = datetime.now(timezone.utc) - timedelta(days=30)
entries = []
with open(sys.argv[1]) as f:
    for line in f:
        line = line.strip()
        if not line:
            continue
        try:
            e = json.loads(line)
            ts = datetime.fromisoformat(e.get("timestamp","").replace("Z","+00:00"))
            if ts >= cutoff:
                entries.append(e)
        except Exception:
            pass

if not entries:
    print("No recent learnings.")
else:
    # Group by backend/model
    by_backend = {}
    for e in entries:
        key = e.get("backend","unknown")
        by_backend.setdefault(key, []).append(e)
    for backend, items in by_backend.items():
        print(f"\n{backend} ({len(items)} past issue(s)):")
        for item in items[-3:]:  # show up to 3 most recent per backend
            print(f"  [{item.get('error_type','?')}] {item.get('lesson', item.get('description',''))}")
PY
else
  echo "No learnings file yet — this will be created after the first run."
fi
```

Use the loaded learnings to:
1. **Flag known-unreliable backends** for the current domain — note them in the roster but mark with ⚠️
2. **Inject relevant warnings into the synthesis prompt** (e.g., "Note: phi3.5 has previously made spatial reasoning errors on home design tasks — weight its geometric claims cautiously")
3. **Not** exclude backends solely based on past failures — one bad run doesn't disqualify a model, but it should raise the scrutiny bar

---

## Phase 1: Understand the Problem

Before generating personas or spawning agents:

1. **Domain**: (e.g., Software Engineering, Travel, Home Design, Business Strategy, Legal, Creative Writing, Life Decisions)
2. **Task type** — pick the aggregation strategy:
   - **Ranking** → Borda count
   - **Recommendation** → semantic grouping + consensus threshold
   - **Decision (binary/multi-choice)** → vote count + argument summary
   - **Synthesis / Open-ended** → synthesis judge pass
   - **Brainstorm** → grouping + valuable outlier detection
3. **Output schema**: Define what each agent *must* return before spawning. If you can't aggregate it mechanically, tighten the schema.
4. If the problem is **ambiguous**, ask one clarifying question. Burning N × tokens on vagueness is wasteful.

---

## Phase 2: Generate Domain-Specific Personas

Generate 4 base personas relevant to the domain, cycle for N > 4:

| Archetype | Role | Anti-Sycophancy framing |
|-----------|------|------------------------|
| **The Domain Expert** | Deep knowledge, established best practices | "Apply your expertise rigorously — don't hedge" |
| **The Skeptic / Devil's Advocate** | Finds holes, questions assumptions, surfaces risks | "Your job is to find what everyone else gets wrong" |
| **The Pragmatist** | What actually works with real constraints | "Ignore theory. What works in the real world?" |
| **The Visionary / Contrarian** | Challenges conventions, non-obvious angles | "Conventional wisdom is your enemy. What does everyone miss?" |
| **The Edge-Case Hunter** | Rare failures, second-order effects | "Map what breaks. What are the third-order effects?" |
| **The User / Recipient** | End-user perspective, human impact | "Optimize for the person who lives with this decision" |

Adapt these to the domain (e.g., for travel: "The Budget Traveler", "The Safety-First Planner", "The Off-the-Beaten-Path Explorer").

---

## Phase 3: Spawn All Agents — In Parallel

**CRITICAL**: Start all agents in the same turn. An agent that sees another's output before generating its own collapses into sycophancy and defeats the entire approach.

Each agent writes its output to `$SMAC_DIR/agent_<id>.md`.

---

### Claude Subagents (Task tool)

Use `subagent_type: "general-purpose"` with appropriate `model` (haiku/sonnet/opus). Spawn all Claude agents in one batch so they run in parallel.

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

### Ollama (Bash)

**Important:** Pass the prompt as a command-line argument, not via stdin redirection. Background bash jobs get stdin from `/dev/null`, so `< file` produces an empty prompt and a useless response. Use `$(cat file)` as the argument instead.

```bash
cat > "$SMAC_DIR/prompt_[id].txt" << 'PROMPT'
You are [PERSONA]. [PROBLEM]. [OUTPUT SCHEMA]. Be concrete and commit to a stance.
PROMPT

timeout 180 ollama run [model] "$(cat "$SMAC_DIR/prompt_[id].txt")" \
  > "$SMAC_DIR/agent_[id].md" 2>&1
echo "Ollama [model] agent [id]: exit $?"
```

Good model choices:
- **Complex reasoning**: `qwen3-coder:30b` (use `timeout 300` — slow)
- **Fast opinion**: `qwen2.5:3b` or `gemma2:2b`
- **General purpose**: `phi3.5` or `qwen2.5-coder:7b`

---

### Gemini CLI (Bash)

```bash
cat > "$SMAC_DIR/prompt_gemini.txt" << 'PROMPT'
You are [PERSONA]. [PROBLEM]. [OUTPUT SCHEMA]. Be concrete and commit to a stance.
PROMPT

timeout 30 gemini "$(cat "$SMAC_DIR/prompt_gemini.txt")" \
  > "$SMAC_DIR/agent_gemini.md" 2>&1
echo "Gemini agent: exit $?"
```

---

### OpenAI via curl (Bash)

```bash
PROMPT_TEXT="You are [PERSONA]. [PROBLEM]. [OUTPUT SCHEMA]. Be concrete and commit to a stance."

curl -s https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d "$(python3 -c "
import json, sys
print(json.dumps({
  'model': 'gpt-4o',
  'temperature': 0.8,
  'messages': [{'role': 'user', 'content': sys.argv[1]}]
}))" "$PROMPT_TEXT")" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(d['choices'][0]['message']['content'])" \
  > "$SMAC_DIR/agent_openai.md" 2>&1
echo "OpenAI agent: exit $?"
```

---

## Phase 3.5: Validate Outputs — Quorum Check

**Do this before aggregating anything.** Empty outputs and error messages are not data points — including them silently is the most dangerous failure mode in multi-agent systems.

```bash
echo "=== Output Validation ==="
VALID_COUNT=0
for f in "$SMAC_DIR"/agent_*.md; do
  SIZE=$(wc -c < "$f" 2>/dev/null || echo 0)
  # Check for common failure patterns
  if [ "$SIZE" -lt 100 ]; then
    echo "EXCLUDED $f — too short (${SIZE} bytes, likely empty or error)"
  elif grep -qi "rate.limit\|quota\|error\|not logged in\|unauthorized\|refused" "$f"; then
    echo "EXCLUDED $f — contains error indicator"
    mv "$f" "${f}.failed"
  else
    echo "OK $f (${SIZE} bytes)"
    VALID_COUNT=$((VALID_COUNT + 1))
  fi
done
echo "Valid outputs: $VALID_COUNT"
```

**If VALID_COUNT < 3: ABORT.**

Do not synthesize. Tell the user:

> "SMAC quorum not reached: only [N] of [total] agents produced valid output. This is not enough for meaningful consensus — a false answer is worse than no answer. Try again with: [specific fix — e.g., 'check Ollama is running', 'verify API key', 'use quick mode with Claude-only']"

Only proceed to Phase 4 if VALID_COUNT ≥ 3.

---

## Phase 4: Collect and Aggregate

Read all *valid* `$SMAC_DIR/agent_*.md` files (skip `.failed` files).

Track for each: backend, model, persona, key claims.

### Aggregation by Task Type

**Ranking (Borda Count):**
1st = N pts, 2nd = N-1 pts, ... last = 1 pt. Sum per option across valid agents.

**Recommendation (Grouping):**
- **Consensus** (≥60% of valid agents): High confidence
- **Divergence** (30–60%): Genuine judgment call — flag for user
- **Valuable Outlier** (<30%): Flag if endorsed by Skeptic, Domain Expert, or Edge-Case Hunter with strong reasoning

**Decision (Vote):**
Count per option. Report split + strongest argument from each side.

**Synthesis / Brainstorm:**
Proceed to Phase 5.

### Always Surface

- Do disagreements correlate with model/backend (systematic bias) or with persona (genuine judgment call)?
- What does the Skeptic see that optimists missed?
- Is there a Valuable Outlier — minority idea endorsed by a specialized persona with a compelling argument?

---

## Phase 5: Synthesis Pass

For Synthesis/Brainstorm tasks, or when aggregation alone is insufficient, run one final synthesis using the **strongest available model** (opus preferred; else sonnet):

```
You are synthesizing [N] independent expert analyses.

PROBLEM: [problem]

[If learnings from Phase 0c are relevant to any of these backends, include them here:]
KNOWN MODEL ISSUES (from prior runs):
[e.g., "ollama/phi3.5 has previously made spatial reasoning errors on home design tasks — weigh its geometric claims carefully"]

ANALYSES (labeled by persona and backend):
[all valid agent outputs]

YOUR TASK:
1. What do most agents agree on? (consensus — the safe foundation)
2. Where do they genuinely disagree? (divergence — real judgment calls for the human)
3. Is there a minority insight that's high-impact even if few agents raised it?
4. Produce a SYNTHESIS — not a summary, but a response better than any individual agent.

Do NOT average everything together. Preserve the strongest specific insights.
If the majority made a clear factual or logical error, override them — but disclose it:
state what the majority said, why it's wrong, and what the correct answer is.
An honest override is more valuable than a false consensus.
```

---

## Phase 6: Final Output

```markdown
# SMAC Report

**Problem**: [problem]
**Mode**: [quick / standard / deep]
**Agents**: [valid N] of [attempted N] — [breakdown by backend/model]
**Task type**: [ranking / recommendation / decision / synthesis / brainstorm]

## Consensus ([X]/[N] agents agree)
[Main finding — what the council converged on. Be specific.]

## Key Divergence
[The most interesting split. What each side argues. Why it's a genuine judgment call.]

## Valuable Outlier
[The minority idea worth examining. Which agent/persona proposed it. Why it might matter.]

## Orchestrator Synthesis
[Your synthesized recommendation. Take a stance.]

---
*Agent outputs: [SMAC_DIR]/agent_*.md*
*Failed agents: [list any .failed files and why]*
```

---

## Phase 6.5: Record Learnings

After every run — successful or not — record what happened so future runs start smarter. This is what makes SMAC get better over time rather than repeating the same failures.

**Always run this.** Even a perfect run is worth recording (it confirms a backend is reliable).

```bash
LEARNINGS_FILE="$HOME/.claude/smac/learnings.jsonl"
RUN_TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
```

### What to record

**1. Any agent that was excluded or failed (Phase 3.5):**

```bash
for f in "$SMAC_DIR"/agent_*.failed; do
  [ -f "$f" ] || continue
  BACKEND=$(basename "$f" | sed 's/agent_//' | sed 's/\.md\.failed//')
  ERROR_CONTENT=$(head -5 "$f")
  # Classify the failure type
  echo "$ERROR_CONTENT" | grep -qi "rate.limit\|quota" && ERROR_TYPE="rate_limit" || \
  echo "$ERROR_CONTENT" | grep -qi "timeout" && ERROR_TYPE="timeout" || \
  echo "$ERROR_CONTENT" | grep -qi "unauthorized\|401\|not logged in" && ERROR_TYPE="auth_failure" || \
  ERROR_TYPE="output_failure"

  python3 - << PY >> "$LEARNINGS_FILE"
import json
print(json.dumps({
  "timestamp": "$RUN_TIMESTAMP",
  "run_id": "smac_$(date +%s)",
  "domain": "[DOMAIN from Phase 1]",
  "task_type": "[TASK TYPE from Phase 1]",
  "backend": "$BACKEND",
  "error_type": "$ERROR_TYPE",
  "description": "Agent excluded during quorum check",
  "lesson": "Verify $BACKEND is available before counting it in the roster for [DOMAIN] tasks",
  "severity": "medium",
  "confirmed_by": "quorum_validation"
}))
PY
done
```

**2. Any orchestrator override (you corrected a majority error in Phase 5):**

If you overrode the majority during synthesis, record what the majority got wrong and which model(s) drove it:

```python
# Write one entry per model that contributed to the wrong majority
import json
entry = {
  "timestamp": "[RUN_TIMESTAMP]",
  "run_id": "smac_[timestamp]",
  "domain": "[DOMAIN]",
  "task_type": "[TASK_TYPE]",
  "backend": "[backend/model that led the wrong majority — e.g., 'ollama/phi3.5']",
  "error_type": "systematic_bias",
  "description": "[One sentence: what the majority concluded and why it was wrong]",
  "lesson": "[One sentence: what to watch for when this backend handles this domain in future]",
  "severity": "high",
  "confirmed_by": "orchestrator_override"
}
# Append to learnings file
with open(f"{home}/.claude/smac/learnings.jsonl", "a") as f:
    f.write(json.dumps(entry) + "\n")
```

**3. User explicitly flags something as wrong (after presenting the report):**

If the user says "that advice was bad" or "agent X was clearly wrong", record it:

```python
entry = {
  "timestamp": "[RUN_TIMESTAMP]",
  "backend": "[backend/model the user flagged]",
  "error_type": "user_rejected",
  "description": "[what the agent claimed]",
  "lesson": "[what the user said was actually correct]",
  "severity": "high",
  "confirmed_by": "user_feedback"
}
```

### What NOT to record

- Agents that simply disagreed with the majority but weren't *wrong* — dissent is a feature
- Timeouts on large Ollama models when running complex tasks — that's expected behavior, not a bug
- One-off failures that are clearly environmental (network blip, Gemini free tier)

### Pruning

The learnings file is append-only and will grow. The load step (Phase 0c) already filters to the last 30 days. No manual pruning needed.

---

## Configuration

| Parameter | Default | Override |
|-----------|---------|---------|
| Mode | standard (5–7 agents) | "quick smac", "deep smac", "use 3 agents" |
| Synthesis model | opus (if available) | "use sonnet for synthesis" |
| Agent models | auto-selected by backend | "only use haiku", "use qwen3-coder for ollama agents" |
| Min valid quorum | 3 | cannot lower — abort instead |

---

## Edge Cases

- **Only Claude available**: haiku×2 + sonnet×2 + opus×1 = 5 agents. Still genuinely diverse.
- **Ambiguous prompt**: Ask ONE clarifying question before spawning.
- **No consensus**: Report the split — this IS the finding. Give the strongest arguments from each side.
- **Unanimity**: Report as high-confidence and note it explicitly.
- **Quorum failure (<3 valid outputs)**: Abort. Do not synthesize. Explain what failed and how to fix it.
- **All Ollama agents time out**: Reduce to faster models (qwen2.5:3b, gemma2:2b) or drop to quick mode.
- **Gemini rate-limited**: It failed the pre-test and should not be in the roster — exclude silently.
- **N > 10 requested**: Honor it, note cost and time implications upfront.
