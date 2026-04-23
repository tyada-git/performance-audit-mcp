# Performance Audit MCP Server

An MCP (Model Context Protocol) server that brings Google Lighthouse performance auditing and code-level optimization suggestions directly into your AI assistant (like Claude).

This server allows you to audit any public URL, compare performance between environments, and even analyze your GitHub source code to get specific, file-level fix suggestions.

## Features

- **`run_performance_audit`**: Get real-time Core Web Vitals (LCP, CLS, INP, etc.) for any URL.
- **`get_fix_suggestions`**: Receive prioritized, actionable advice on how to improve specific failing metrics.
- **`compare_audits`**: Side-by-side comparison of two URLs (e.g., Localhost vs. Production).
- **`analyze_with_repo`**: **The Killer Feature.** Audits a URL and then scans your GitHub repository to point out exactly which files and lines of code are causing performance bottlenecks.

## Prerequisites

- **Node.js**: v18 or higher.
- **Google Chrome**: Must be installed on the host machine (Lighthouse uses it to run the audit).
- **GitHub Token**: Required for the `analyze_with_repo` tool.

## Installation

1. Clone the repository:
   ```bash
   git clone https://github.com/tyada-git/performance-audit-mcp.git
   cd performance-audit-mcp
   ```

2. Install dependencies:
   ```bash
   npm install
   ```

3. Build the project:
   ```bash
   npm run build
   ```

4. Create a `.env` file in the root directory:
   ```env
   GITHUB_TOKEN=your_github_personal_access_token
   ```

## Usage with Claude Desktop

To use this with Claude Desktop, add the following to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "performance-audit": {
      "command": "node",
      "args": ["C:/path/to/performance-audit-mcp/build/index.js"],
      "env": {
        "GITHUB_TOKEN": "your_github_token_here"
      }
    }
  }
}
```

*Note: Replace `C:/path/to/` with the actual absolute path to your project folder.*

## Core Web Vitals Tracked

- **LCP** (Largest Contentful Paint): Measuring loading performance.
- **CLS** (Cumulative Layout Shift): Measuring visual stability.
- **FID/TBT** (Total Blocking Time): Measuring interactivity.
- **INP** (Interaction to Next Paint): Measuring responsiveness.
- **FCP** (First Contentful Paint): Measuring perceived load speed.
- **TTFB** (Time to First Byte): Measuring server responsiveness.

## License

MIT
