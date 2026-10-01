# OTPBox Agent Skills

[OTPBox](https://otpbox.io) gives a coding or testing agent a real disposable email
inbox to drive signup, OTP, magic-link, and email-verification flows end-to-end — no
human relaying a code.

This repo holds the official OTPBox [Agent Skill](https://otpbox.io/skills): packaged
instructions that teach an agent the right workflow for email-verification testing —
mint a key, create an inbox, block for the message, extract the code or link, clean up
— plus the gotchas to avoid (reusing an inbox across runs, polling instead of blocking,
burning quota on unnecessary reads).

## Install

```bash
npx skills add harshilshah1310/otpbox-skills
```

This works with Claude Code and any other agent that reads Skills from a local
`SKILL.md`-in-a-folder convention. See the [skills CLI](https://www.npmjs.com/package/skills)
for agent-specific install targets (`--agent claude-code`, etc.).

### Manual install

If you'd rather not run a third-party installer, copy the file directly:

```bash
mkdir -p .claude/skills/otpbox
curl -o .claude/skills/otpbox/SKILL.md https://raw.githubusercontent.com/harshilshah1310/otpbox-skills/main/skills/otpbox/SKILL.md
```

## What's in it

- When to reach for OTPBox (signup, login, 2FA, magic-link, QA-data-generation tasks)
- The core loop: create an inbox, wait for the message, extract the code/link, clean up
- Both call paths — MCP tools (13 of them, at `https://otpbox.io/mcp`) and plain REST
- Test identities — realistic personas (name, email, country), single or in bulk
- Gotchas — quota accounting, the MCP rate limit, the `wait_for_email` timeout cap, and
  why sandbox keys never fire webhooks

## Links

- [otpbox.io](https://otpbox.io) — the product
- [otpbox.io/docs](https://otpbox.io/docs) — full API/MCP reference
- [otpbox.io/ai](https://otpbox.io/ai) — MCP server setup for 12+ AI coding agents
- [otpbox.io/guides](https://otpbox.io/guides) — framework-specific walkthroughs (Playwright, Cypress, Selenium, Puppeteer, Postman)

## License

MIT — see [LICENSE](LICENSE).
