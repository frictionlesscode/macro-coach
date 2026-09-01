# Security policy

## Reporting a vulnerability

Please don't open a public issue for anything security-sensitive. Use GitHub's private
vulnerability reporting instead — the repository's **Security** tab → **Report a
vulnerability**.

## Scope

This repository is a **Claude Skill**: Markdown documentation (`SKILL.md`), evaluation cases
(`evals/`), and `package.py`, a standard-library-only script that validates and zips
`SKILL.md` into a `.skill` bundle. It ships no server, holds no credentials, and makes no
network calls.

Anything involving stored data, credentials, or account access belongs to
[macro-mcp](https://github.com/frictionlesscode/macro-mcp), which has its own security
policy.

## No warranty

Provided "as is", without warranty of any kind — see [LICENSE](LICENSE).
