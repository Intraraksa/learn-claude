# Building Collaborative Business Research Agents with Claude Code
### A Step-by-Step Guide by a Claude Code Expert at Anthropic

---

## Table of Contents
1. [Understanding the Architecture](#1-understanding-the-architecture)
2. [Prerequisites & Setup](#2-prerequisites--setup)
3. [How Claude Code Multi-Agent Works](#3-how-claude-code-multi-agent-works)
4. [Building Your Project Structure](#4-building-your-project-structure)
5. [Designing Each Agent](#5-designing-each-agent)
6. [The Orchestrator: Making Agents Collaborate](#6-the-orchestrator-making-agents-collaborate)
7. [The CLAUDE.md — Your Agents' Brain](#7-the-claudemd--your-agents-brain)
8. [Running Your Agent System](#8-running-your-agent-system)
9. [Advanced Patterns](#9-advanced-patterns)
10. [Real Example: Full Business Opportunity Research](#10-real-example-full-business-opportunity-research)
11. [Troubleshooting & Best Practices](#11-troubleshooting--best-practices)

---

## 1. Understanding the Architecture

Before writing a single line, understand the two ways Claude Code lets agents collaborate:

### Option A — Subagents (Simpler, Recommended to Start)
```
You (Main Session)
    └── Orchestrator Agent (Claude Code)
            ├── Research Agent     → runs as subagent, reports back
            ├── Marketing Agent    → runs as subagent, reports back
            ├── Financial Agent    → runs as subagent, reports back
            └── Analysis Agent     → runs as subagent, synthesizes all
```
- Each subagent has its **own isolated context window**
- They run in parallel or sequentially based on dependencies
- They **report back** to the orchestrator when done
- Best for: "do your job, return results"

### Option B — Agent Teams (Experimental, More Powerful)
```
Team Lead (Orchestrator)
    ├── Research Teammate  ←→ can message each other directly
    ├── Marketing Teammate ←→ can message each other directly
    ├── Financial Teammate ←→ can message each other directly
    └── Analysis Teammate  ←→ can message each other directly
```
- Teammates can **communicate directly** without going through the lead
- Enable with: `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true`
- Best for: "work together, challenge each other, discover together"

> **Recommendation:** Start with Subagents (Option A). Once comfortable, graduate to Agent Teams for complex research.

---

## 2. Prerequisites & Setup

### Install Claude Code
```bash
npm install -g @anthropic-ai/claude-code
```
Requires Node.js 18+. Verify:
```bash
claude --version
```

### Set Your API Key
```bash
export ANTHROPIC_API_KEY=sk-ant-...
# Or add to ~/.bashrc / ~/.zshrc for persistence
```

### Enable Agent Teams (Optional, for advanced use)
```bash
# In your project directory, create settings file
mkdir -p .claude
cat > .claude/settings.json << 'EOF'
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "true"
  }
}
EOF
```

---

## 3. How Claude Code Multi-Agent Works

### The Task Tool — How Subagents Are Spawned
When Claude Code sees a complex task, it can spawn subagents using its internal `Task()` tool. You **guide** this by writing clear CLAUDE.md files and prompts.

```
Main Agent thinks: "I need to research 3 things in parallel"
  → Spawns Task A (Research Agent)   → runs in background
  → Spawns Task B (Marketing Agent)  → runs in background  
  → Waits for both to complete
  → Spawns Task C (Analysis Agent)   → uses results from A and B
  → Synthesizes final report
```

### Shared Memory — How Agents Exchange Data
Agents communicate through **files on disk**. This is the key insight:

```
/project/
  shared/
    research_output.md      ← Research agent writes here
    marketing_output.md     ← Marketing agent writes here
    financial_output.md     ← Financial agent writes here
    analysis_final.md       ← Analysis agent reads all, writes synthesis
```

Each agent:
1. **Reads** relevant previous outputs
2. **Does its specialized work**
3. **Writes** its output to a shared file
4. **Reports completion** to orchestrator

---

## 4. Building Your Project Structure

Create this exact folder structure:

```bash
mkdir -p business-agents/{agents,shared,outputs,templates,scripts}
cd business-agents
```

```
business-agents/
├── CLAUDE.md                    ← Master instructions (the brain)
├── .claude/
│   └── settings.json            ← Claude Code config
├── agents/
│   ├── orchestrator.md          ← Orchestrator agent prompt
│   ├── research_agent.md        ← Research agent prompt
│   ├── marketing_agent.md       ← Marketing agent prompt
│   ├── financial_agent.md       ← Financial agent prompt
│   └── analysis_agent.md        ← Analysis agent prompt
├── shared/
│   ├── task_queue.json          ← What needs to be done
│   ├── research_output.md       ← Research results (written by research agent)
│   ├── marketing_output.md      ← Marketing results
│   ├── financial_output.md      ← Financial results
│   └── status.json              ← Agent status tracking
├── outputs/
│   └── final_report.md          ← Final synthesized report
├── templates/
│   └── task_template.json       ← Template for new tasks
└── scripts/
    └── start_research.sh        ← Launch script
```

Create the structure:
```bash
touch CLAUDE.md
mkdir -p .claude && touch .claude/settings.json
touch agents/{orchestrator,research_agent,marketing_agent,financial_agent,analysis_agent}.md
touch shared/{task_queue,status}.json
touch scripts/start_research.sh && chmod +x scripts/start_research.sh
```

---

## 5. Designing Each Agent

Each agent is defined in a **Markdown file** that gives Claude its role, rules, tools, and output format.

### 5.1 — Research Agent (`agents/research_agent.md`)
```markdown
# Research Agent

## Your Identity
You are the Research Agent. Your ONE job is to discover new product opportunities 
in a given market. You are thorough, data-driven, and systematic.

## Your Mission
Given a market or industry, research:
1. Unmet customer needs and pain points
2. Emerging trends (using web search)
3. Competitor gaps and whitespace
4. New technologies enabling new products
5. Customer segments being underserved

## Tools You Use
- Web search: Find recent market reports, news, Reddit/forum discussions
- File read: Check shared/task_queue.json for your specific task
- File write: Write ALL findings to shared/research_output.md

## Output Format
Write your output to `shared/research_output.md` in this structure:

```
# Research Report
**Date:** [date]
**Market:** [market name]
**Agent:** Research Agent

## Top 3 Product Opportunities Found
### Opportunity 1: [Name]
- Problem: ...
- Evidence: ...
- Market Size: ...
- Source: ...

### Opportunity 2: [Name]
...

## Raw Data & Sources
...

## Status: COMPLETE
```

## Completion Rule
You are DONE only when `shared/research_output.md` contains "Status: COMPLETE".
Do not stop until the file is written and complete.
```

### 5.2 — Marketing Agent (`agents/marketing_agent.md`)
```markdown
# Marketing Agent

## Your Identity
You are the Marketing Agent. You identify business opportunities, channels, 
and positioning strategies. You think like a growth hacker meets strategist.

## Your Mission
Given the Research Agent's findings, identify:
1. Target customer segments and their demographics
2. Best acquisition channels (B2B/B2C/D2C)
3. Competitor positioning and how to differentiate
4. Pricing models that work in this space
5. Go-to-market strategy options

## Input
FIRST read `shared/research_output.md` to understand what products were found.
Then conduct your own research to validate and expand on marketing angles.

## Output Format
Write all findings to `shared/marketing_output.md`:

```
# Marketing Intelligence Report
**Date:** [date]
**Agent:** Marketing Agent
**Based On:** Research Agent findings

## Target Customer Segments
### Primary Segment
- Demographics: ...
- Pain Points: ...
- Willingness to Pay: ...

## Go-To-Market Strategy
...

## Competitive Positioning
...

## Recommended Channels
...

## Status: COMPLETE
```

## Dependency Rule
Do NOT start until `shared/research_output.md` has "Status: COMPLETE".
Check the file first. If not ready, wait 10 seconds and check again.
```

### 5.3 — Financial Agent (`agents/financial_agent.md`)
```markdown
# Financial Agent

## Your Identity
You are the Financial Agent. You assess financial viability, model unit economics, 
and identify funding/monetization paths. You think like a startup CFO.

## Your Mission
For the top product opportunities identified, assess:
1. Estimated market size (TAM, SAM, SOM)
2. Unit economics (CAC, LTV, margins)
3. Revenue model options
4. Capital required to launch
5. Break-even timeline estimate
6. Risk factors

## Input
Read BOTH:
- `shared/research_output.md` — for the product opportunities
- `shared/marketing_output.md` — for pricing and customer data

## Output Format
Write to `shared/financial_output.md`:

```
# Financial Assessment Report
**Date:** [date]
**Agent:** Financial Agent

## Market Sizing
- TAM: $X billion
- SAM: $X million  
- SOM (Year 1): $X million

## Per Opportunity Analysis
### Opportunity 1: [Name]
- Revenue Model: ...
- Estimated CAC: $X
- Estimated LTV: $X
- LTV:CAC Ratio: X
- Gross Margin: X%
- Capital to Launch: $X
- Break-even: X months

## Risk Matrix
...

## Status: COMPLETE
```

## Dependency Rule
Do NOT start until BOTH research and marketing outputs have "Status: COMPLETE".
```

### 5.4 — Analysis Agent (`agents/analysis_agent.md`)
```markdown
# Analysis Agent

## Your Identity
You are the Analysis Agent — the synthesizer. You wait for ALL other agents 
to finish, then combine their findings into one definitive business recommendation.

## Your Mission
Read all three reports and produce a final executive recommendation:
1. Rank opportunities by viability score
2. Identify the single best opportunity to pursue
3. Outline a 90-day launch plan
4. Highlight top risks and mitigations
5. Give a clear GO / NO-GO decision per opportunity

## Input — Read ALL of these:
- `shared/research_output.md`
- `shared/marketing_output.md`
- `shared/financial_output.md`

## Output Format
Write to `outputs/final_report.md`:

```
# Executive Business Intelligence Report
**Date:** [date]
**Prepared By:** Analysis Agent (synthesizing all agents)

## TL;DR (30-second read)
[One paragraph with the #1 recommendation]

## Opportunity Scorecard
| Opportunity | Market Score | Financial Score | Execution Score | TOTAL |
|-------------|-------------|-----------------|-----------------|-------|
| Opp 1       | 8/10        | 7/10            | 9/10            | 24/30 |

## #1 Recommended Opportunity
### [Name]
**Decision: GO ✅**
- Why: ...
- 90-Day Launch Plan: ...
- Budget Required: ...
- Key Risks: ...

## Other Opportunities
...

## Status: COMPLETE — REPORT DELIVERED
```

## Critical Rule
You are the LAST agent. Only start when ALL three input files have "Status: COMPLETE".
Your output file is the deliverable the human asked for.
```

---

## 6. The Orchestrator: Making Agents Collaborate

The orchestrator is the most important piece. It controls the workflow.

### `agents/orchestrator.md`
```markdown
# Orchestrator Agent — Business Research Command Center

## Your Role
You are the master orchestrator. You receive a business research task from the human,
break it into specialized sub-tasks, assign them to agents, monitor progress, 
and ensure the final report is delivered. You do NOT stop until final_report.md is complete.

## Workflow — Execute in This Exact Order

### Phase 1: Initialize
1. Read the human's task
2. Update `shared/task_queue.json` with the specific research topic
3. Reset all output files to empty / "PENDING" state

### Phase 2: Parallel Research (Spawn Both Simultaneously)
Spawn these TWO subagents IN PARALLEL (run_in_background: true):
- **Research Agent**: "Read agents/research_agent.md and execute your mission for this topic: [TOPIC]"
- **Marketing Agent**: Can start in parallel with research but must wait for research output before writing its analysis

### Phase 3: Financial Analysis (After Research + Marketing)
Once research_output.md AND marketing_output.md both show "Status: COMPLETE":
- Spawn **Financial Agent**: "Read agents/financial_agent.md and execute your mission"

### Phase 4: Synthesis (After All Three Complete)
Once all three show "Status: COMPLETE":
- Spawn **Analysis Agent**: "Read agents/analysis_agent.md and produce the final report"

### Phase 5: Delivery
Verify `outputs/final_report.md` contains "Status: COMPLETE — REPORT DELIVERED"
Inform the human the report is ready.

## Status Checking
Between phases, check status by reading `shared/status.json` and the output files.
Poll every 30 seconds if waiting. Maximum wait: 10 minutes per phase.

## Rules
- Never skip phases
- Never spawn Analysis Agent before the other 3 are complete
- If any agent fails, retry once, then report the error
- Update `shared/status.json` after each phase completes
```

---

## 7. The CLAUDE.md — Your Agents' Brain

This is the single most important file. Claude Code reads this automatically in every session.

### `CLAUDE.md`
```markdown
# Business Research Agent System

## What This Project Does
This is a multi-agent business research system. When given a market or product idea,
multiple specialized AI agents collaborate to produce a full business intelligence report.

## Agent Roster
| Agent | File | Job | Writes To |
|-------|------|-----|-----------|
| Orchestrator | agents/orchestrator.md | Coordinates all agents | shared/status.json |
| Research | agents/research_agent.md | Market research | shared/research_output.md |
| Marketing | agents/marketing_agent.md | GTM & positioning | shared/marketing_output.md |
| Financial | agents/financial_agent.md | Unit economics | shared/financial_output.md |
| Analysis | agents/analysis_agent.md | Final synthesis | outputs/final_report.md |

## Execution Order (CRITICAL — Never Skip)
1. Research Agent starts first (or in parallel with Marketing Agent setup)
2. Marketing Agent reads Research output, then runs
3. Financial Agent reads both Research + Marketing, then runs
4. Analysis Agent reads all three, produces final report
5. Orchestrator verifies completion and delivers to human

## Shared State Rules
- ALL agents communicate ONLY via files in `shared/` and `outputs/`
- Agents MUST check dependency files before starting their work
- Agents MUST write "Status: COMPLETE" at the end of their output file
- Agents NEVER modify another agent's output file

## How to Launch
Human says: "Research [TOPIC]" or "Find business opportunity in [MARKET]"
Claude Code reads this file, then reads agents/orchestrator.md, and begins.

## Tools Permitted for All Agents
- Web search (always use for current market data)
- File read/write (within this project only)
- No external API calls unless explicitly approved

## Quality Standards
- Every claim must have a source
- Financial figures must be labeled as estimates
- Opportunities must be ranked by concrete criteria
- Report must be actionable, not generic
```

---

## 8. Running Your Agent System

### Step 1: Initialize the Status File
```bash
cat > shared/status.json << 'EOF'
{
  "session_id": "001",
  "topic": "",
  "phase": "IDLE",
  "agents": {
    "research": "PENDING",
    "marketing": "PENDING",
    "financial": "PENDING",
    "analysis": "PENDING"
  },
  "started_at": "",
  "completed_at": ""
}
EOF
```

### Step 2: Create the Launch Script
```bash
cat > scripts/start_research.sh << 'EOF'
#!/bin/bash
TOPIC="${1:-AI productivity tools for small businesses}"
echo "🚀 Starting Business Research Agent System"
echo "📋 Topic: $TOPIC"
echo ""
claude "I need a complete business research report on this opportunity: $TOPIC

Please act as the Orchestrator Agent (read agents/orchestrator.md for instructions).
Spawn the required subagents in the correct order.
Do not stop until outputs/final_report.md has 'Status: COMPLETE'.
Update shared/status.json throughout the process.

BEGIN NOW."
EOF
chmod +x scripts/start_research.sh
```

### Step 3: Launch with a Topic
```bash
./scripts/start_research.sh "AI-powered tools for restaurant inventory management"
```

Or directly in Claude Code:
```bash
claude
# Then type:
> Research business opportunity: AI tools for restaurant inventory management
```

### Step 4: Monitor Progress
Open a second terminal:
```bash
# Watch status in real-time
watch -n 5 'cat shared/status.json && echo "---" && tail -5 shared/research_output.md'

# Or check final report
cat outputs/final_report.md
```

---

## 9. Advanced Patterns

### Pattern 1: Dynamic Agent Spawning (Agent Count Changes Per Task)
Add to your `CLAUDE.md`:
```markdown
## Dynamic Agent Rules
- If topic is B2B SaaS: add a "Sales Agent" (agents/sales_agent.md)
- If topic involves hardware: add a "Supply Chain Agent"
- If topic is regulated industry: add a "Compliance Agent"
Always adapt the team to the task complexity.
```

### Pattern 2: Agent Teams (Peer-to-Peer Communication)
Enable Agent Teams for agents that need to **debate and challenge each other**:
```json
// .claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "true"
  }
}
```

Then in your orchestrator, agents can send messages directly:
```markdown
## Inter-Agent Communication (Agent Teams Mode)
- Research Agent can send questions to Marketing Agent via Teammate tool
- Marketing Agent can challenge Financial Agent's assumptions
- Any agent can flag blockers to the orchestrator
```

### Pattern 3: Parallel Research Tracks
For larger research jobs, parallelize within the Research Agent:
```markdown
## Research Agent — Parallel Sub-tasks
Spawn 3 sub-research tracks simultaneously:
1. "Research customer problems in [MARKET]"
2. "Research existing solutions and competitors in [MARKET]"  
3. "Research technology enablers and trends in [MARKET]"
Merge all three findings before writing research_output.md
```

### Pattern 4: Hooks for Quality Control
Add hooks to automatically validate outputs:

```bash
# .claude/hooks/post-agent-write.sh
#!/bin/bash
# Run after any agent writes to shared/
FILE=$1
if ! grep -q "Status: COMPLETE" "$FILE"; then
  echo "OUTPUT_INCOMPLETE: $FILE missing completion marker" >&2
  exit 2  # Exit code 2 = send feedback, keep agent working
fi
echo "✅ $FILE validated"
```

Register in `settings.json`:
```json
{
  "hooks": {
    "TaskCompleted": "bash .claude/hooks/post-agent-write.sh"
  }
}
```

### Pattern 5: Memory Across Sessions
Agents lose memory between sessions. Solve this by persisting key data:
```markdown
## Persistent Memory Rules (add to CLAUDE.md)
At the start of every session, read `shared/memory/past_research.json`
for context from previous research sessions.
At the end of every session, append key findings to that file.
```

---

## 10. Real Example: Full Business Opportunity Research

Here is a **complete, working example** you can copy and run immediately.

### The Task
`"Find a profitable SaaS opportunity in the legal tech space for small law firms"`

### What Each Agent Does

**Research Agent** searches for:
- Pain points of small law firms (< 10 lawyers)
- Tools they currently use (Clio, MyCase, PracticePanther)
- Features they wish existed (forums, Reddit r/Lawyertalk, G2 reviews)
- Adjacent successful SaaS products in similar verticals

**Marketing Agent** then analyzes:
- Buyer persona: solo practitioner vs. small firm partner
- CAC channels: LinkedIn ads, bar association newsletters, referral
- Pricing: per-seat SaaS $50–150/month is industry norm
- Differentiation angles vs. Clio

**Financial Agent** models:
- US small law firms: ~450,000 firms with < 10 lawyers
- SOM: target 0.1% = 450 firms × $100/month = $540K ARR Year 1
- CAC estimate: $300–500 via content + LinkedIn
- LTV at 24-month average tenure: $2,400
- LTV:CAC ratio: 4.8x (healthy)
- Capital to reach $1M ARR: ~$400K

**Analysis Agent** synthesizes:
- Ranks this as 8.5/10 overall
- GO decision with specific niche: **AI-powered deposition summarization for small firms**
- 90-day plan: MVP → 10 beta users → Product Hunt launch

### The Final Report (outputs/final_report.md) Will Look Like:
```markdown
# Executive Business Intelligence Report
Date: 2026-03-02
Topic: Legal Tech SaaS for Small Law Firms

## TL;DR
The strongest opportunity is AI-powered deposition/document summarization 
specifically for solo and small firm practitioners. Existing tools (Clio, MyCase) 
focus on case management; nobody owns AI-native document intelligence at the 
sub-10-lawyer segment...

## Opportunity Scorecard
| Opportunity              | Market | Financial | Execution | TOTAL |
|--------------------------|--------|-----------|-----------|-------|
| AI Deposition Summarizer | 9/10   | 8/10      | 8/10      | 25/30 |
| Smart Billing Assistant  | 7/10   | 9/10      | 6/10      | 22/30 |
| Client Portal Upgrade    | 6/10   | 7/10      | 9/10      | 22/30 |

## #1 Recommendation: AI Deposition Summarizer
Decision: GO ✅
...
```

---

## 11. Troubleshooting & Best Practices

### Problem: Agents Don't Wait for Dependencies
**Fix:** Be explicit in agent prompts:
```markdown
## BEFORE YOU START — REQUIRED CHECK:
Run: `grep "Status: COMPLETE" shared/research_output.md`
If it returns nothing, WAIT 20 seconds and check again.
Maximum 10 retries. If still not ready, message the orchestrator.
```

### Problem: Context Window Gets Too Large
**Fix:** Keep agent outputs concise. Add to CLAUDE.md:
```markdown
## Output Length Rules
- research_output.md: max 1,000 words
- marketing_output.md: max 800 words
- financial_output.md: max 600 words
- final_report.md: max 1,500 words
Prioritize signal over comprehensiveness.
```

### Problem: Agents Hallucinate Financial Data
**Fix:** Force citation in Financial Agent prompt:
```markdown
Every financial figure MUST cite a source.
Format: "$X [Source: URL or 'Industry estimate based on Y']"
Never invent numbers. If unknown, write "Data unavailable — recommend primary research."
```

### Problem: Agents Stop Mid-Task
**Fix:** Add explicit completion loops to all agent prompts:
```markdown
## Completion Protocol
After writing your output file, verify it by reading it back.
Confirm "Status: COMPLETE" is the last line.
If not, add it and re-verify.
Only THEN report completion to the orchestrator.
```

### Token Cost Management
- Subagents consume tokens per session
- Use `claude-sonnet-4-6` for research/marketing agents (faster, cheaper)
- Use `claude-opus-4-6` only for the Analysis Agent (needs deepest reasoning)
- Set this in `.claude/settings.json`:
```json
{
  "model": "claude-sonnet-4-6",
  "agentOverrides": {
    "analysis_agent": "claude-opus-4-6"
  }
}
```

### Best Practices Summary

| Practice | Why |
|----------|-----|
| Keep each agent file under 500 words | Reduces context overhead |
| Always define input/output files explicitly | Prevents agents from guessing |
| Use `Status: COMPLETE` markers | Makes dependency checking reliable |
| Write structured output (headers, tables) | Makes it easy for downstream agents to parse |
| Run research with web search enabled | Agents need current market data |
| Test each agent individually first | Before orchestrating the full system |
| Store sessions in `/shared/sessions/` | Enables learning across tasks |

---

## Quick Reference Card

```bash
# Install
npm install -g @anthropic-ai/claude-code

# Start a research job
./scripts/start_research.sh "your market here"

# Monitor
watch -n 5 cat shared/status.json

# Read final report
cat outputs/final_report.md

# Direct Claude Code session  
claude --model claude-opus-4-6

# Enable agent teams
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=true
```

---

## What to Build Next

Once this system works, you can extend it with:

1. **Web dashboard** — Real-time agent status in a React UI
2. **Slack integration** — Agents post updates to channels
3. **Scheduled runs** — Cron job runs weekly market scans
4. **Competitive tracking agent** — Monitors competitor moves
5. **Investor brief agent** — Reformats analysis as pitch deck content
6. **Validation agent** — Cold-calls (simulates) potential customers via research

---

*Built with Claude Code by Anthropic — Version reflects Claude Code capabilities as of March 2026*
*Official docs: https://docs.claude.com/en/docs/claude-code/overview*
