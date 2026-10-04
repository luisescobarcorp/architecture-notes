# Architecture note: a real-time trading platform and governed MCP servers

Luis Escobar · Miami, FL · luisescobarcorp@gmail.com · [linkedin.com/in/luesco](https://linkedin.com/in/luesco)

This page backs the two claims on my résumé that are hardest to take on faith: a private trading platform I designed and built alone, and the production MCP servers I built at ENNU Life. The trading repository is private because it trades my own account. I walk through it on screen in interviews. Nothing here reports results or balances; it is about how the systems are built.

## 1. The trading platform (private, self-funded, 2026)

**Shape.** One Node/TypeScript process, a React client, shared wire types and an `mcp/` package, with Python operations scripts. 1,357 tracked paths, about 442,000 lines. A written contract sits at the top of 847 of 1,345 source files stating what the file is for, so a reader or an agent does not reconstruct intent from the implementation.

**API and real-time layer.** 254 REST routes under `/api` on Express. A Server-Sent Events hub replaced about 40 polling hooks for the eight topics that change often. WebSocket clients hold live tapes from a prediction-market venue and several crypto exchanges. Server and client share pinned wire contracts, and a test fails when either side drifts.

**Money-safe design.** Every order carries an idempotent `client_order_id` derived from the bet's identity, so a retry cannot double-buy. An in-flight guard blocks overlapping placements. Risk guardrails (per-order and daily limits, max price, open-order cap, balance floor, loss breaker) run server-side in one function shared by preview and placement, so the preview can never disagree with the gate. Stores write atomically (write, then rename) after a torn write once destroyed about fifteen days of history. Corrupt stores are quarantined, never silently reset. Anything that grants authority fails closed.

**Storage.** The trade ledger began as an 80 MB JSON file rewritten every one to five seconds, about 68 GB an hour of disk writes and three quarters of a core. It moved to row-level SQLite writes. JSON ledgers remain only where append-only is the right shape, with versioned S3 backups of both.

**Authorization and audit.** A bearer gate with constant-time compare on every API request. On the MCP server, an optional OAuth 2.1 authorization server with PKCE and dynamic client registration, so hosted AI clients can connect as custom connectors. Role leases (observer, risk-officer, trader, admin): a tool a role may not use is not even advertised to it. An append-only audit of every mutating call. A failed-auth throttle designed so it cannot be used to lock an operator out.

**Testing.** About 6,900 Vitest tests across hundreds of files. Fixtures are built from real settled market windows. Mutation passes break one clause at a time and re-run the suite to find clauses no test protects. Integration tests boot Express on a real port and read SSE frames. Source scans prove that a module which must never place orders does not import the placement path. A test fails if any store module forgets its test-mode in-memory guard.

**Operations.** A GitHub Actions gate (full server suite, both `tsc` configs, client build) before any push. systemd services on an AWS Graviton instance reached only through SSM, with a timer-driven autodeploy. A parity script hashes every tracked file on the box against the pushed commit. An off-box Lambda watchdog runs under a least-privilege IAM policy. A CI check fails when the executor's behavioural reference document drifts from the module it describes, because that document went stale twice.

**How AI is used, and how it is kept from authority.** Built spec-first with Claude Code and Cursor across twelve agent worktrees. Models in the product arrive display-and-grade first: a nightly review proposes changes and never applies them; forecasts are shown and graded until their record earns influence; in-loop agents hold zero authority behind a fail-closed control file. The only way a model-proposed order is placed is a push-to-phone approval with a one-time token. Two rules carried over from this work: a number that was not measured is never shown as zero, and no component gets authority over data or money until its own graded record has earned it.

## 2. MCP servers in production at ENNU Life (2025–2026)

**HQ Dashboard, 37 tools.** The executive operating dashboard (React 19 / TypeScript on Cloudflare Pages, a Cloudflare Worker API) joins OpenDental, MindBody, HubSpot, Meta, Google Ads, GA4, Search Console, Google Business Profile, Stripe, Authorize.net, WooCommerce, Paubox and Microsoft Teams. Its `POST /mcp` endpoint exposes 37 read-only tools over that operating data to Claude and other agents, behind OAuth or scoped service tokens. Staff sign in with Microsoft accounts (MSAL); the API validates the Entra ID token on every request, with an email allow-list and admin/member roles. The honest-data rule applies throughout: a value that was not measured renders as a dash, never as zero.

**HubSpot MCP server, 51 tools.** Sits alongside the WordPress monorepo and gives agents governed access to a 69,000-contact CRM instance with 300+ synced fields and 42+ field mappings to the clinical and commerce systems.

**Rules I hold MCP servers to.** Tool descriptions are an interface contract and are written as carefully as code. Arguments have tight schemas. Read-only by default; writes go through the same endpoints and guardrails the UI uses, never a second path. The server, not the model, enforces authorization. Every mutating call is audited.

## 3. Context: the estate these sit in

Twenty-seven repositories across two GitHub organizations run ENNU's digital side. The largest, `ennueco`, is a WordPress monorepo of about 45,000 PHP files and 113 custom plugins behind the consumer sites: assessments, biomarker scoring, LabCorp PDF parsing, member onboarding, e-signature, HubSpot integration, memberships, affiliates, booking and payments, under 13 GitHub Actions workflows (lint, phpcs, PHPUnit, quality gates). Its deploy workflow backs up the live plugin before writing, verifies the version after, and purges cache, because production once ran code that existed nowhere in git.

---

*Facts above are drawn from the repositories themselves. The trading platform makes no claim about results and is not a business. Everything else is production work I led or built. Happy to screen-share any of it.*
