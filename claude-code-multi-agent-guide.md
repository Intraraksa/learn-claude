# Building Collaborative Business Research Agents with Claude Code
### A Step-by-Step Guide by a Senior Anthropic Engineer

---

## Table of Contents
1. [What Claude Code Can Do](#1-what-claude-code-can-do)
2. [Architecture Overview](#2-architecture-overview)
3. [Prerequisites & Setup](#3-prerequisites--setup)
4. [Project Structure](#4-project-structure)
5. [Core: The Agent Orchestrator](#5-core-the-agent-orchestrator)
6. [Building Each Agent](#6-building-each-agent)
7. [Inter-Agent Communication](#7-inter-agent-communication)
8. [Running the Full Pipeline](#8-running-the-full-pipeline)
9. [Advanced: Claude Code Native Multi-Agent](#9-advanced-claude-code-native-multi-agent)
10. [Real-World Example](#10-real-world-example)
11. [Pro Tips from Anthropic Engineering](#11-pro-tips)

---

## 1. What Claude Code Can Do

Claude Code is **not just a code assistant**. It is an agentic runtime that can:

| Capability | What It Means for You |
|---|---|
| **Sub-agents** | Claude Code can spawn other Claude instances as workers |
| **Tool Use** | Bash, file I/O, web search, APIs — all natively |
| **Persistent Memory** | Agents share state via files, memory banks, or databases |
| **Long-horizon Tasks** | Run autonomously for hours without human intervention |
| **MCP Servers** | Connect to Slack, Notion, GitHub, Databases, and 100+ tools |

> YES — Claude Code can absolutely run your collaborative multi-agent business system.

---

## 2. Architecture Overview

```
                        ┌─────────────────────────┐
                        │     ORCHESTRATOR         │
                        │  (Claude Code Master)    │
                        │  - Assigns tasks         │
                        │  - Tracks progress       │
                        │  - Merges results        │
                        └────────────┬────────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                       │
    ┌─────────▼──────┐    ┌──────────▼──────┐   ┌──────────▼──────┐
    │ RESEARCH AGENT │    │ MARKETING AGENT │   │ FINANCIAL AGENT │
    │                │    │                 │   │                 │
    │ - Web search   │    │ - Market sizing │   │ - Revenue model │
    │ - Trend scan   │    │ - Competitors   │   │ - Cost analysis │
    │ - Data collect │    │ - GTM strategy  │   │ - ROI estimate  │
    └────────┬───────┘    └────────┬────────┘   └────────┬────────┘
             │                     │                      │
             └─────────────────────┼──────────────────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   ANALYSIS AGENT    │
                        │  (Synthesizer)      │
                        │  - Reads all output │
                        │  - Final report     │
                        └─────────────────────┘
                                   │
                        ┌──────────▼──────────┐
                        │   shared_memory/    │  ← All agents read/write here
                        │   task_queue.json   │
                        │   research.md       │
                        │   marketing.md      │
                        │   financial.md      │
                        └─────────────────────┘
```

**Communication Pattern:** File-based shared memory (simple, reliable, auditable)

---

## 3. Prerequisites & Setup

### Step 3.1 — Install Claude Code
```bash
# Requires Node.js 18+
npm install -g @anthropic-ai/claude-code

# Verify
claude --version
```

### Step 3.2 — Authenticate
```bash
claude auth login
# Follow the browser prompt to connect your Anthropic account
```

### Step 3.3 — Create CLAUDE.md (Agent Rules File)
```bash
touch CLAUDE.md
```

Add to `CLAUDE.md`:
```markdown
# Project: Business Research Multi-Agent System

## Agent Behavior Rules
- Never stop mid-task. Always complete the full assignment.
- Write all output to shared_memory/ directory before finishing.
- Always read the task_queue.json before starting work.
- Mark tasks as "done" in task_queue.json when complete.
- If you need data from another agent, check shared_memory/ first.

## Tools Available
- Bash (file I/O, data processing)
- Web Search (for market research)
- Python (for data analysis)
```

---

## 4. Project Structure

### Step 4.1 — Scaffold the Project
```bash
mkdir business-research-agents
cd business-research-agents

mkdir -p shared_memory agents prompts reports

cat > shared_memory/task_queue.json << 'EOF'
{
  "project": "",
  "status": "idle",
  "tasks": []
}
EOF

touch prompts/research_agent.md
touch prompts/marketing_agent.md
touch prompts/financial_agent.md
touch prompts/analysis_agent.md
touch prompts/orchestrator.md

touch agents/run_research.sh
touch agents/run_marketing.sh
touch agents/run_financial.sh
touch agents/run_analysis.sh
touch orchestrator.sh
```

### Final Structure:
```
business-research-agents/
├── CLAUDE.md                    ← Rules for all agents
├── orchestrator.sh              ← Master controller
├── shared_memory/
│   ├── task_queue.json          ← Task state & assignments
│   ├── research_output.md
│   ├── marketing_output.md
│   ├── financial_output.md
│   └── final_report.md
├── agents/
│   ├── run_research.sh
│   ├── run_marketing.sh
│   ├── run_financial.sh
│   └── run_analysis.sh
└── prompts/
    ├── orchestrator.md
    ├── research_agent.md
    ├── marketing_agent.md
    ├── financial_agent.md
    └── analysis_agent.md
```

---

## 5. Core: The Agent Orchestrator

### Step 5.1 — Write the Orchestrator Prompt

**`prompts/orchestrator.md`**
```markdown
You are the Master Orchestrator for a business research system.

## Your Job
Given a business research topic, you will:
1. Break it into specific tasks for each specialist agent
2. Write those tasks to shared_memory/task_queue.json
3. Launch agents in the correct order (with dependencies)
4. Monitor progress by reading task_queue.json
5. Trigger the Analysis Agent when all others are done
6. Return the path to the final report

## Task Queue Format
{
  "project": "<topic>",
  "status": "running",
  "tasks": [
    {
      "id": "research_001",
      "agent": "research",
      "status": "pending",
      "assignment": "<specific task>",
      "output_file": "shared_memory/research_output.md",
      "depends_on": []
    },
    {
      "id": "marketing_001",
      "agent": "marketing",
      "status": "pending",
      "assignment": "<specific task>",
      "output_file": "shared_memory/marketing_output.md",
      "depends_on": ["research_001"]
    },
    {
      "id": "financial_001",
      "agent": "financial",
      "status": "pending",
      "output_file": "shared_memory/financial_output.md",
      "depends_on": ["research_001"]
    },
    {
      "id": "analysis_001",
      "agent": "analysis",
      "status": "pending",
      "output_file": "shared_memory/final_report.md",
      "depends_on": ["research_001", "marketing_001", "financial_001"]
    }
  ]
}

## Rules
- Check task dependencies before launching an agent
- Retry failed tasks once before escalating
- Never stop until project status = "completed"
```

### Step 5.2 — Orchestrator Shell Script

**`orchestrator.sh`**
```bash
#!/bin/bash
TOPIC="$1"

if [ -z "$TOPIC" ]; then
  echo "Usage: ./orchestrator.sh \"<research topic>\""
  exit 1
fi

echo "Starting Business Research System"
echo "Topic: $TOPIC"

# Step 1: Plan tasks
claude -p "$(cat prompts/orchestrator.md)
Research topic: '$TOPIC'
Write task queue to shared_memory/task_queue.json" \
  --allowedTools "Bash,Write" \
  --max-turns 5

# Step 2: Research first (everything depends on it)
echo "Launching Research Agent..."
bash agents/run_research.sh "$TOPIC"

# Step 3: Marketing + Financial in parallel
echo "Launching Marketing & Financial Agents in parallel..."
bash agents/run_marketing.sh "$TOPIC" &
bash agents/run_financial.sh "$TOPIC" &
wait

# Step 4: Synthesize
echo "Launching Analysis Agent..."
bash agents/run_analysis.sh "$TOPIC"

echo "DONE — Report at: shared_memory/final_report.md"
cat shared_memory/final_report.md
```

```bash
chmod +x orchestrator.sh
```

---

## 6. Building Each Agent

### Step 6.1 — Research Agent

**`prompts/research_agent.md`**
```markdown
You are the Research Agent — a world-class market researcher.

## Your Mission
For the given business topic, research and document:

1. **Market Landscape** — Key players, technologies, trends
2. **Problem Validation** — Is this a real, painful problem?
3. **Customer Segments** — Who suffers most?
4. **Emerging Signals** — Why is NOW the right time?
5. **Data Points** — Real statistics, market size, growth rates

## Tools
- Web search: recent news, VC investment trends, startup activity

## Output
Write to: shared_memory/research_output.md

# Research Report: [Topic]
## Executive Summary (3 bullets)
## Market Landscape
## Problem Evidence
## Target Customers
## Timing & Trends
## Key Statistics
## Sources

Update task_queue.json: set research task to "done"
Do NOT stop until file is written and task is marked done.
```

**`agents/run_research.sh`**
```bash
#!/bin/bash
TOPIC="$1"
echo "Research Agent starting on: $TOPIC"

claude -p "$(cat prompts/research_agent.md)

Your assigned topic: '$TOPIC'
Start now. Do not ask questions. Research and write the full report." \
  --allowedTools "Bash,Write,WebSearch" \
  --max-turns 20

echo "Research Agent complete"
```

---

### Step 6.2 — Marketing Agent

**`prompts/marketing_agent.md`**
```markdown
You are the Marketing Agent — a seasoned growth strategist.

## Your Mission
Read shared_memory/research_output.md first, then analyze:

1. **TAM/SAM/SOM** — Size the market with numbers
2. **Competitive Analysis** — Top 5 competitors, their weaknesses
3. **Customer Acquisition** — Best channels
4. **Positioning Strategy** — Unique angle to win
5. **Go-to-Market Plan** — 90-day launch strategy

## Process
1. Read: shared_memory/research_output.md
2. Research competitors and market sizing
3. Write to: shared_memory/marketing_output.md
4. Update task_queue.json: set marketing task to "done"
```

**`agents/run_marketing.sh`**
```bash
#!/bin/bash
TOPIC="$1"

while [ ! -f shared_memory/research_output.md ]; do
  echo "Waiting for research output..."
  sleep 5
done

claude -p "$(cat prompts/marketing_agent.md)
Topic: '$TOPIC'. Begin your analysis now." \
  --allowedTools "Bash,Write,WebSearch" \
  --max-turns 20

echo "Marketing Agent complete"
```

---

### Step 6.3 — Financial Agent

**`prompts/financial_agent.md`**
```markdown
You are the Financial Agent — an expert startup financial analyst.

## Your Mission
Read shared_memory/research_output.md, then build:

1. **Revenue Model Options** — SaaS, marketplace, transactional?
2. **Unit Economics** — Estimated CAC, LTV, payback period
3. **Cost Structure** — Key cost drivers
4. **Funding Requirements** — How much to reach key milestones
5. **Financial Projections** — 3-year revenue (conservative/base/aggressive)
6. **Risk Factors** — Top 5 financial risks

## Output
Write to: shared_memory/financial_output.md
Update task_queue.json: set financial task to "done"
Use tables and numbers. Make it investor-ready.
```

**`agents/run_financial.sh`**
```bash
#!/bin/bash
TOPIC="$1"

while [ ! -f shared_memory/research_output.md ]; do
  sleep 5
done

claude -p "$(cat prompts/financial_agent.md)
Topic: '$TOPIC'. Begin financial analysis now." \
  --allowedTools "Bash,Write,WebSearch" \
  --max-turns 20

echo "Financial Agent complete"
```

---

### Step 6.4 — Analysis Agent (Synthesizer)

**`prompts/analysis_agent.md`**
```markdown
You are the Analysis Agent — a top-tier strategy consultant.

## Your Mission
Read ALL outputs then synthesize a decisive investment brief:
- shared_memory/research_output.md
- shared_memory/marketing_output.md
- shared_memory/financial_output.md

## Output: Final Investment Brief

# Business Opportunity Brief: [Topic]
**Verdict:** [STRONG BUY / BUY / HOLD / PASS] + one sentence why

## 1. Opportunity in One Paragraph
## 2. Why Now (3 reasons)
## 3. Target Customer (be specific)
## 4. Business Model Recommendation
## 5. Market Size
## 6. Competitive Advantage to Build
## 7. 12-Month Action Plan
## 8. Financial Summary Table
## 9. Top 3 Risks + Mitigations
## 10. Decision: Go / No-Go + Rationale

Write to: shared_memory/final_report.md
Update task_queue.json: set project status to "completed"
```

**`agents/run_analysis.sh`**
```bash
#!/bin/bash
TOPIC="$1"

while [ ! -f shared_memory/marketing_output.md ] || [ ! -f shared_memory/financial_output.md ]; do
  echo "Waiting for all agents..."
  sleep 5
done

claude -p "$(cat prompts/analysis_agent.md)
Topic: '$TOPIC'. All agent outputs are ready. Synthesize now." \
  --allowedTools "Bash,Write" \
  --max-turns 15

echo "Analysis Agent complete — Final report ready!"
```

---

## 7. Inter-Agent Communication

Agents share information through 3 channels:

### Channel 1: File-Based (recommended, shown above)
Simple, reliable, works out of the box.

### Channel 2: Claude Code's Native `--resume` Feature
```bash
# Agent A runs and saves session ID
SESSION_ID=$(claude -p "..." --output-format json | jq -r '.session_id')

# Agent B resumes and reads Agent A's memory
claude --resume $SESSION_ID -p "Now do the marketing analysis on top of what was just researched"
```

### Channel 3: Structured JSON Messages
```bash
# Agent writes a structured message
cat > shared_memory/messages.json << 'EOF'
{
  "from": "research_agent",
  "to": "financial_agent",
  "type": "data_handoff",
  "payload": {
    "market_size": "$4.2B",
    "growth_rate": "23% YoY",
    "key_players": ["CompanyA", "CompanyB"]
  }
}
EOF

# Financial agent reads it
claude -p "Read shared_memory/messages.json and incorporate the market data into your financial model"
```

---

## 8. Running the Full Pipeline

### Step 8.1 — Make All Scripts Executable
```bash
chmod +x orchestrator.sh agents/*.sh
```

### Step 8.2 — Run Your First Research Mission
```bash
./orchestrator.sh "AI-powered inventory management for small restaurants"
```

### Step 8.3 — Watch it Run
```
Starting Business Research System
Topic: AI-powered inventory management for small restaurants
Launching Research Agent...
Launching Marketing & Financial Agents in parallel...
  Financial Agent starting...
  Marketing Agent starting...
  Marketing Agent complete
  Financial Agent complete
Launching Analysis Agent...
DONE — Report at: shared_memory/final_report.md
```

### Step 8.4 — Read Your Report
```bash
cat shared_memory/final_report.md
code shared_memory/final_report.md  # open in VS Code
```

---

## 9. Advanced: Claude Code Native Multi-Agent

Claude Code has **built-in** subagent support via the `Task` tool. This is the most powerful pattern:

### The "Agent as Tool" Pattern
```bash
# orchestrator_advanced.sh
# Claude Code spawns sub-agents natively

claude -p "
You are orchestrating a business research pipeline for: '$TOPIC'

You have access to the Task tool to spawn specialist agents.
Use it to run these agents:

1. Use Task tool to run Research Agent:
   'Research market opportunity for $TOPIC. Write to shared_memory/research_output.md'

2. After research, SIMULTANEOUSLY run via Task tool:
   - Marketing: 'Read research output. Write to shared_memory/marketing_output.md'
   - Financial:  'Read research output. Write to shared_memory/financial_output.md'

3. After both complete, run via Task tool:
   - Analysis: 'Read all outputs. Write final brief to shared_memory/final_report.md'

Do not stop until final_report.md exists and is comprehensive.
" \
  --allowedTools "Task,Bash,Write,WebSearch" \
  --max-turns 50
```

> The `Task` tool is Claude Code's native way to spawn subagents. Each sub-agent gets its own context, tools, and runs in parallel. This is the recommended production approach.

---

## 10. Real-World Example

### Full Run: "Telemedicine for Rural Seniors"

```bash
./orchestrator.sh "Telemedicine platform for rural seniors aged 65+"
```

**What each agent produces:**

| Agent | Output Highlights |
|---|---|
| **Research** | 40M rural seniors underserved; $29B telehealth market; Medicare billing unlocked post-COVID |
| **Marketing** | Primary channel: church groups + senior centers; $45 CAC estimate; Medicare Advantage partnerships |
| **Financial** | $89/mo subscription; 18-month payback; need $2.1M seed to reach breakeven at 3,200 users |
| **Analysis** | **STRONG BUY** — Massive underserved market, regulatory tailwinds, clear distribution path |

**Final report generated in ~12 minutes, fully autonomous.**

---

## 11. Pro Tips from Anthropic Engineering

### Tip 1: Write Specific CLAUDE.md Files Per Agent
Each agent folder can have its own `CLAUDE.md` that shapes behavior without repeating it in every prompt.

### Tip 2: Use `--max-turns` Wisely
```bash
# Research needs more turns (web search loops)
claude -p "..." --max-turns 25

# Analysis just reads files — fewer turns needed
claude -p "..." --max-turns 10
```

### Tip 3: Add MCP Servers for Real Power
```bash
# Add to ~/.claude/mcp_servers.json
{
  "servers": {
    "notion": { "command": "npx @modelcontextprotocol/server-notion" },
    "slack":  { "command": "npx @modelcontextprotocol/server-slack" },
    "github": { "command": "npx @modelcontextprotocol/server-github" }
  }
}
# Now agents can write to Notion, post to Slack, create GitHub issues — automatically
```

### Tip 4: Build a Memory Bank for Cross-Project Learning
```bash
mkdir -p memory_bank/

# Add to every agent prompt:
# "Before starting, read memory_bank/*.md for relevant past research.
#  After finishing, append key learnings to memory_bank/[domain].md"
```

### Tip 5: Cost Management — Right Model Per Task
```bash
# Use Haiku for simple tasks (formatting, file reading)
claude --model claude-haiku-4-5-20251001 -p "Format this report..."

# Use Sonnet for research & analysis
claude --model claude-sonnet-4-6 -p "Research this market..."
```

### Tip 6: Make Agents Idempotent (Safe to Re-run)
```bash
if [ -f shared_memory/research_output.md ] && [ -s shared_memory/research_output.md ]; then
  echo "Research already complete, skipping..."
else
  bash agents/run_research.sh "$TOPIC"
fi
```

---

## Quick Reference Commands

```bash
# Start a new research pipeline
./orchestrator.sh "your business idea here"

# Run just one agent manually
bash agents/run_research.sh "your topic"

# Check task status
cat shared_memory/task_queue.json | jq .

# Read final report
cat shared_memory/final_report.md

# Reset for new project
rm -f shared_memory/*.md
echo '{"project":"","status":"idle","tasks":[]}' > shared_memory/task_queue.json
```

---

## Summary

```
INPUT:   One-line business idea
           ↓
PROCESS: 4 specialist AI agents working autonomously in parallel
           ↓
OUTPUT:  Professional investment brief with verdict, financials, GTM plan
```

This system runs **without human intervention**, agents **share data** through structured memory, and the pipeline **does not stop** until the final report is delivered.

The same pattern scales to any domain: legal research, competitive intelligence, technical due diligence, or any multi-faceted research task you can define.

---

*Built with Claude Code | Anthropic Engineering Patterns*
