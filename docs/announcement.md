TITLE: [FEATURE] Superpowers bundle — agents can't skip design, TDD, or review

---

Jesse Vincent's Superpowers methodology (43k+ stars) now ships as an Amplifier bundle with structural enforcement — tool policies make agents physically unable to skip design, bypass TDD, or dodge code review.

What changed:
• New bundle at microsoft/amplifier-bundle-superpowers with 6 workflow modes (/brainstorm, /write-plan, /execute-plan, /debug, /verify, /finish), 5 specialist agents, and 7 declarative recipes including a full-development-cycle meta-recipe with approval gates
• Write tools are blocked during design phases at the platform level — agents must delegate to specialist sub-agents for all code changes. This is structural enforcement, not instructional prompting
• Composable behavior lets any existing bundle adopt just the methodology and agents without the full superpowers stack
• All 14 upstream Superpowers skills auto-fetched from obra/superpowers via remote git source, always pulling latest

Try it: amplifier bundle add git+https://github.com/microsoft/amplifier-bundle-superpowers@main --name superpowers

More info:
• Repo: github.com/microsoft/amplifier-bundle-superpowers
• Upstream: github.com/obra/superpowers (Jesse Vincent's original)
• Docs: docs/USAGE_GUIDE.md in the repo (getting started, all 7 recipes, composability patterns)
