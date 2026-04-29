# claudeskills

Claude Code skills for getting more out of AI — starting with SMAC.

---

## SMAC — Stochastic Multi-Agent Consensus

Instead of asking one AI, SMAC spawns a diverse council of agents across every LLM backend you have available, collects their independent answers, and synthesizes a consensus stronger than any single model could produce.

Works for anything: relocating to a new city, arranging furniture, choosing a database, brainstorming a business, legal questions, travel planning, technical architecture — any problem where a single AI opinion feels insufficient.

### How it works

```
Your question
     │
     ▼
Phase 0: Discover + pre-test all available backends
     │   (Claude haiku/sonnet/opus, Ollama local models,
     │    OpenAI GPT-4o, Gemini CLI — only confirmed ones join)
     │
     ▼
Phase 3: Spawn N agents in parallel — each with a different
         model AND a different expert persona
         (The Skeptic, The Domain Expert, The Pragmatist,
          The Visionary, The Edge-Case Hunter...)
     │
     ▼
Phase 3.5: Validate outputs — quorum check
           (< 3 valid outputs → abort, never fake a consensus)
     │
     ▼
Phase 4: Aggregate results by task type
         (Borda count for rankings, semantic grouping for
          recommendations, vote count for decisions)
     │
     ▼
Phase 5: Synthesis pass by strongest available model
         (Honest override if majority made a factual error)
     │
     ▼
SMAC Report: Consensus · Key Divergence · Valuable Outlier
```

Over time, SMAC learns from its own mistakes — a `~/.claude/smac/learnings.jsonl` file records which models made which types of errors on which domains, and future runs use that to weight the synthesis appropriately.

### Supported backends (auto-detected)

| Backend | What it uses | Cost |
|---------|-------------|------|
| Claude subagents | haiku / sonnet / opus | ~$0.01–0.05/agent |
| Ollama | any locally installed model | Free (local) |
| OpenAI | gpt-4o via curl | Pay-per-token |
| Gemini CLI | gemini (default model) | Free tier (rate-limited) |

Only Claude subagents are required. All others are optional and auto-detected at runtime.

### Install

```bash
# Install from this repo
claude skills install https://github.com/swapnilsurdi/claudeskills/raw/main/smac.skill
```

Or clone and install locally:

```bash
git clone https://github.com/swapnilsurdi/claudeskills
claude skills install ./claudeskills/smac.skill
```

### Usage

Just invoke it in Claude Code:

```
/smac Should I use React or Vue for my new project?
```

```
smac what are the best cities for a remote engineer who loves hiking?
```

```
get consensus on whether we should rewrite our auth service in Go
```

```
poll agents about the best way to arrange my living room furniture
```

SMAC also triggers automatically when Claude detects you'd benefit from multiple perspectives — stress-testing an idea, comparing options, or asking "what would different experts say about this?"

### Modes

| Mode | Agents | Use when |
|------|--------|----------|
| `quick smac` | 3 | Fast sanity check, time-sensitive |
| `smac` (default) | 5–7 | Most decisions and analysis |
| `deep smac` | 10+ | High-stakes or genuinely contested decisions |

### What makes this different

Most multi-agent skills use the same model with slightly different prompts. SMAC combines:

- **True model diversity** — different architectures have different training data, biases, and failure modes. Claude + Ollama qwen3-coder + GPT-4o will disagree in genuinely different ways.
- **Backend auto-discovery** — no config required. SMAC probes each backend before counting it, so a rate-limited Gemini or a missing API key never silently corrupts your effective N.
- **Quorum enforcement** — if fewer than 3 agents produce valid output, SMAC aborts rather than synthesizing from silence. A false consensus is worse than no answer.
- **Transparent override** — if the majority made a clear factual error (our furniture eval: 3/5 Ollama models recommended placing a TV facing direct window glare), the synthesis model overrides them and explains why.
- **Self-calibration** — learnings persist across runs. After phi3.5 makes a spatial reasoning error once, future home design runs automatically flag its geometric claims for extra scrutiny.

---

## Adding more skills

This repo will grow. PRs welcome — see the [skill-creator plugin](https://github.com/anthropics/claude-plugins-official) for the authoring workflow.
