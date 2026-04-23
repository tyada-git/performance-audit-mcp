# 🚀 Frontend Performance Audit MCP Server

An MCP (Model Context Protocol) server that bridges Claude AI with Google Lighthouse to deliver real-time, code-level frontend performance auditing — including GitHub repository analysis for file-specific fix suggestions.

---

## 🧠 Thought Process

The idea started from a simple problem: performance auditing tools like Lighthouse give you numbers, but they don't tell you _where_ in your codebase to fix things. You get "LCP is 19.9s — Poor" but you're left searching through your own files to figure out what's causing it.

The goal was to build something that:

- Runs performance audits automatically without leaving your AI chat
- Returns structured Core Web Vitals data Claude can reason about
- Goes beyond generic suggestions and points at _your actual code_
- Demonstrates a real-world, production-relevant use of the MCP protocol

Rather than building another dashboard or CLI tool, we chose MCP because it lets Claude act as the interface — you just describe what you want in plain English and Claude calls the right tool at the right time.

---

## 🗺️ How We Proceeded

### Phase 1 — Foundation

We started by understanding the core constraint: `lighthouse` and `chrome-launcher` are ESM-only packages, which meant the entire project had to be set up as an ES Module from the start. This required careful TypeScript configuration with `"module": "NodeNext"` and `"moduleResolution": "NodeNext"` to ensure compatibility.

**Project setup:**

- Initialised a Node.js project with TypeScript
- Configured `tsconfig.json` for ESM compatibility
- Installed `@modelcontextprotocol/sdk`, `lighthouse`, and `chrome-launcher`

### Phase 2 — Lighthouse Engine (`auditor.ts`)

We built a clean abstraction over Lighthouse that:

- Launches Chrome in headless mode programmatically using `chrome-launcher`
- Runs a full Lighthouse audit restricted to the performance category
- Extracts all six Core Web Vitals from the raw Lighthouse result
- Rates each metric as `good`, `needs-improvement`, or `poor` using Google's official thresholds
- Returns a strongly typed `AuditResult` object
- Always kills the Chrome instance in a `finally` block to prevent process leaks

### Phase 3 — Fix Suggestion Engine (`suggestions.ts`)

We built a metric-to-fix lookup engine that:

- Maps each metric and rating combination to a curated list of code-level fix suggestions
- Assigns priority (`high`, `medium`, `low`) based on metric rating
- Sorts all fixes by priority so the worst problems appear first
- Extracts Lighthouse "opportunities" which include estimated byte/time savings
- Generates a plain English summary paragraph describing the overall result

### Phase 4 — MCP Server (`index.ts`)

We wired everything into three MCP tools Claude can call:

| Tool                    | Description                                       |
| ----------------------- | ------------------------------------------------- |
| `run_performance_audit` | Full Lighthouse audit with Core Web Vitals scores |
| `get_fix_suggestions`   | Prioritised code-level fix suggestions per metric |
| `compare_audits`        | Side-by-side comparison of two URLs               |

### Phase 5 — GitHub Integration (`github.ts`)

After the first test run, we identified the core limitation of the generic suggestion approach — it couldn't reference actual file names or component structures because it had no access to the codebase. This led to Phase 5.

We added a fourth tool (`analyze_with_repo`) that:

- Accepts both a URL and a GitHub repository URL
- Runs Lighthouse to identify which metrics are failing
- Uses those failing metrics to decide _which files_ are most relevant to fetch
- Fetches the actual source files from GitHub via the Octokit REST API
- Passes both the audit results and real source code to Claude
- Lets Claude reason about _your specific code_ to give file-level, line-level suggestions

---

## 🔧 What We Built

### Architecture Overview

```
performance-audit-mcp/
├── src/
│   ├── index.ts          → MCP server — registers all 4 tools
│   ├── auditor.ts        → Lighthouse engine — runs audits, extracts CWVs
│   ├── suggestions.ts    → Fix suggestion engine — maps metrics to fixes
│   └── github.ts         → GitHub integration — fetches relevant source files
├── build/                → Compiled JavaScript output
├── package.json
├── tsconfig.json
└── .env                  → GITHUB_TOKEN
```

### The 4 MCP Tools

#### Tool 1 — `run_performance_audit`

