# Part 2: Equipping Business Agents with Skills & MCP Servers

## Giving Your Agents Superpowers — External Tools, Knowledge, and Real-World Connectivity

> **Prerequisite**: Read Part 1 first (`building-multi-agent-business-systems-with-claude-code.md`)
> This guide adds Skills (reusable expertise) and MCP (external tool connections) to your agents.

---

## Table of Contents

1. [The Three Building Blocks](#1-the-three-building-blocks)
2. [How Skills and MCP Work in Agents](#2-how-skills-and-mcp-work-in-agents)
3. [Step 1: Build Custom Skills for Business Research](#step-1-build-custom-skills-for-business-research)
4. [Step 2: Connect MCP Servers for External Data](#step-2-connect-mcp-servers-for-external-data)
5. [Step 3: Wire Skills + MCP into Your Subagents](#step-3-wire-skills--mcp-into-your-subagents)
6. [Step 4: Full Working Example — Equipped Agent Team](#step-4-full-working-example--equipped-agent-team)
7. [Step 5: Agent Teams with Shared Skills + MCP](#step-5-agent-teams-with-shared-skills--mcp)
8. [Advanced: Building a Plugin for Your Whole Team](#advanced-building-a-plugin-for-your-whole-team)
9. [Architecture Decision Guide](#architecture-decision-guide)
10. [Troubleshooting & Performance](#troubleshooting--performance)

---

## 1. The Three Building Blocks

Your agents have three types of capabilities. Think of it like equipping a character in a game:

```
┌─────────────────────────────────────────────────────────────┐
│                     YOUR AGENT                              │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │   SKILLS    │  │    MCP      │  │   BUILT-IN TOOLS    │ │
│  │             │  │  SERVERS    │  │                     │ │
│  │ "Knowledge" │  │ "Hands"    │  │ "Senses"           │ │
│  │             │  │             │  │                     │ │
│  │ How to do   │  │ Connect to  │  │ Read, Write, Edit,  │ │
│  │ SWOT analysis│ │ Google Drive│  │ Bash, Grep, Glob,   │ │
│  │ How to model│  │ Slack       │  │ WebSearch, WebFetch  │ │
│  │ financials  │  │ Databases   │  │                     │ │
│  │ How to write│  │ CRM (HubSpot│  │ (Always available)   │ │
│  │ reports     │  │ Notion      │  │                     │ │
│  │             │  │ GitHub      │  │                     │ │
│  └─────────────┘  └─────────────┘  └─────────────────────┘ │
│                                                             │
│  Skills = Lazy-loaded expertise (low token cost)            │
│  MCP = Real-time external connections (moderate token cost) │
│  Tools = Built-in capabilities (always present)             │
└─────────────────────────────────────────────────────────────┘
```

### Quick Comparison

| Feature | Skills | MCP Servers | Built-in Tools |
|---------|--------|-------------|----------------|
| What it does | Gives **knowledge/procedures** | Gives **external access** | Gives **local actions** |
| Token cost | Very low (lazy-loaded) | Moderate (schema always loaded) | Minimal |
| Example | "How to do competitive analysis" | "Search Google Drive for docs" | "Read a file" |
| When loaded | On-demand when relevant | At session start | Always |
| Best for | Reusable expertise, templates | APIs, databases, SaaS tools | File ops, search, code |

### The Key Insight

**Skills + MCP + Subagents work in harmony:**

- The **Skill** provides expertise and instructions ("how to analyze a market")
- The **MCP server** provides capability ("search Google Drive for market reports")
- The **Subagent** provides context isolation (keeps the main agent clean)

Each subagent can have its **own** Skills and MCP servers — the main agent doesn't pay the token cost for tools it never uses.

---

## 2. How Skills and MCP Work in Agents

### Skills: Progressive Disclosure

Skills use a smart loading system:

```
Session Start:
  → Load ONLY names + descriptions of all skills (~100 tokens each)
  → Full skill content stays on disk

User asks something relevant:
  → Claude matches the task to a skill description
  → THEN loads the full SKILL.md content (~5,000 tokens typical)
  → Follows the skill's instructions

Task complete:
  → Skill content can be released from context
```

This means you can have 50+ skills available and only pay for the ones actually used.

### MCP: Always-On Connections

MCP servers work differently — their tool schemas load at session start:

```
Session Start:
  → Connect to all configured MCP servers
  → Load ALL tool schemas into context (~500-2000 tokens per server)
  → Tools remain available throughout session

Agent calls an MCP tool:
  → JSON-RPC request to MCP server
  → Server executes (API call, DB query, etc.)
  → Returns result to agent
```

**Warning**: Too many MCP servers bloat your context. Keep to 2-3 per agent.

### Subagent Inheritance Rules

```
┌─────────────────────────────────────────────────────┐
│ WHAT SUBAGENTS GET                                  │
│                                                     │
│ ✅ Inherits from project:                           │
│    - CLAUDE.md instructions                         │
│    - Project-level MCP servers (if tools not locked) │
│    - Project-level skills                           │
│                                                     │
│ ❌ Does NOT inherit:                                │
│    - Parent conversation history                    │
│    - Parent's in-memory state                       │
│                                                     │
│ 🔧 Can override:                                    │
│    - tools (whitelist specific tools)               │
│    - mcpServers (add agent-specific MCP)            │
│    - skills (preload specific skills)               │
│    - model (use different model)                    │
│                                                     │
│ ⚠️ If you set `tools:` explicitly, the agent        │
│    ONLY gets those tools. Omit it to inherit all.   │
└─────────────────────────────────────────────────────┘
```

---

## Step 1: Build Custom Skills for Business Research

### Skill Directory Structure

```
your-project/
├── .claude/
│   ├── skills/
│   │   ├── competitive-analysis/
│   │   │   ├── SKILL.md              # Main skill instructions
│   │   │   ├── templates/
│   │   │   │   └── competitor-matrix.md
│   │   │   └── scripts/
│   │   │       └── extract-data.py
│   │   ├── financial-modeling/
│   │   │   ├── SKILL.md
│   │   │   └── templates/
│   │   │       ├── revenue-model.md
│   │   │       └── unit-economics.md
│   │   ├── market-sizing/
│   │   │   └── SKILL.md
│   │   ├── gtm-strategy/
│   │   │   └── SKILL.md
│   │   └── executive-report/
│   │       ├── SKILL.md
│   │       └── templates/
│   │           └── recommendation-template.md
│   └── agents/
│       └── (your agents from Part 1)
```

### Skill 1: Competitive Analysis

Create `.claude/skills/competitive-analysis/SKILL.md`:

```markdown
---
name: competitive-analysis
description: Conduct structured competitive analysis including competitor identification, feature comparison, positioning maps, and strategic implications. Use when analyzing competitors, market positioning, or competitive landscape.
---

# Competitive Analysis Framework

## When to Use
Activate this skill when asked to analyze competitors, compare products,
assess competitive positioning, or evaluate market landscape.

## Analysis Protocol

### Phase 1: Competitor Identification
1. Identify direct competitors (same product category)
2. Identify indirect competitors (alternative solutions)
3. Identify potential future competitors (adjacent markets)
4. Categorize by tier: Primary (top 3), Secondary (next 5), Emerging

### Phase 2: Feature Matrix
Build a comparison table using this structure:

| Feature | Our Product | Competitor A | Competitor B | Competitor C |
|---------|------------|-------------|-------------|-------------|
| Core Feature 1 | ✅/❌/🔶 | ✅/❌/🔶 | ✅/❌/🔶 | ✅/❌/🔶 |
| Pricing | $X/mo | $X/mo | $X/mo | $X/mo |
| Target Segment | ... | ... | ... | ... |

Legend: ✅ Strong  🔶 Partial  ❌ Missing

### Phase 3: Positioning Analysis
For each major competitor, assess:
- **Value Proposition**: What promise do they make?
- **Target Customer**: Who are they serving?
- **Pricing Strategy**: Premium, mid-market, or budget?
- **Differentiation**: What makes them unique?
- **Weaknesses**: Where do they fall short?

### Phase 4: Strategic Implications
1. Identify gaps in competitor offerings (opportunity zones)
2. Assess barriers to entry
3. Map switching costs for target customers
4. Evaluate competitive moats (network effects, data, brand, etc.)

### Phase 5: Competitive Response Scenarios
- If we enter, how will competitors likely respond?
- What is our sustainable competitive advantage?
- What would it take for a competitor to copy our approach?

## Output Format
Write the complete analysis with:
- Executive Summary (3-5 bullets)
- Competitor Overview Table
- Feature Comparison Matrix
- Positioning Map (describe, or ASCII if possible)
- Gap Analysis
- Strategic Recommendations
- Sources and Confidence Level

## Data Sources
When using web search:
- Search for "[competitor] pricing" for pricing data
- Search for "[competitor] reviews" for user sentiment
- Search for "[competitor] funding" for investment signals
- Search for "best [category] software 2026" for market lists
- Check Crunchbase, G2, Capterra for structured data
```

### Skill 2: Financial Modeling

Create `.claude/skills/financial-modeling/SKILL.md`:

```markdown
---
name: financial-modeling
description: Build financial models including revenue projections, unit economics, break-even analysis, and cash flow forecasts. Use when creating financial projections, analyzing profitability, or estimating ROI.
---

# Financial Modeling Framework

## When to Use
Activate for revenue modeling, cost estimation, unit economics,
break-even analysis, ROI calculations, or financial projections.

## Standard Models

### Model 1: SaaS Revenue Projection
```python
# Template: Run via bash for calculations
def saas_revenue_model(
    monthly_price,
    starting_customers,
    monthly_growth_rate,
    monthly_churn_rate,
    months=36
):
    customers = starting_customers
    revenue_data = []
    for month in range(1, months + 1):
        mrr = customers * monthly_price
        new_customers = customers * monthly_growth_rate
        churned = customers * monthly_churn_rate
        customers = customers + new_customers - churned
        revenue_data.append({
            'month': month,
            'customers': round(customers),
            'mrr': round(mrr, 2),
            'arr': round(mrr * 12, 2)
        })
    return revenue_data
```

### Model 2: Unit Economics
Key metrics to calculate:
- **CAC** (Customer Acquisition Cost) = Total Marketing Spend / New Customers
- **LTV** (Lifetime Value) = ARPU × Gross Margin / Monthly Churn Rate
- **LTV:CAC Ratio** — Target: > 3:1 for healthy SaaS
- **Payback Period** = CAC / (ARPU × Gross Margin) — Target: < 12 months
- **Gross Margin** = (Revenue - COGS) / Revenue — Target: > 70% for SaaS

### Model 3: Break-Even Analysis
```python
def break_even(fixed_costs_monthly, price_per_unit, variable_cost_per_unit):
    contribution_margin = price_per_unit - variable_cost_per_unit
    units_needed = fixed_costs_monthly / contribution_margin
    return {
        'units_per_month': round(units_needed),
        'revenue_at_breakeven': round(units_needed * price_per_unit, 2),
        'contribution_margin': round(contribution_margin, 2)
    }
```

### Model 4: Cash Flow Projection
Always project:
- 3 scenarios: Conservative (-30%), Base, Optimistic (+30%)
- 12-month and 36-month horizons
- Include: one-time costs, recurring costs, revenue ramp

## Output Standards
- All calculations should be reproducible (show formulas or code)
- State all assumptions explicitly
- Include sensitivity analysis for key variables
- Use tables for projections, not paragraphs
- Flag if LTV:CAC < 3 or payback > 18 months as concerns

## Templates
See `templates/revenue-model.md` and `templates/unit-economics.md` for
pre-built output templates.
```

### Skill 3: GTM Strategy

Create `.claude/skills/gtm-strategy/SKILL.md`:

```markdown
---
name: gtm-strategy
description: Develop go-to-market strategies including customer personas, channel strategy, positioning, messaging, and launch plans. Use when planning market entry, customer acquisition, or marketing strategy.
---

# Go-to-Market Strategy Framework

## When to Use
Activate when developing market entry plans, customer acquisition
strategies, positioning, or marketing plans.

## Framework

### 1. Customer Discovery
Build personas using this template:

**Persona: [Name]**
- Demographics: Age, role, income, location
- Pain Points: Top 3 problems they face
- Current Solutions: What they use today
- Buying Behavior: How they discover and purchase
- Decision Criteria: What matters most (price, features, trust)
- Objections: Why they might NOT buy

### 2. Channel Strategy Matrix

| Channel | CAC Estimate | Time to Result | Scalability | Priority |
|---------|-------------|----------------|-------------|----------|
| Content/SEO | $XX | 6-12 months | High | ... |
| Paid Search | $XX | Immediate | Medium | ... |
| Social Media | $XX | 3-6 months | High | ... |
| Partnerships | $XX | 3-6 months | Medium | ... |
| Cold Outreach | $XX | 1-3 months | Low | ... |
| Product-Led | $XX | 3-6 months | High | ... |

### 3. Positioning Statement
Template: "For [target customer] who [need/problem],
[product name] is a [category] that [key benefit].
Unlike [competitors], we [unique differentiator]."

### 4. First 90 Days Plan
- Days 1-30: Foundation (landing page, waitlist, content)
- Days 31-60: Traction (first customers, feedback loop)
- Days 61-90: Scale (optimize channels, expand reach)

## Data Sources for Research
- Google Trends for demand signals
- Reddit/forums for pain point discovery
- LinkedIn for B2B persona research
- SimilarWeb for competitor traffic estimates
- App stores for mobile market sizing
```

### Skill 4: Executive Report

Create `.claude/skills/executive-report/SKILL.md`:

```markdown
---
name: executive-report
description: Generate executive-level business reports with SWOT analysis, risk matrices, strategic recommendations, and actionable next steps. Use when producing final reports, recommendations, or decision documents.
---

# Executive Report Framework

## When to Use
Activate when synthesizing research into a final business recommendation,
writing executive summaries, or producing decision documents.

## Report Structure

### 1. Executive Summary (Max 1 Page)
- The Opportunity (2 sentences)
- Key Findings (3-5 bullets)
- Recommendation (Go / No-Go / Conditional Go)
- Required Investment and Expected Return

### 2. SWOT Analysis
Present as a 2x2 matrix:

| | Helpful | Harmful |
|---|---------|---------|
| **Internal** | Strengths | Weaknesses |
| **External** | Opportunities | Threats |

### 3. Risk Matrix
Rate each risk on Likelihood (1-5) × Impact (1-5):

| Risk | Likelihood | Impact | Score | Mitigation |
|------|-----------|--------|-------|------------|
| ... | 1-5 | 1-5 | L×I | Strategy |

### 4. Financial Viability
Summarize from financial analysis:
- Investment Required
- Break-even Timeline
- Expected ROI (3-year)
- LTV:CAC Ratio

### 5. Strategic Recommendation
- Clear Go / No-Go / Conditional decision
- If Go: Top 3 next steps with owners and timelines
- If No-Go: What would need to change
- Confidence level and key assumptions

## Writing Standards
- Use active voice
- Lead with conclusions, support with data
- Every claim needs a source or explicit assumption flag
- Keep sections to 1-2 paragraphs max
- Use tables over prose for comparisons
```

---

## Step 2: Connect MCP Servers for External Data

### Which MCP Servers for Business Research?

Here are the most useful MCP servers for your agent team:

```
┌──────────────────────────────────────────────────────────┐
│ RECOMMENDED MCP SERVERS FOR BUSINESS AGENTS              │
│                                                          │
│ 🔍 Research & Data                                       │
│    • Brave Search — Web search (if not using built-in)   │
│    • Exa — AI-native search for research                 │
│    • Firecrawl — Web scraping and data extraction        │
│                                                          │
│ 📄 Documents & Knowledge                                 │
│    • Google Drive — Access company docs, spreadsheets    │
│    • Notion — Access company wiki and databases          │
│    • Confluence — Access company documentation           │
│                                                          │
│ 💬 Communication                                         │
│    • Slack — Post updates, search conversations          │
│    • Gmail — Send reports, search emails                 │
│                                                          │
│ 📊 Data & Analytics                                      │
│    • PostgreSQL (dbhub) — Query business databases       │
│    • Airtable — Access structured business data          │
│    • Google Sheets — Read/write spreadsheet data         │
│                                                          │
│ 🏢 Business Tools                                        │
│    • HubSpot — CRM data, contacts, deals                │
│    • Linear/Jira — Project management                    │
│    • GitHub — Repository access                          │
│                                                          │
│ ⚠️ RULE: Max 2-3 MCP servers per agent for performance   │
└──────────────────────────────────────────────────────────┘
```

### Install MCP Servers

#### Method 1: CLI Commands (Quick Setup)

```bash
# Navigate to your project
cd business-agent-team

# Add Google Drive access
claude mcp add google-drive \
  --transport http \
  -- https://gdrive.mcp.claude.com/mcp

# Add Slack
claude mcp add slack \
  --transport http \
  -- https://slack.mcp.claude.com/mcp

# Add a PostgreSQL database
claude mcp add business-db \
  --transport stdio \
  -- npx -y @bytebase/dbhub \
  --dsn "postgresql://user:pass@host:5432/business_db"

# Add Notion
claude mcp add notion \
  --transport http \
  -- https://mcp.notion.com/mcp

# Add Firecrawl for web scraping
claude mcp add firecrawl \
  --transport stdio \
  --env FIRECRAWL_API_KEY=your-key \
  -- npx -y firecrawl-mcp

# Check what's connected
claude mcp list
```

#### Method 2: Config File (Recommended for Teams)

Create `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "google-drive": {
      "type": "url",
      "url": "https://gdrive.mcp.claude.com/mcp"
    },
    "slack": {
      "type": "url",
      "url": "https://slack.mcp.claude.com/mcp"
    },
    "business-db": {
      "command": "npx",
      "args": ["-y", "@bytebase/dbhub", "--dsn", "postgresql://user:pass@host:5432/business_db"],
      "env": {}
    },
    "firecrawl": {
      "command": "npx",
      "args": ["-y", "firecrawl-mcp"],
      "env": {
        "FIRECRAWL_API_KEY": "your-key-here"
      }
    },
    "exa-search": {
      "command": "npx",
      "args": ["-y", "exa-mcp-server"],
      "env": {
        "EXA_API_KEY": "your-key-here"
      }
    }
  }
}
```

### Verify MCP Connections

```bash
# Start Claude Code
claude

# Inside Claude Code, check context usage
/context

# List available MCP tools
# Claude will show all connected servers and their tools
```

---

## Step 3: Wire Skills + MCP into Your Subagents

### The Magic: Per-Agent Skills and MCP

Each subagent can have its **own** set of Skills and MCP servers.
This means:

- The **researcher** agent gets Google Drive + Firecrawl + competitive-analysis skill
- The **finance** agent gets the database + financial-modeling skill (no need for web scraping)
- The **analyst** agent gets only the executive-report skill (no external tools needed)

**Each agent only pays the token cost for what it actually needs.**

### Updated Agent Definitions with Skills + MCP

#### Research Agent with Skills + MCP

Update `.claude/agents/researcher.md`:

```markdown
---
name: researcher
description: >
  Market research specialist. Use this agent PROACTIVELY when you need to
  research market opportunities, analyze competitors, identify trends,
  or gather industry data.
tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch
mcpServers:
  - google-drive
  - firecrawl
skills:
  - competitive-analysis
  - market-sizing
model: sonnet
maxTurns: 50
background: true
---

You are a **Senior Market Research Analyst** equipped with:

## Your Tools
1. **Web Search** — Search the internet for market data, news, trends
2. **Firecrawl** — Scrape competitor websites for pricing, features
3. **Google Drive** — Access internal company documents and past research
4. **Competitive Analysis Skill** — Structured framework for competitor analysis
5. **Market Sizing Skill** — TAM/SAM/SOM estimation framework

## Research Protocol

### Step 1: Check Internal Knowledge First
Before searching the web, check Google Drive for:
- Existing market research documents
- Previous competitive analyses
- Internal strategy documents
Use the Google Drive search tool to find relevant docs.

### Step 2: Web Research
Use WebSearch for:
- Market size and growth data
- Competitor information
- Industry trends and news

Use Firecrawl to:
- Extract detailed pricing from competitor websites
- Scrape feature comparison pages
- Pull product documentation

### Step 3: Apply Analysis Frameworks
Use the competitive-analysis skill for structured competitor evaluation.
Use the market-sizing skill for TAM/SAM/SOM calculations.

### Step 4: Write Findings
Save complete findings to `docs/research/researcher-findings.md`

## Communication
When done, send a summary of key findings.
Flag anything critical that finance or marketing agents need to know.
```

#### Marketing Agent with Skills + MCP

Update `.claude/agents/marketer.md`:

```markdown
---
name: marketer
description: >
  Marketing strategy specialist. Use this agent PROACTIVELY when you need
  GTM strategy, customer personas, channel analysis, or positioning work.
tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch
mcpServers:
  - google-drive
  - firecrawl
skills:
  - gtm-strategy
model: sonnet
maxTurns: 50
background: true
---

You are a **Chief Marketing Strategist** equipped with:

## Your Tools
1. **Web Search** — Research channels, audiences, benchmarks
2. **Firecrawl** — Analyze competitor marketing (landing pages, messaging)
3. **Google Drive** — Access internal marketing data and brand guides
4. **GTM Strategy Skill** — Structured go-to-market framework

## Research Protocol

### Step 1: Internal Context
Search Google Drive for:
- Existing customer data or personas
- Brand guidelines
- Past marketing performance data

### Step 2: Market Research
Use web search for:
- Industry CAC benchmarks
- Channel effectiveness data
- Audience demographics

Use Firecrawl to:
- Analyze competitor landing pages and messaging
- Extract competitor positioning and pricing

### Step 3: Apply GTM Framework
Follow the gtm-strategy skill for structured analysis.

### Step 4: Deliverables
Write findings to `docs/research/marketer-findings.md`
Include CAC/LTV estimates for the finance agent.
```

#### Financial Agent with Skills + MCP

Update `.claude/agents/finance.md`:

```markdown
---
name: finance
description: >
  Financial analysis specialist. Use this agent PROACTIVELY when you need
  revenue modeling, cost estimation, unit economics, ROI analysis, or
  financial projections.
tools: Read, Write, Edit, Bash, Glob, Grep
mcpServers:
  - business-db
skills:
  - financial-modeling
model: sonnet
maxTurns: 40
background: true
---

You are a **Senior Financial Analyst** equipped with:

## Your Tools
1. **Bash** — Run Python calculations for financial models
2. **Business Database** — Query real business data via SQL
3. **Financial Modeling Skill** — Revenue models, unit economics, break-even

## Analysis Protocol

### Step 1: Gather Input Data
Read findings from other agents:
- `docs/research/researcher-findings.md` for market data
- `docs/research/marketer-findings.md` for CAC/LTV estimates

Query business database for:
- Historical revenue data
- Cost structures
- Customer metrics

### Step 2: Build Models
Use the financial-modeling skill templates.
Run all calculations via bash/Python for reproducibility.

### Step 3: Deliverables
Write findings to `docs/research/finance-findings.md`
All projections in 3 scenarios (conservative, base, optimistic).
```

#### Analysis Agent with Skills (No MCP Needed)

Update `.claude/agents/analyst.md`:

```markdown
---
name: analyst
description: >
  Strategic synthesis analyst. Use this agent to synthesize findings from
  all research agents and produce a final recommendation report.
tools: Read, Write, Edit, Glob, Grep
skills:
  - executive-report
  - competitive-analysis
model: sonnet
maxTurns: 30
---

You are a **Chief Strategy Officer** equipped with:

## Your Tools
1. **Read** — Read all agent findings
2. **Executive Report Skill** — Structured report framework
3. **Competitive Analysis Skill** — For cross-referencing competitor findings

## Synthesis Protocol

### Step 1: Read All Findings
- `docs/research/researcher-findings.md`
- `docs/research/marketer-findings.md`
- `docs/research/finance-findings.md`

### Step 2: Cross-Reference
Look for contradictions and alignment across agents.
Use the competitive-analysis skill framework for validation.

### Step 3: Produce Report
Follow the executive-report skill for the final output.
Write to `docs/reports/final-recommendation.md`
```

---

## Step 4: Full Working Example — Equipped Agent Team

### Complete Project Setup Script

Run this to set up the entire project:

```bash
#!/bin/bash
# setup-business-agents.sh

# Create project structure
mkdir -p business-research-team
cd business-research-team
git init

# Create directories
mkdir -p .claude/agents
mkdir -p .claude/skills/competitive-analysis/templates
mkdir -p .claude/skills/financial-modeling/templates
mkdir -p .claude/skills/gtm-strategy
mkdir -p .claude/skills/market-sizing
mkdir -p .claude/skills/executive-report/templates
mkdir -p docs/research
mkdir -p docs/reports
mkdir -p data/handoffs

echo "✅ Project structure created"
echo ""
echo "Next steps:"
echo "1. Copy agent definitions into .claude/agents/"
echo "2. Copy skill definitions into .claude/skills/"
echo "3. Create .mcp.json with your MCP server configs"
echo "4. Create CLAUDE.md with project instructions"
echo "5. Run: CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1 claude"
```

### The CLAUDE.md (Updated with Skills + MCP)

```markdown
# Business Research Agent Team

## Mission
Multi-agent system for evaluating business opportunities.
Each agent has specialized skills and external tool access.

## Agent Configuration Summary

| Agent | Skills | MCP Servers | Model |
|-------|--------|-------------|-------|
| researcher | competitive-analysis, market-sizing | google-drive, firecrawl | sonnet |
| marketer | gtm-strategy | google-drive, firecrawl | sonnet |
| finance | financial-modeling | business-db | sonnet |
| analyst | executive-report, competitive-analysis | none | sonnet |

## Workflow
1. **Fan-Out**: researcher, marketer, finance run in PARALLEL (background)
2. **Gather**: analyst runs AFTER all three complete
3. **Deliver**: Final report at docs/reports/final-recommendation.md

## Skill Usage Rules
- Agents should invoke their assigned skills automatically
- Skills provide the framework; agents add real data and analysis
- If a skill template exists, use it for consistent output

## MCP Usage Rules
- Always check Google Drive for internal docs BEFORE web searching
- Use Firecrawl only for detailed competitor website analysis
- Database queries should be read-only (SELECT only)
- Limit web searches to 10 per agent to control costs

## Communication Protocol
- Each agent writes to docs/research/[name]-findings.md
- Include a "Key Findings for Other Agents" section at the top
- Flag any CRITICAL discoveries that change the analysis
```

### Run It

```bash
# Set environment
export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1

# Start Claude Code
claude

# Then prompt:
```

```
Evaluate the business opportunity for an AI-powered contract review
tool targeting mid-size law firms (50-200 attorneys).

Use the full agent team:
- researcher: Use competitive-analysis and market-sizing skills.
  Search Google Drive for any existing legal tech research.
  Use Firecrawl to scrape competitor pricing pages.
- marketer: Use the gtm-strategy skill.
  Focus on law firm buying behavior and channels.
- finance: Use the financial-modeling skill.
  Model SaaS pricing at $499/month/firm. Query the business-db for
  comparable deal sizes.
- analyst: Synthesize everything using the executive-report skill.

Run researcher, marketer, and finance in parallel as background tasks.
After they complete, run analyst to produce the final recommendation.
```

---

## Step 5: Agent Teams with Shared Skills + MCP

When using Agent Teams (not just subagents), teammates automatically inherit:

```
┌────────────────────────────────────────────────┐
│ AGENT TEAM INHERITANCE                         │
│                                                │
│ ✅ Auto-inherited by ALL teammates:            │
│    • CLAUDE.md (project context)               │
│    • Project-level MCP servers (.mcp.json)     │
│    • Project-level skills (.claude/skills/)    │
│                                                │
│ ❌ NOT inherited:                              │
│    • Leader's conversation history             │
│    • Other teammates' conversation history     │
│                                                │
│ 💡 Best practice:                              │
│    Include task-specific context in the         │
│    spawn prompt when creating teammates.        │
└────────────────────────────────────────────────┘
```

### Team Launch Prompt with Skills + MCP

```
Create an agent team called "legal-tech-eval" with shared task list.

Spawn teammates:

1. "researcher" with prompt:
   "You are a market researcher for legal tech. You have access to
   Google Drive and Firecrawl MCP servers. Use the competitive-analysis
   skill to evaluate competitors like Kira Systems, Luminance, and
   Ironclad. Write findings to docs/research/researcher-findings.md.
   When done, message 'analyst' with your key findings."

2. "marketer" with prompt:
   "You are a legal tech marketing strategist. Use the gtm-strategy
   skill. Focus on how law firms buy software (long sales cycles,
   partner approval). Use Google Drive to find any existing legal
   market research. Write findings to docs/research/marketer-findings.md.
   When done, message 'finance' with your CAC estimates and
   message 'analyst' with your key findings."

3. "finance" with prompt:
   "You are a financial analyst modeling a legal SaaS at $499/firm/month.
   Use the financial-modeling skill. Wait for marketer's CAC estimates
   (check your inbox). Query business-db for comparable deal metrics.
   Write findings to docs/research/finance-findings.md.
   When done, message 'analyst' with your projections."

4. "analyst" with prompt:
   "You are the Chief Strategy Officer. Wait for messages from
   researcher, marketer, and finance (check your inbox). Read all
   findings. Use the executive-report skill to produce a final
   go/no-go recommendation at docs/reports/final-recommendation.md.
   When done, message 'team-lead' with the recommendation."

Create tasks:
- Task 1: "Market Research" - no dependencies
- Task 2: "GTM Strategy" - no dependencies
- Task 3: "Financial Model" - blocked by Task 2
- Task 4: "Final Analysis" - blocked by Tasks 1, 2, 3
```

---

## Advanced: Building a Plugin for Your Whole Team

If you want to **package** all your agents, skills, and MCP configs into a reusable, shareable unit, build a Plugin:

### Plugin Directory Structure

```
business-research-plugin/
├── .claude-plugin/
│   └── plugin.json                    # Plugin manifest
├── agents/
│   ├── researcher.md
│   ├── marketer.md
│   ├── finance.md
│   └── analyst.md
├── skills/
│   ├── competitive-analysis/
│   │   └── SKILL.md
│   ├── financial-modeling/
│   │   └── SKILL.md
│   ├── gtm-strategy/
│   │   └── SKILL.md
│   └── executive-report/
│       └── SKILL.md
├── commands/
│   └── evaluate.md                    # /evaluate slash command
├── .mcp.json                          # MCP server definitions
└── CHANGELOG.md
```

### Plugin Manifest

Create `.claude-plugin/plugin.json`:

```json
{
  "name": "business-research-team",
  "version": "1.0.0",
  "description": "Multi-agent business research and evaluation system with 4 specialized agents, 4 skills, and MCP integrations.",
  "author": "Your Team",
  "components": {
    "agents": ["agents/"],
    "skills": ["skills/"],
    "commands": ["commands/"]
  }
}
```

### Slash Command for Quick Launch

Create `commands/evaluate.md`:

```markdown
---
name: evaluate
description: Launch a full business opportunity evaluation with all agents
argument-hint: [product description]
---

Launch a full business evaluation for the specified opportunity.

Run these steps:
1. Spawn researcher, marketer, and finance agents in parallel (background)
2. Each agent uses their assigned skills and MCP servers
3. After all complete, spawn analyst for synthesis
4. Produce final report at docs/reports/final-recommendation.md

The opportunity to evaluate: $ARGUMENTS
```

### Install and Use

```bash
# Install the plugin
claude /plugin install ./business-research-plugin

# Use it
claude
> /evaluate AI-powered contract review tool for mid-size law firms
```

---

## Architecture Decision Guide

### When to Use What

```
"I need my agent to KNOW HOW to do something"
    → Use a SKILL
    Example: "How to do a SWOT analysis" → executive-report skill

"I need my agent to ACCESS external data"
    → Use an MCP SERVER
    Example: "Query our sales database" → PostgreSQL MCP

"I need my agent to SEARCH the web"
    → Use BUILT-IN WebSearch/WebFetch tools
    Example: "Find competitor pricing" → WebSearch tool

"I need my agent to SCRAPE a website"
    → Use Firecrawl MCP
    Example: "Extract all features from competitor site" → Firecrawl MCP

"I need my agent to READ internal docs"
    → Use Google Drive / Notion MCP
    Example: "Find our Q3 strategy doc" → Google Drive MCP

"I need agents to TALK TO EACH OTHER"
    → Use AGENT TEAMS with inbox messaging
    Example: "Marketer sends CAC to Finance" → Team messaging

"I need to SHARE this setup with my team"
    → Build a PLUGIN
    Example: Package agents + skills + MCP → Plugin
```

### Token Cost Cheat Sheet

```
Component                   Approximate Token Cost
─────────────────────────────────────────────────
Skill metadata (per skill)  ~100 tokens (always loaded)
Skill full content          ~3,000-5,000 tokens (on-demand)
MCP server schema           ~500-2,000 tokens (always loaded)
MCP tool call + result      ~200-1,000 tokens (per use)
Subagent context            ~500 tokens (at spawn)
Agent Team per teammate     3-4x single session cost
```

### Rule of Thumb

```
Per Agent:
  Skills:      No practical limit (lazy-loaded)
  MCP Servers: Max 2-3 (schemas always consume context)
  Tools:       Be explicit — whitelist only what's needed

Per Project:
  Total MCP servers: Keep under 10 (use per-agent isolation)
  Total skills:      Unlimited (progressive disclosure)
  Simultaneous agents: Up to 10 subagents in parallel
```

---

## Troubleshooting & Performance

### Common Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| Skill not loading | Multi-line `description` in YAML | Keep description on ONE line |
| MCP tool not available | Server not in agent's `mcpServers` | Add to agent frontmatter or remove `tools` whitelist |
| Agent ignores skill | Description doesn't match task | Rewrite description with trigger keywords |
| Context window full | Too many MCP servers loaded | Limit to 2-3 per agent, use per-agent isolation |
| Agent can't use MCP tool | `tools:` whitelist is set | Either add MCP tool names or remove `tools:` to inherit all |
| Skill loads but agent doesn't follow it | Skill instructions too vague | Add specific step-by-step procedures |
| Teammate can't find MCP server | MCP not in project `.mcp.json` | Add to project-level config (teammates inherit project config) |

### Performance Optimization

```markdown
## Add to CLAUDE.md for cost control

### Token Budget Rules
1. Agents should use haiku model for simple lookup tasks
2. Use sonnet for analysis and synthesis tasks
3. Each agent: max 10 web searches, max 5 MCP tool calls
4. Skills should be under 500 lines (split into files if larger)
5. Monitor context with /context command
6. If context exceeds 70%, wrap up and summarize
```

### Debugging Agent Skill Loading

```bash
# Inside Claude Code, check what's loaded:
/context              # See context window usage
/skills               # List available skills
/agents               # List available agents
/mcp                  # Check MCP server status
```

---

## Summary: The Complete Stack

```
┌──────────────────────────────────────────────────┐
│            YOUR BUSINESS AGENT SYSTEM            │
│                                                  │
│  CLAUDE.md          → Project context & rules    │
│  .claude/agents/    → 4 specialized agents       │
│  .claude/skills/    → 4 reusable expertise packs │
│  .mcp.json          → External tool connections  │
│                                                  │
│  Researcher ─── competitive-analysis skill       │
│     │            market-sizing skill             │
│     │            Google Drive MCP                │
│     │            Firecrawl MCP                   │
│     │                                            │
│  Marketer ──── gtm-strategy skill                │
│     │            Google Drive MCP                │
│     │            Firecrawl MCP                   │
│     │                                            │
│  Finance ───── financial-modeling skill           │
│     │            Business DB MCP                 │
│     │                                            │
│  Analyst ───── executive-report skill            │
│                  competitive-analysis skill       │
│                  (no external MCP needed)         │
│                                                  │
│  Orchestration: Fan-Out → Gather → Report        │
│  Communication: Files + Inbox + Task Dependencies│
└──────────────────────────────────────────────────┘
```

---

## Further Reading

- **Claude Code Skills Docs**: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- **Skills Overview (Anthropic Blog)**: https://claude.com/blog/skills-explained
- **Subagent MCP Configuration**: https://code.claude.com/docs/en/sub-agents
- **MCP Protocol**: https://modelcontextprotocol.io
- **Best MCP Servers List**: https://claudefa.st/blog/tools/mcp-extensions/best-addons
- **Skills vs MCP vs Subagents**: https://smithhorngroup.substack.com/p/choosing-between-skills-subagents

---

*Your agents now have knowledge (Skills), hands (MCP), and coordination (Teams). Ship it.*
