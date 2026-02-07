# NavCLI - Goals & Vision

## Core Goal

**Enable AI Agents to browse the web like humans.**

Existing solutions (HTTP APIs, headless browser scripts, Playwright MCP) have limitations:
- No support for JS-rendered SPAs
- No session persistence
- Lack of interactive exploration

NavCLI's positioning: **An interactive, explorable browser CLI**

## Core Value

| Feature | What NavCLI Solves |
|---------|-------------------|
| JS Rendering | Full SPA support |
| Session Persistence | Cookies, session maintained |
| Interactive CLI | Agent can explore while operating |
| Token Optimization | Lightweight elements + on-demand text/html |

## Typical Workflow

```bash
> g https://example.com          # Navigate
> elements                       # Observe interactive elements
> c .btn-login                   # Click login
> t #email "test@example.com"    # Type email
> t #password "123456"           # Type password
> c button[type="submit"]       # Submit
> text                           # Confirm result
```

Agent can: **Navigate → Observe → Interact → Feedback → Continue**

## Vision

Become the **standard browser interaction layer** for AI Agents, enabling any Agent to control browsers via command-line interface:
- Form filling, login authentication
- Information scraping, content exploration
- Complex multi-step business processes

## Related Documentation

- [Skill File Download](https://github.com/wumu2013/navcli/blob/main/docs/skill.md)
- [Project Homepage](https://make.datavoid.fun/navcli/)
- [PRD Product Requirements](./NAVCLI_PRD.md)