Runs a full Lighthouse audit on any URL and returns all six Core Web Vitals with ratings and opportunities.

```
Input:   { url: "http://localhost:5173/products" }
Output:  LCP, CLS, FID, INP, FCP, TTFB scores with good/needs-improvement/poor ratings
```

#### Tool 2 — `get_fix_suggestions`

Runs an audit and returns prioritised, code-level fix suggestions for every failing metric.

```
Input:   { url: "http://localhost:5173/products" }
Output:  HIGH/MEDIUM/LOW priority fixes sorted by impact
```

#### Tool 3 — `compare_audits`

Runs Lighthouse on two URLs and returns a side-by-side comparison table.

```
Input:   { url1: "http://localhost:5173/", url2: "http://localhost:5173/products" }
Output:  Metric-by-metric comparison of both pages
```

#### Tool 4 — `analyze_with_repo`

The most powerful tool. Combines Lighthouse audit results with actual GitHub source code to give file-specific, line-level suggestions.

```
Input:   { url: "http://localhost:5173/products", repoUrl: "https://github.com/owner/repo" }

Steps:
  1. Runs Lighthouse → identifies failing metrics
  2. Maps failing metrics to relevant file paths
     (LCP failing → fetch App.tsx, components/, pages/)
  3. Fetches those files from GitHub via Octokit API
  4. Passes audit results + real source code to Claude
  5. Claude analyzes YOUR actual code and returns:
     - Exact file names and line numbers
     - Before/after code snippets
     - Priority-ordered fix plan
```

### Core Web Vitals Covered

| Metric                          | Measures                                  | Good Threshold |
| ------------------------------- | ----------------------------------------- | -------------- |
| LCP — Largest Contentful Paint  | How fast the biggest element loads        | < 2.5s         |
| CLS — Cumulative Layout Shift   | How much the page jumps around            | < 0.1          |
| FID — First Input Delay (TBT)   | How fast the page responds to first click | < 100ms        |
| INP — Interaction to Next Paint | How responsive all interactions are       | < 200ms        |
| FCP — First Contentful Paint    | How fast first content appears            | < 1.8s         |
| TTFB — Time to First Byte       | How fast the server responds              | < 600ms        |

---

## 📈 Improvements Made

### From Generic to Specific

The first version returned hardcoded suggestion strings like "Add `loading='lazy'` to your hero image". While useful as a starting point, this was not specific enough — the user still had to search their own codebase to find where to apply the fix.

The GitHub integration solved this by giving Claude access to the actual source code, transforming suggestions from generic best practices into file-specific, component-aware recommendations.

### From Single Audit to Comparison

We added `compare_audits` after recognising a common real-world need — developers often want to compare the performance of different pages (home vs product listing vs checkout) or compare before/after a deployment.

### Production Architecture Awareness

A key architectural decision was acknowledging the limitation of local file path access in production. Rather than reading files directly from the filesystem (which only works locally), we chose the GitHub API approach — the same pattern used by production tools like Lighthouse CI, CodeClimate, and SonarQube. This makes the tool's architecture production-ready even in its current form.

---

## 🛠️ Tech Stack

| Technology                  | Purpose                                 |
| --------------------------- | --------------------------------------- |
| TypeScript                  | Type-safe development                   |
| `@modelcontextprotocol/sdk` | MCP server protocol (Anthropic)         |
| `lighthouse`                | Google's performance audit engine       |
| `chrome-launcher`           | Programmatic headless Chrome management |
| `@octokit/rest`             | GitHub REST API client                  |
| `dotenv`                    | Environment variable management         |
| `zod`                       | Tool input schema validation            |

---

## 🚀 How to Run Locally

### Prerequisites

- Node.js 18+
- Google Chrome installed
- Claude Desktop app
- GitHub Personal Access Token with `repo` scope

### Setup

```bash
git clone https://github.com/yourusername/performance-audit-mcp
cd performance-audit-mcp
npm install
```

Create a `.env` file:

```
GITHUB_TOKEN=your_github_token_here
```

Build the project:

```bash
npm run build
```

### Connect to Claude Desktop

Open Claude Desktop → Settings → Developer → Edit Config and add:

