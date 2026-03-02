# WisdomBase Agent Skills

This repository contains customized and self-developed AI agent skills for WisdomBase (ShareWis).

Skills are organized under `skills/<skill-name>/` and typically include:

- `SKILL.md`: main skill definition, usage trigger, and workflow
- `rules/*.md`: detailed best-practice or review rules used by the skill

## Current Skills

| Skill | What It Is | Path |
| --- | --- | --- |
| `owasp-security-check` | Security audit guidelines for web apps and REST APIs based on OWASP Top 10 and common web security practices. Useful for vulnerability reviews, auth/authz checks, API audits, and pre-production security checks. | `skills/owasp-security-check/SKILL.md` |
| `ruby-on-rails-best-practices` | Ruby on Rails architecture and coding patterns inspired by Basecamp-style conventions. Useful when writing, reviewing, or refactoring Rails models, controllers, jobs, concerns, and Turbo/Hotwire features. | `skills/ruby-on-rails-best-practices/SKILL.md` |

## Repository Structure

```text
wisdombase-agent-skills/
  skills/
    owasp-security-check/
      SKILL.md
      rules/
    ruby-on-rails-best-practices/
      SKILL.md
      rules/
```

## Purpose

The goal of this project is to provide reusable, opinionated skill packs that help AI agents produce more consistent, secure, and maintainable engineering outcomes inside WisdomBase workflows.
