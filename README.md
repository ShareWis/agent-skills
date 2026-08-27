# WisdomBase Agent Skills

This repository contains AI agent skills that are either self-developed or forked from open-source collections and customized for WisdomBase (ShareWis) workflows.

Skills are organized under `skills/<skill-name>/` and typically include:

- `SKILL.md`: main skill definition, usage trigger, and workflow
- `rules/*.md`: detailed best-practice or review rules used by the skill (where applicable)

## Current Skills

| Skill | Origin | What It Is | Path |
| --- | --- | --- | --- |
| `owasp-security-check` | Self-developed | Security audit guidelines for web apps and REST APIs based on OWASP Top 10 and common web security practices. Useful for vulnerability reviews, auth/authz checks, API audits, and pre-production security checks. | `skills/owasp-security-check/SKILL.md` |
| `ruby-on-rails-best-practices` | Self-developed | Ruby on Rails architecture and coding patterns inspired by Basecamp-style conventions. Useful when writing, reviewing, or refactoring Rails models, controllers, jobs, concerns, and Turbo/Hotwire features. | `skills/ruby-on-rails-best-practices/SKILL.md` |
| `coding-standards` | Forked & customized from [everything-claude-code](https://github.com/affaan-m/everything-claude-code) | Universal coding standards: naming, immutability, error handling, async patterns, API design, and testing. Code examples use TypeScript/JavaScript but the principles apply to any stack. | `skills/coding-standards/SKILL.md` |
| `backend-patterns` | Forked & customized from [everything-claude-code](https://github.com/affaan-m/everything-claude-code) | Backend architecture patterns: repository/service layers, N+1 prevention, caching, rate limiting, auth, background jobs, and structured logging. Code examples use TypeScript/Node.js but the patterns are language-agnostic. | `skills/backend-patterns/SKILL.md` |
| `ephemeral-e2e-tests` | Self-developed | How to write, run and prove Playwright E2E specs against the persistent `e2e` ephemeral environment behind the manually-triggered E2E release gate (SWWB-24455). Spans `ShareWis/wisdombase-playwright-tests` (the spec) and `ShareWis/sharewis-act` (the fixture seeder). | `skills/ephemeral-e2e-tests/SKILL.md` |

## Repository Structure

```text
sharewis-agent-skills/
  skills/
    owasp-security-check/
      SKILL.md
      rules/
    ruby-on-rails-best-practices/
      SKILL.md
      rules/
    coding-standards/
      SKILL.md
    backend-patterns/
      SKILL.md
    ephemeral-e2e-tests/
      SKILL.md
      rules/
```

## Purpose

The goal of this project is to provide reusable, opinionated skill packs that help AI agents produce more consistent, secure, and maintainable engineering outcomes inside WisdomBase workflows.
