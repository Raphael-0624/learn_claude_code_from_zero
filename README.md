# Learn Claude Code from Zero

> A comprehensive, source-level deep dive into Anthropic's Claude Code — from execution engine internals to production-ready workflows.

[![Built with GitHub Pages](https://img.shields.io/badge/Built%20with-GitHub%20Pages-232323?style=flat-square&logo=github)](https://pages.github.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue?style=flat-square)](LICENSE)
[![Status: Actively Maintained](https://img.shields.io/badge/Status-Actively%20Maintained-3fb950?style=flat-square)](https://github.com/Rockelbel/learn_claude_code_from_zero)

---

## What Is This?

When the Claude Code source code leaked in March 2026, the community gained an unprecedented view into how a production AI coding agent actually works internally. This project is a systematic, structured analysis of that codebase — combined with official documentation — to produce the most complete technical reference available.

This is **not** a surface-level overview. Every module has:

- A **visual HTML report** — interactive charts, architecture diagrams, and call graphs
- A **Markdown source document** — the full analysis with code references and citations
- A **live demo page** — see the concepts in action without running anything

---

## Module Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Claude Code                              │
├─────────────┬─────────────┬──────────────┬─────────────────┤
│   Harness   │   Context   │   Memory     │   Tool System   │
│  Execution  │  200K ctx    │  4-layer     │   ~40 built-in  │
│   Engine    │  compression │  记忆架构     │   + MCP protocol│
├─────────────┼─────────────┼──────────────┼─────────────────┤
│  Agent Loop │Multi-Agent  │  Prompt Eng  │   Cases & Compare│
│  ReAct+Plan │ 2.5-3x speed│  实战技巧    │   vs Cursor/GHCP│
└─────────────┴─────────────┴──────────────┴─────────────────┘
```

### Core Engine Modules

| Module | Key Topics |
|--------|-----------|
| **🔧 Harness** | Sandbox execution, 3-tier permission model, PTC invocation, session state |
| **🧠 Context** | 200K token working memory, 3-level compaction, U-shaped attention, context rotting |
| **💾 Memory** | CLAUDE.md, MEMORY.md, Auto-Dream, 4-layer cross-session learning |
| **🛠️ Tool System** | ~40 built-in tools, MCP protocol, Computer Use, 6000+ external apps |
| **🔄 Agent Loop** | ReAct pattern, Plan Mode, self-healing loops, checkpoint & crash recovery |
| **🤝 Multi-Agent** | SubAgent isolation, Coordinator pattern, Git Worktree parallelism |

### Usage & Guides

| Module | Key Topics |
|--------|-----------|
| **📖 User Manual** | Full CLI reference, 19 chapters, all permission modes, CI/CD integration |
| **✍️ Prompt Guide** | Few-shot patterns, chain-of-thought, constraint prompting, meta-prompting |
| **🚀 Cases** | Real-world workflows — TDD, refactoring, code review, architecture docs |
| **⚖️ Comparison** | Claude Code vs Cursor, Copilot, GitHub Copilot Chat — capability matrix |

### Deep Analysis

| Module | Key Topics |
|--------|-----------|
| **🏗️ Report** | System architecture decomposition, data flow diagrams, design patterns |

---

## Quick Start

```bash
# Clone the repository
git clone https://github.com/Rockelbel/learn_claude_code_from_zero.git
cd learn_claude_code_from_zero

# Open the portal (any static server works)
python -m http.server 8080
# → http://localhost:8080

# Or open directly in browser
open index.html
```

No build step. No dependencies. Pure static HTML/CSS/JS.

---

## How This Was Built

- **Source**: March 2026 Claude Code source leak + official Anthropic documentation
- **Charts**: Chart.js 4.4.0 (UMD, CDN)
- **Icons**: Native emoji (no icon font dependencies)
- **Fonts**: System font stack (no external font requests)
- **Hosting**: GitHub Pages (or open `index.html` directly)

---

## Contributing

Corrections, expanded analysis, and new case studies are welcome. Open an issue or submit a PR.

---

## License

MIT — free to use, modify, and distribute.