```json
{
  "mcpServers": {
    "performance-audit-mcp": {
      "command": "node",
      "args": ["/absolute/path/to/performance-audit-mcp/build/index.js"],
      "env": {
        "GITHUB_TOKEN": "your_github_token_here"
      }
    }
  }
}
```

Restart Claude Desktop.

### Usage

In Claude Desktop, simply ask:

```
Audit the performance of http://localhost:3000

Get fix suggestions for http://localhost:3000

Compare http://localhost:3000 and http://localhost:3000/products

Analyze http://localhost:3000 using my repo https://github.com/me/my-app
and give me file-specific performance fixes
```

---

## 🌍 Using This MCP Server in a Real Project (Beyond Claude Desktop)

Claude Desktop is great for local development and demos but it is not how teams use MCP servers in production. Here are all the real-world integration patterns, from simplest to most powerful.

---

### The Core Concept First

Claude Desktop is just one **MCP client** — it is the thing that starts your server and calls your tools. In production, anything built on the Anthropic API can act as that client. Your MCP server code does not change at all. Only the **client** changes.

```
Local Dev:                     Production:

Claude Desktop                 Your App / CI Pipeline / Internal Tool
      ↓                                      ↓
MCP Server (your code)         MCP Server (your code) ← same server, unchanged
      ↓                                      ↓
Lighthouse + GitHub API        Lighthouse + GitHub API
```

---

### Option 1 — Publish to npm and Let Anyone Install It

This is how most public MCP servers are distributed. You publish your server to npm and anyone can add it to their Claude Desktop or any MCP client with a single line in their config.

**Step 1 — Publish your server**

```bash
# Add "bin" to your package.json so it runs as a CLI
# Then publish
npm publish --access public
```

**Step 2 — Anyone on your team adds this to their Claude Desktop config**

```json
{
  "mcpServers": {
    "performance-audit-mcp": {
      "command": "npx",
      "args": ["-y", "@yourusername/performance-audit-mcp"],
      "env": {
        "GITHUB_TOKEN": "their_own_token_here"
      }
    }
  }
}
```

That is it. No cloning, no building. `npx` downloads and runs it automatically.

**Best for:** Open source tools you want the community to use. Developer tools your whole company uses on their local machines.

---

### Option 2 — Use It via the Anthropic API (In Your Own App)

If your friend or teammate is building an app that uses the Anthropic API, they can call your MCP server directly from their backend code without Claude Desktop at all.

This is how you would integrate it into a Next.js app, an Express API, or any backend service.

**Step 1 — Run your MCP server as a persistent process on a server**

```bash
# On any server (AWS EC2, DigitalOcean, Railway, Render)
node /path/to/performance-audit-mcp/build/index.js
```

**Step 2 — Connect to it from the Anthropic API using the MCP client SDK**

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { Client } from "@modelcontextprotocol/sdk/client/index.js";
import { StdioClientTransport } from "@modelcontextprotocol/sdk/client/stdio.js";

// Start your MCP server as a subprocess
const transport = new StdioClientTransport({
  command: "node",
  args: ["/path/to/performance-audit-mcp/build/index.js"],
  env: { GITHUB_TOKEN: process.env.GITHUB_TOKEN },
});

// Connect the MCP client to your server
const mcpClient = new Client({ name: "my-app", version: "1.0.0" }, {});
await mcpClient.connect(transport);

// Get the list of tools your server exposes
const { tools } = await mcpClient.listTools();

// Call Claude with your tools available
const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });

const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  max_tokens: 4096,
  tools: tools, // ← your MCP tools passed directly to Claude
  messages: [
    {
      role: "user",
      content:
        "Audit https://mysite.com and give me file-specific fixes for my repo https://github.com/me/my-app",
    },
  ],
});
```

Claude will automatically call `run_performance_audit`, `analyze_with_repo`, or any of your tools based on what the user asked.

**Best for:** Building your own internal tools, web apps, Slack bots, or dashboards that use your MCP server as a backend service.

---

### Option 3 — GitHub Actions CI/CD Integration

This is the most powerful production pattern. Every time a pull request is opened, your GitHub Actions pipeline automatically runs a performance audit and posts the results as a PR comment. No one has to remember to check performance — it is enforced automatically.

**Step 1 — Add your MCP server to your project as a dev dependency**

```bash
npm install --save-dev @yourusername/performance-audit-mcp
```

**Step 2 — Create `.github/workflows/performance-audit.yml` in your repo**

```yaml
name: Performance Audit

