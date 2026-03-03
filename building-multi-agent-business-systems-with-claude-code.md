# Building Multi-Agent Business Systems with Claude Code

## A Complete Step-by-Step Guide for Research-to-Decision Agent Teams

> **Author's Note**: This guide teaches you how to build collaborative AI agent teams using Claude Code — where multiple specialized agents work autonomously, share results, and coordinate until the job is done. This is the same architecture used internally at Anthropic-scale projects.

---

## Table of Contents

1. [Understanding the Architecture](#1-understanding-the-architecture)
2. [Prerequisites & Setup](#2-prerequisites--setup)
3. [The Three Layers of Multi-Agent in Claude Code](#3-the-three-layers-of-multi-agent-in-claude-code)
4. [Step 1: Design Your Agent Team](#step-1-design-your-agent-team)
5. [Step 2: Create Custom Subagents](#step-2-create-custom-subagents)
6. [Step 3: Build the Orchestration Layer](#step-3-build-the-orchestration-layer)
7. [Step 4: Enable Agent Teams (Experimental)](#step-4-enable-agent-teams-experimental)
8. [Step 5: Wire Inter-Agent Communication](#step-5-wire-inter-agent-communication)
9. [Step 6: Build a Real Business Research System](#step-6-build-a-real-business-research-system)
10. [Step 7: Advanced Patterns](#step-7-advanced-patterns)
11. [Step 8: Production Deployment with Agent SDK](#step-8-production-deployment-with-agent-sdk)
12. [Troubleshooting & Best Practices](#troubleshooting--best-practices)

---

## 1. Understanding the Architecture

### What You're Building

```
┌─────────────────────────────────────────────────────────┐
│                    YOU (Team Lead)                       │
│              Give the mission, monitor progress          │
└──────────────────────┬──────────────────────────────────┘
                       │
          ┌────────────▼────────────┐
          │    ORCHESTRATOR AGENT   │
          │  (Supervisor / Leader)  │
          │  Decomposes tasks,      │
          │  routes work, collects  │
          │  results                │
          └────┬────┬────┬────┬────┘
               │    │    │    │
    ┌──────────▼┐ ┌─▼────┐ ┌─▼──────┐ ┌▼──────────┐
    │ RESEARCH  │ │MARKET│ │FINANCE │ │ ANALYSIS  │
    │  AGENT    │ │AGENT │ │ AGENT  │ │  AGENT    │
    │           │ │      │ │        │ │           │
    │ Web search│ │Trends│ │Revenue │ │Synthesize │
    │ Competitor│ │Oppty │ │Costs   │ │Cross-ref  │
    │ analysis  │ │Gaps  │ │ROI     │ │Recommend  │
    └─────┬─────┘ └──┬───┘ └───┬────┘ └─────┬─────┘
          │          │         │             │
          └──────────┴────┬────┴─────────────┘
                          │
                ┌─────────▼─────────┐
                │   SHARED STATE    │
                │  (Task List +     │
                │   Message Inbox)  │
                └───────────────────┘
```

### Why Claude Code for This?

Claude Code has **three built-in mechanisms** for multi-agent work, and you don't need external frameworks:

| Mechanism | What It Does | Best For |
|-----------|-------------|----------|
| **Subagents** (Task tool) | Spawn child agents within a session | Parallel tasks, isolated context |
| **Agent Teams** (TeammateTool) | Multiple Claude Code sessions collaborating | True collaboration, debate, shared tasks |
| **Agent SDK** (Programmatic) | Code-level orchestration via Python/TypeScript | Production systems, custom workflows |

---

## 2. Prerequisites & Setup

### Install Claude Code

```bash
# Install Claude Code globally
npm install -g @anthropic-ai/claude-code

# Verify installation
claude --version

# You need Node.js 18+ installed
node --version
```

### Required Accounts & Keys

```bash
# Set your Anthropic API key
export ANTHROPIC_API_KEY="sk-ant-..."

# Optional: for web search capabilities in agents
# Claude Code has built-in web search via the WebSearch tool
```

### Enable Experimental Features

```bash
# Enable Agent Teams (experimental but powerful)
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# Or add to your settings.json:
# ~/.claude/settings.json
```

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### Project Setup

```bash
# Create your project directory
mkdir business-agent-team
cd business-agent-team

# Initialize
git init
mkdir -p .claude/agents
mkdir -p docs/research
mkdir -p docs/reports
mkdir -p data

# Create your CLAUDE.md (project instructions)
touch CLAUDE.md
```

---

## 3. The Three Layers of Multi-Agent in Claude Code

Before building, understand what each layer offers:

### Layer 1: Subagents (Simplest — Start Here)

Subagents are **child processes** spawned by the main Claude Code session. They run in their own context window, do their job, and return results. Think of them as "send someone to do an errand."

```
Main Session → spawns → Subagent A (background)
            → spawns → Subagent B (background)
            → spawns → Subagent C (background)
            → collects results from all three
```

**Limitation**: Subagents report back to the parent only. They can't talk to each other directly.

### Layer 2: Agent Teams (True Collaboration)

Agent Teams are **separate Claude Code sessions** that can message each other, share a task list, and coordinate. Think of them as "a team in the same room."

```
Leader ←→ Teammate A
  ↕          ↕
Teammate B ←→ Teammate C
  (all share a task list and inbox system)
```

**This is what you want for your business agent scenario.**

### Layer 3: Agent SDK (Production Code)

The Agent SDK lets you orchestrate Claude programmatically from Python or TypeScript. You define agents in code, wire them together, and run them as a production application.

---

## Step 1: Design Your Agent Team

### Define Roles and Responsibilities

Before writing any code, design your team on paper. Here's a template:

```markdown
## Team: New Product Opportunity Research

### Mission
Evaluate whether [PRODUCT IDEA] is a viable business opportunity,
producing a comprehensive go/no-go recommendation with supporting data.

### Agent Roles

1. **Research Agent** (researcher)
   - Job: Market research, competitor analysis, trend identification
   - Tools: WebSearch, WebFetch, Read, Write
   - Outputs: Market size data, competitor landscape, trend report

2. **Marketing Agent** (marketer)
   - Job: Identify target segments, channels, positioning
   - Tools: WebSearch, WebFetch, Read, Write
   - Outputs: Customer personas, channel strategy, positioning brief

3. **Financial Agent** (finance)
   - Job: Revenue modeling, cost estimation, ROI analysis
   - Tools: Read, Write, Bash (for calculations)
   - Outputs: Financial projections, unit economics, break-even

4. **Analysis Agent** (analyst)
   - Job: Synthesize all findings, identify risks, make recommendation
   - Tools: Read, Write
   - Depends on: Researcher, Marketer, Finance
   - Outputs: Final recommendation report

### Communication Flow
researcher → shares findings → analyst
marketer   → shares findings → analyst
finance    → shares findings → analyst
analyst    → produces final  → team lead (you)
```

### Create the CLAUDE.md

This is the "brain" of your project. Every agent reads this on startup.

Create the file `.claude/CLAUDE.md` in your project root:

```markdown
# Business Research Agent Team

## Project Purpose
This project uses a multi-agent team to evaluate business opportunities.
Each agent has a specialized role and communicates through shared files
and the team messaging system.

## Agent Coordination Rules
1. Each agent writes findings to `docs/research/[agent-name]-findings.md`
2. Agents should be thorough but concise — aim for actionable insights
3. The analyst agent waits for all other agents before synthesizing
4. All financial figures should include sources and assumptions
5. Final deliverable goes to `docs/reports/final-recommendation.md`

## Output Standards
- Use markdown for all reports
- Include data sources with URLs
- Quantify claims where possible
- Flag confidence levels (High/Medium/Low) for each finding

## Task Dependencies
- Research, Marketing, and Finance agents can work IN PARALLEL
- Analysis agent starts AFTER the above three complete
- This is a Fan-Out → Gather pattern
```

---

## Step 2: Create Custom Subagents

### Agent Definition Files

Each agent is a markdown file in `.claude/agents/`. Here's how to create each one:

### 2a. Research Agent

Create `.claude/agents/researcher.md`:

```markdown
---
name: researcher
description: >
  Market research specialist. Use this agent when you need to research
  market opportunities, analyze competitors, identify trends, or gather
  industry data. Automatically invoked for research-related tasks.
tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch
model: sonnet
---

You are a **Senior Market Research Analyst** on a business evaluation team.

## Your Mission
Conduct thorough market research and deliver actionable intelligence.

## Research Protocol

### Phase 1: Market Landscape
1. Search for the target market size and growth rate
2. Identify key players and their market share
3. Map the value chain and distribution channels
4. Note regulatory considerations

### Phase 2: Competitor Analysis
1. Identify top 5-10 competitors (direct and indirect)
2. For each: pricing, positioning, strengths, weaknesses
3. Identify gaps and underserved segments
4. Assess barriers to entry

### Phase 3: Trend Analysis
1. Search for emerging trends in the space
2. Technology shifts that could impact the market
3. Consumer behavior changes
4. Macro-economic factors

## Output Format
Write your complete findings to `docs/research/researcher-findings.md` with:
- Executive Summary (3-5 bullets)
- Market Size & Growth (with sources)
- Competitive Landscape (table format)
- Key Trends (ranked by impact)
- Opportunities Identified
- Risks & Concerns
- Confidence Assessment

## Communication
When finished, write a summary of your key findings.
Flag any critical discoveries that other agents should know about.
```

### 2b. Marketing Agent

Create `.claude/agents/marketer.md`:

```markdown
---
name: marketer
description: >
  Marketing strategy specialist. Use this agent for identifying target
  segments, go-to-market strategy, positioning, channel analysis, and
  customer persona development. Invoked for marketing-related tasks.
tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch
model: sonnet
---

You are a **Chief Marketing Strategist** on a business evaluation team.

## Your Mission
Develop a go-to-market assessment for the proposed product/opportunity.

## Analysis Protocol

### Phase 1: Customer Discovery
1. Define 2-3 primary customer personas
2. For each persona: demographics, pain points, buying behavior
3. Estimate addressable audience size
4. Map the customer journey

### Phase 2: Channel Strategy
1. Identify top acquisition channels (paid, organic, partnerships)
2. Estimate customer acquisition cost (CAC) per channel
3. Benchmark against industry averages
4. Assess channel scalability

### Phase 3: Positioning & Messaging
1. Define unique value proposition
2. Positioning relative to competitors
3. Key messaging pillars
4. Brand differentiation strategy

### Phase 4: Go-to-Market Timeline
1. Launch strategy (soft launch vs. full launch)
2. First 90 days plan
3. Key milestones and KPIs

## Output Format
Write your complete findings to `docs/research/marketer-findings.md` with:
- Executive Summary
- Customer Personas (detailed)
- Channel Strategy Matrix
- Positioning Statement
- Estimated CAC and LTV
- Go-to-Market Roadmap
- Risks & Assumptions

## Communication
After finishing, share your CAC and LTV estimates — the finance agent needs these.
```

### 2c. Financial Agent

Create `.claude/agents/finance.md`:

```markdown
---
name: finance
description: >
  Financial analysis specialist. Use this agent for revenue modeling,
  cost estimation, unit economics, ROI analysis, break-even calculations,
  and financial projections. Invoked for finance-related tasks.
tools: Read, Write, Edit, Bash, Glob, Grep
model: sonnet
---

You are a **Senior Financial Analyst** on a business evaluation team.

## Your Mission
Build financial projections and assess the economic viability of the opportunity.

## Analysis Protocol

### Phase 1: Revenue Modeling
1. Define pricing strategy options (freemium, subscription, one-time, etc.)
2. Estimate monthly recurring revenue (MRR) scenarios
3. Model 3 scenarios: Conservative, Base, Optimistic
4. Project 12-month and 36-month revenue

### Phase 2: Cost Structure
1. Fixed costs (team, infrastructure, tools)
2. Variable costs (hosting, support, COGS)
3. Customer acquisition costs
4. One-time setup costs

### Phase 3: Unit Economics
1. Calculate LTV:CAC ratio
2. Gross margin analysis
3. Contribution margin per customer
4. Payback period

### Phase 4: Financial Assessment
1. Break-even analysis (when and at what volume)
2. Cash flow projection (first 12 months)
3. Funding requirements
4. ROI for a 3-year horizon

## Computational Approach
Use bash for calculations when needed:
```bash
python3 -c "
# Example: Break-even calculation
fixed_costs = 50000
price = 29.99
variable_cost_per_unit = 5
break_even = fixed_costs / (price - variable_cost_per_unit)
print(f'Break-even units: {break_even:.0f}')
"
```

## Output Format
Write findings to `docs/research/finance-findings.md` with:
- Executive Summary
- Revenue Projections (3 scenarios, tabular)
- Cost Structure Breakdown
- Unit Economics Summary
- Break-even Analysis
- Cash Flow Projection
- Investment Recommendation
- Key Assumptions & Sensitivities

## Communication
Share your break-even point and ROI estimates with the team.
Flag if the economics are fundamentally challenging.
```

### 2d. Analysis Agent

Create `.claude/agents/analyst.md`:

```markdown
---
name: analyst
description: >
  Strategic synthesis analyst. Use this agent to synthesize findings from
  multiple research streams, identify cross-cutting themes, assess risks,
  and produce final recommendations. Invoked after research is complete.
tools: Read, Write, Edit, Glob, Grep
model: sonnet
---

You are a **Chief Strategy Officer** synthesizing a business evaluation.

## Your Mission
Read all agent findings, synthesize insights, and produce a go/no-go recommendation.

## Synthesis Protocol

### Phase 1: Gather All Findings
1. Read `docs/research/researcher-findings.md`
2. Read `docs/research/marketer-findings.md`
3. Read `docs/research/finance-findings.md`
4. If any file is missing, note it as a gap

### Phase 2: Cross-Reference Analysis
1. Do market size estimates align with revenue projections?
2. Are CAC estimates from marketing consistent with finance models?
3. Are there contradictions between agents' findings?
4. What assumptions are shared vs. different?

### Phase 3: SWOT Synthesis
1. Strengths: What advantages does this opportunity have?
2. Weaknesses: What internal challenges exist?
3. Opportunities: What external factors favor this?
4. Threats: What could derail this?

### Phase 4: Risk Assessment
1. Rank top 5 risks by likelihood × impact
2. Identify mitigation strategies for each
3. Flag any deal-breakers

### Phase 5: Final Recommendation
1. Go / No-Go / Conditional Go decision
2. If Go: key success factors and next steps
3. If No-Go: what would need to change
4. Confidence level in the recommendation

## Output Format
Write the final report to `docs/reports/final-recommendation.md` with:
- Executive Summary (1 page max)
- Synthesis of Key Findings
- SWOT Analysis
- Risk Matrix
- Financial Viability Assessment
- Strategic Recommendation
- Next Steps (if Go)
- Appendix: Summary of Each Agent's Findings
```

---

## Step 3: Build the Orchestration Layer

### Method A: Using Subagents (Task Tool) — Simpler

Open Claude Code in your project directory and give it this prompt:

```
I need you to orchestrate a business research project using subagents.

The product to evaluate: [YOUR PRODUCT IDEA HERE]

Execute this workflow:

PHASE 1 — PARALLEL RESEARCH (run all 3 simultaneously):
1. Use the "researcher" agent to conduct market research on this opportunity
2. Use the "marketer" agent to develop go-to-market analysis
3. Use the "finance" agent to build financial projections

Run all three as background tasks. Each should write their findings to
docs/research/[agent-name]-findings.md

PHASE 2 — SYNTHESIS (after Phase 1 completes):
4. Use the "analyst" agent to read all findings and produce a final
   recommendation at docs/reports/final-recommendation.md

Share the product context with each agent:
Product: [describe your product]
Target Market: [describe target]
Budget: [your budget range]
Timeline: [your timeline]
```

Claude Code will then:
1. Spawn three subagents in the background
2. Each agent works independently with its own context
3. Wait for all three to complete
4. Spawn the analyst agent to synthesize
5. Present you with the final report

### Method B: Using a CLAUDE.md Orchestration Prompt

Add this to your `CLAUDE.md`:

```markdown
## Orchestration Instructions

When asked to "evaluate [product/opportunity]", automatically:

1. **Fan-Out Phase**: Spawn 3 parallel background tasks:
   - Task 1: `researcher` agent — market research
   - Task 2: `marketer` agent — GTM strategy
   - Task 3: `finance` agent — financial modeling

2. **Gather Phase**: After all 3 complete, spawn:
   - Task 4: `analyst` agent — synthesis and recommendation

3. **Deliver**: Present the final recommendation summary and
   link to the full report.

Always pass the full product context to each agent.
Always run Phase 1 tasks in background (`run_in_background: true`).
Wait for all Phase 1 tasks before starting Phase 2.
```

Now you can simply type:

```
Evaluate the opportunity for an AI-powered meal planning SaaS
targeting busy professionals, budget $100k, 6-month runway.
```

And Claude Code handles the rest.

---

## Step 4: Enable Agent Teams (Experimental)

Agent Teams take it further — agents can **talk to each other**, not just report back to you.

### Enable the Feature

```bash
# In your terminal before starting Claude Code
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# Start Claude Code
claude
```

### Launch a Team

In Claude Code, give this prompt:

```
Create an agent team called "product-eval" to evaluate a new business
opportunity. Spawn these teammates:

1. "researcher" — Market research specialist. Research the market for
   [YOUR PRODUCT]. Find market size, competitors, and trends. Share your
   key findings with the finance and analyst teammates when done.

2. "marketer" — Marketing strategist. Develop GTM strategy for
   [YOUR PRODUCT]. Identify customer personas, channels, and CAC estimates.
   Share CAC/LTV estimates with the finance teammate.

3. "finance" — Financial analyst. Build revenue projections and unit
   economics for [YOUR PRODUCT]. Wait for the marketer's CAC/LTV
   estimates before finalizing. Share your break-even analysis with
   the analyst.

4. "analyst" — Strategy synthesizer. Wait for findings from all three
   teammates. Read everything they've produced. Create a final
   go/no-go recommendation.

Create tasks with proper dependencies:
- research, marketing, finance tasks can run in parallel
- analysis task is blocked by the other three

Have them coordinate through the team messaging system.
```

### How Agent Teams Communicate

Under the hood, teammates use an inbox system:

```
~/.claude/teams/product-eval/
├── tasks.json              # Shared task list
├── inboxes/
│   ├── team-lead.json      # Your inbox
│   ├── researcher.json     # Researcher's inbox
│   ├── marketer.json       # Marketer's inbox
│   ├── finance.json        # Finance's inbox
│   └── analyst.json        # Analyst's inbox
```

Teammates send messages like:

```
# Researcher → Analyst
Teammate({
  operation: "write",
  target_agent_id: "analyst",
  value: "Market research complete. Key finding: market is $4.2B
         and growing 23% YoY. Full report in docs/research/researcher-findings.md"
})
```

### Task Dependencies

The task system supports blocking relationships:

```
Task #1: Market Research       → status: in_progress, owner: researcher
Task #2: GTM Strategy          → status: in_progress, owner: marketer
Task #3: Financial Projections → status: pending, blocked_by: [#2]
Task #4: Final Analysis        → status: pending, blocked_by: [#1, #2, #3]
```

When Task #2 completes, Task #3 automatically unblocks. When all three complete, Task #4 begins.

---

## Step 5: Wire Inter-Agent Communication

### Pattern 1: File-Based Communication (Simple, Reliable)

The simplest way for agents to share results:

```
docs/research/
├── researcher-findings.md    # Researcher writes here
├── marketer-findings.md      # Marketer writes here
├── finance-findings.md       # Finance writes here
└── shared-assumptions.md     # Any agent can update shared assumptions

docs/reports/
└── final-recommendation.md   # Analyst writes the final output
```

Each agent's instructions tell it where to write and where to read from.

### Pattern 2: Structured Handoff Protocol

Define a handoff format in your `CLAUDE.md`:

```markdown
## Agent Handoff Protocol

When an agent completes work, it writes a handoff file:

File: `data/handoffs/[from-agent]-to-[to-agent].json`

Format:
{
  "from": "researcher",
  "to": "analyst",
  "timestamp": "2026-03-02T10:30:00Z",
  "status": "complete",
  "key_findings": [
    "Market size is $4.2B",
    "Top competitor has 35% market share",
    "Fastest growing segment is SMB"
  ],
  "full_report_path": "docs/research/researcher-findings.md",
  "confidence": "high",
  "flags": ["competitor recently raised $50M"]
}
```

### Pattern 3: Debate Pattern (Agent Teams Only)

For decisions where you want agents to challenge each other:

```
Prompt to Claude Code:

After all agents complete their individual research, initiate a
debate phase:

1. Broadcast to all teammates: "Present your top 3 concerns
   about this opportunity. Challenge each other's assumptions."

2. Each agent reads others' findings and posts rebuttals or
   agreements to the team channel.

3. After 2 rounds of debate, the analyst synthesizes the
   consensus and dissenting views into the final report.
```

This mirrors how a real executive team would evaluate an opportunity.

---

## Step 6: Build a Real Business Research System

### Complete Working Example

Here's a fully self-contained example you can run today:

#### 1. Set Up the Project

```bash
mkdir ai-meal-planner-eval
cd ai-meal-planner-eval
git init
mkdir -p .claude/agents docs/research docs/reports data/handoffs
```

#### 2. Create CLAUDE.md

```bash
cat > CLAUDE.md << 'EOF'
# AI Meal Planner Business Evaluation

## Mission
Evaluate the viability of an AI-powered meal planning SaaS for busy
professionals. Budget: $100k. Timeline: 6 months to MVP launch.

## Agent Team
- researcher: Market research and competitor analysis
- marketer: GTM strategy and customer acquisition
- finance: Financial modeling and projections
- analyst: Synthesis and final recommendation

## Workflow
1. Research, Marketing, and Finance work in PARALLEL
2. Each writes findings to docs/research/[name]-findings.md
3. Analyst reads all findings, writes docs/reports/final-recommendation.md
4. Use background tasks for Phase 1, foreground for Phase 2

## Standards
- All dollar figures in USD
- Include sources for market data
- Flag assumptions explicitly
- Rate confidence: High / Medium / Low
EOF
```

#### 3. Create All Four Agent Files

(Use the agent definitions from Step 2 above — copy them into `.claude/agents/`)

#### 4. Run It

```bash
# Start Claude Code with agent teams enabled
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude
```

Then type:

```
Run the full business evaluation for an AI-powered meal planning SaaS.
Target market: busy professionals aged 25-45 in the US.
Pricing model: $14.99/month subscription.
Budget: $100k seed funding.
Timeline: 6 months to MVP.

Use all four agents. Run researcher, marketer, and finance in parallel
as background tasks. After they complete, run the analyst to synthesize.
```

#### 5. What Happens Next

Claude Code will:
1. Spawn `researcher` in background → searches the web, analyzes competitors
2. Spawn `marketer` in background → identifies channels, estimates CAC
3. Spawn `finance` in background → builds revenue model, calculates break-even
4. Wait for all three to complete
5. Spawn `analyst` in foreground → reads all three reports, synthesizes
6. Present you with the final go/no-go recommendation

**Total time**: ~5-15 minutes depending on depth of web research.

---

## Step 7: Advanced Patterns

### Pattern A: Iterative Refinement Loop

Agents don't just run once — they can loop:

```markdown
## In CLAUDE.md

### Iteration Protocol
After the analyst produces the first draft:
1. Share the draft with all agents
2. Each agent reviews for gaps in their domain
3. Each agent produces a "revision memo" with corrections
4. Analyst incorporates revisions into final version
5. Maximum 2 revision cycles
```

### Pattern B: Adaptive Agent Spawning

Let the orchestrator decide which agents are needed:

```
Evaluate this opportunity: [description]

First, analyze the opportunity and decide which specialist agents
would be most valuable. You don't have to use all four — pick the
ones most relevant. For example:
- If it's a B2B enterprise play, prioritize finance and research
- If it's a consumer product, prioritize marketing and research
- If it's a crowded market, add a "competitive intelligence" specialist

Explain your team composition choice, then execute.
```

### Pattern C: Multi-Project Portfolio Evaluation

Scale to evaluating multiple opportunities simultaneously:

```
I have 3 product ideas to evaluate:
1. AI meal planner
2. AI-powered resume builder
3. AI fitness coach

For EACH idea:
- Spawn a researcher, marketer, and finance agent (9 agents total)
- Run all 9 in parallel
- After each trio finishes, spawn an analyst for that idea
- Finally, spawn a "portfolio analyst" to rank all 3 and recommend
  which to pursue first

Organize outputs in:
docs/ideas/meal-planner/
docs/ideas/resume-builder/
docs/ideas/fitness-coach/
docs/reports/portfolio-recommendation.md
```

### Pattern D: Human-in-the-Loop Checkpoints

Add approval gates between phases:

```markdown
## In CLAUDE.md

### Checkpoint Protocol
After Phase 1 (parallel research) completes:
1. Present a summary of all findings to the user
2. Ask: "Should I proceed to synthesis, or do you want to
   redirect any agent's research?"
3. Wait for user approval before starting Phase 2
4. If user requests changes, re-run only the affected agent
```

---

## Step 8: Production Deployment with Agent SDK

For production systems (APIs, scheduled runs, integrations), use the **Claude Agent SDK**.

### Install the SDK

```bash
npm install @anthropic-ai/claude-code
```

### Basic Programmatic Orchestration

```typescript
// business-eval-agent.ts
import Anthropic from "@anthropic-ai/claude-code";

const client = new Anthropic();

// Define the agents
const agents = [
  {
    name: "researcher",
    description: "Market research specialist",
    tools: ["Read", "Write", "WebSearch", "WebFetch", "Bash"],
  },
  {
    name: "marketer",
    description: "Marketing strategy specialist",
    tools: ["Read", "Write", "WebSearch", "WebFetch", "Bash"],
  },
  {
    name: "finance",
    description: "Financial analysis specialist",
    tools: ["Read", "Write", "Bash"],
  },
];

async function evaluateOpportunity(productDescription: string) {
  // Phase 1: Parallel research
  const researchPromises = agents.map((agent) =>
    client.messages.create({
      model: "claude-sonnet-4-5-20250929",
      max_tokens: 8096,
      system: `You are a ${agent.description}. ${getAgentPrompt(agent.name)}`,
      messages: [
        {
          role: "user",
          content: `Evaluate this opportunity: ${productDescription}`,
        },
      ],
    })
  );

  const results = await Promise.all(researchPromises);

  // Phase 2: Synthesis
  const synthesisPrompt = results
    .map(
      (r, i) =>
        `## ${agents[i].name} Findings:\n${r.content[0].type === "text" ? r.content[0].text : ""}`
    )
    .join("\n\n");

  const finalReport = await client.messages.create({
    model: "claude-sonnet-4-5-20250929",
    max_tokens: 8096,
    system: "You are a Chief Strategy Officer. Synthesize all findings into a go/no-go recommendation.",
    messages: [
      {
        role: "user",
        content: `Synthesize these research findings into a final recommendation:\n\n${synthesisPrompt}`,
      },
    ],
  });

  return finalReport;
}
```

### Using the Agent SDK with Subagents

```typescript
// With the Agent SDK, you can define subagents programmatically
import { claude } from "@anthropic-ai/claude-code";

const session = await claude({
  allowedTools: ["Read", "Write", "Edit", "Bash", "Task", "WebSearch"],
  agents: [
    {
      name: "researcher",
      description: "Market research. Use for competitive analysis and market sizing.",
      tools: ["Read", "Write", "WebSearch", "WebFetch"],
      model: "sonnet",
      prompt: "You are a senior market researcher..."
    },
    {
      name: "finance",
      description: "Financial modeling. Use for revenue projections and ROI.",
      tools: ["Read", "Write", "Bash"],
      model: "sonnet",
      prompt: "You are a financial analyst..."
    }
  ]
});

// Claude will automatically delegate to subagents when appropriate
const result = await session.send(
  "Evaluate the meal planning SaaS opportunity. Use the researcher for market data and finance for projections."
);
```

---

## Troubleshooting & Best Practices

### Common Issues

| Problem | Solution |
|---------|----------|
| Agent doesn't start | Check `.claude/agents/` file format — needs YAML frontmatter with `---` delimiters |
| Agent can't find files | Use absolute paths or paths relative to project root |
| Agent Teams not working | Ensure `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is set |
| Agents write conflicting files | Each agent should write to its own file, never share output files |
| Context window overflow | Use `model: haiku` for simple agents, `sonnet` for complex ones |
| Background task seems stuck | Press `Ctrl+B` to check background task status |

### Token Cost Management

Agent teams multiply token usage. Here's how to manage costs:

```markdown
## Cost Optimization Rules (add to CLAUDE.md)

1. Use model: haiku for research/exploration agents (cheaper, faster)
2. Use model: sonnet for synthesis/analysis agents (smarter)
3. Keep agent outputs concise — bullet points over paragraphs
4. Limit web search to 5 queries per agent
5. Set maximum output length per agent: 2000 words
```

### Best Practices

1. **Start simple**: Begin with subagents (Method A) before trying Agent Teams
2. **One file per agent**: Never have two agents write to the same file
3. **Explicit is better**: Give agents very specific instructions, not vague ones
4. **Test incrementally**: Run one agent at a time first, then combine
5. **Use CLAUDE.md**: This is your project's constitution — make it detailed
6. **Monitor costs**: Agent teams use 3-4x tokens of a single session
7. **Design for failure**: If one agent fails, the system should still produce partial results
8. **Version your agents**: Keep agent definitions in git — iterate on their prompts

### Recommended Learning Path

```
Week 1: Single subagent experiments
         → Create one custom agent, run it, review output quality

Week 2: Parallel subagents
         → Run 2-3 agents in parallel, practice file-based communication

Week 3: Agent Teams
         → Enable experimental flag, try team messaging and shared tasks

Week 4: Production patterns
         → Agent SDK, programmatic orchestration, error handling

Week 5: Advanced patterns
         → Debate pattern, iterative refinement, portfolio evaluation
```

---

## Quick Reference Card

### File Structure

```
your-project/
├── CLAUDE.md                        # Project-level instructions (all agents read this)
├── .claude/
│   ├── agents/
│   │   ├── researcher.md            # Research agent definition
│   │   ├── marketer.md              # Marketing agent definition
│   │   ├── finance.md               # Finance agent definition
│   │   └── analyst.md               # Analysis agent definition
│   └── settings.json                # Claude Code settings
├── docs/
│   ├── research/
│   │   ├── researcher-findings.md   # Research output
│   │   ├── marketer-findings.md     # Marketing output
│   │   └── finance-findings.md      # Finance output
│   └── reports/
│       └── final-recommendation.md  # Final deliverable
└── data/
    └── handoffs/                    # Inter-agent communication files
```

### Key Commands

```bash
# Start Claude Code
claude

# Start with Agent Teams enabled
CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude

# Check running background tasks
# (Press Ctrl+B inside Claude Code)

# View agent team status
# (Claude Code shows this in the UI)
```

### Useful Prompts

```
# Quick single-agent test
"Use the researcher agent to analyze the competitive landscape for [product]"

# Full parallel pipeline
"Run researcher, marketer, and finance agents in parallel to evaluate [product],
then have the analyst synthesize a recommendation"

# Agent team with debate
"Create an agent team. Have researcher and marketer investigate [product]
from their perspectives, then debate the key risks before the analyst
produces the final recommendation"
```

---

## Further Reading

- **Claude Code Docs — Subagents**: https://code.claude.com/docs/en/sub-agents
- **Claude Code Docs — Agent Teams**: https://code.claude.com/docs/en/agent-teams
- **Agent SDK Documentation**: https://platform.claude.com/docs/en/agent-sdk/subagents
- **LangGraph for Python Orchestration**: https://langchain-ai.github.io/langgraph/
- **Awesome Claude Code Subagents**: https://github.com/VoltAgent/awesome-claude-code-subagents

---

*Built with Claude Code. Orchestrate intelligently.*
