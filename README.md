# kirby-scheduled-content

*This skill is part of the [Kirby Skills Collection](https://github.com/markkirby125/kirby-skills-collection).*

An architectural pattern and test matrix for securely gating and scheduling static content publishing at the edge.

This skill equips AI agents with the blueprints necessary to build resilient edge-worker logic (e.g., Cloudflare Workers, Vercel Edge Middleware) that strictly hides pre-published static content from users, APIs, crawlers, and sitemaps until a zoned ISO 8601 `publishAt` timestamp is reached.

## 🪄 The Magic Prompt

Copy and paste this directly to your AI (Cursor, Windsurf, Claude Code, Antigravity):

```markdown
@agent Please install the kirby-scheduled-content skill into this workspace.
1. Read the `SKILL.md` file from this repository: https://github.com/markkirby125/kirby-scheduled-content
2. Identify the correct rules system for our current environment (e.g., `.cursor/rules/` for Cursor, `.windsurfrules` for Windsurf, `.clinerules` for Cline, or `~/.agents/skills/` for Antigravity).
3. Save the contents appropriately.
4. Confirm when the installation is complete.
```

## Manual Installation

- **Cursor**: Save the contents of `SKILL.md` to `.cursor/rules/kirby-scheduled-content.mdc`
- **Windsurf**: Save the contents of `SKILL.md` to `.windsurfrules`
- **Antigravity**: Clone this repository directly into `~/.agents/skills/kirby-scheduled-content`

## Tech Stack

- **Format**: Markdown / YAML
- **Compatibility**: Antigravity, Claude Code, Cursor, Windsurf, Cline

## External Resources & Authority Links

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Vercel Edge Middleware Patterns](https://vercel.com/docs/functions/edge-middleware)
- [Google Search Central: HTTP Status Codes & Network Errors](https://developers.google.com/search/docs/crawling-indexing/http-network-errors)
- [MDN Web Docs: X-Robots-Tag HTTP Header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Robots-Tag)