on:
  pull_request:
    branches: [main]

jobs:
  audit:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install dependencies
        run: npm install

      - name: Install Chrome
        run: |
          sudo apt-get update
          sudo apt-get install -y google-chrome-stable

      - name: Build app
        run: npm run build

      - name: Start app in background
        run: npm run preview &
        # Starts your built app on http://localhost:4173

      - name: Wait for app to be ready
        run: npx wait-on http://localhost:4173 --timeout 30000

      - name: Run Performance Audit
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        run: |
          node scripts/run-audit.mjs http://localhost:4173

      - name: Post results as PR comment
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('audit-report.txt', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## 🚀 Performance Audit Results\n\`\`\`\n${report}\n\`\`\``
            });
```

**Step 3 — Create `scripts/run-audit.mjs` in your project**

```javascript
import { runAudit } from "@yourusername/performance-audit-mcp";
import { writeFileSync } from "fs";

const url = process.argv[2];
const result = await runAudit(url);
const { coreWebVitals: cwv } = result;

const report = `
URL: ${url}
Score: ${cwv.performanceScore}/100

LCP:  ${cwv.lcp.displayValue}  → ${cwv.lcp.rating.toUpperCase()}
CLS:  ${cwv.cls.displayValue}  → ${cwv.cls.rating.toUpperCase()}
FCP:  ${cwv.fcp.displayValue}  → ${cwv.fcp.rating.toUpperCase()}
TTFB: ${cwv.ttfb.displayValue} → ${cwv.ttfb.rating.toUpperCase()}
`.trim();

writeFileSync("audit-report.txt", report);
console.log(report);

// Fail the CI build if score is below 70
if (cwv.performanceScore < 70) {
  console.error("Performance score below threshold (70). Failing build.");
  process.exit(1);
}
```

Now every PR automatically gets a performance report posted as a comment and the build fails if scores drop below your threshold.

**Best for:** Teams that want performance to be a hard requirement on every PR, not an afterthought.

---

### Option 4 — Internal Slack Bot or Dashboard

Your team can interact with the MCP server through a Slack bot instead of Claude Desktop. Anyone on the team can type a command in Slack and get a performance report back without setting up anything locally.

```
Developer types in Slack:
  /audit https://staging.myapp.com

Slack bot calls your backend →
Backend calls MCP server →
MCP server runs Lighthouse →
Result posted back in Slack thread
```

The MCP server runs on a shared server (AWS, Railway, Render) and everyone on the team uses it through the Slack interface. No one needs Claude Desktop installed.

**Best for:** Teams that want a shared, always-on performance monitoring tool accessible to everyone including non-developers.

---

### Summary — Which Option to Choose

| Scenario                                       | Best Option                                |
| ---------------------------------------------- | ------------------------------------------ |
| Your teammate wants to use it locally          | Option 1 — publish to npm                  |
| You are building a web app that needs auditing | Option 2 — Anthropic API integration       |
| You want automated PR performance checks       | Option 3 — GitHub Actions                  |
| Your whole team wants easy access              | Option 4 — Slack bot or internal dashboard |
| Just learning / demoing                        | Claude Desktop (what we built)             |

The key insight is that **your MCP server code never changes** across all these options. You write the tools once. The client (Claude Desktop, your app, GitHub Actions, Slack) is just the interface layer on top.

---

## 🔮 Future Improvements

- **CI/CD integration** — Run audits automatically on every pull request via GitHub Actions
- **Historical tracking** — Store audit results over time to track performance regressions
- **Source map support** — Map minified production bundles back to original source files for production auditing
- **Multi-page crawling** — Automatically discover and audit all pages in a site
- **Slack/Teams notifications** — Alert the team when performance scores drop below a threshold

---

## 👩‍💻 Author

Built as a learning project to explore the Model Context Protocol (MCP) and real-world AI tooling integration. Designed with production architecture patterns in mind — GitHub API integration instead of local filesystem access, structured typed outputs, and graceful error handling throughout.
