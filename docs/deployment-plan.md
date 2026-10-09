# Deployment Plan: Super Productivity for a 20-Person Company

**Status:** Proposal for approval. Nothing has been deployed, no code has been changed, and no costs have been incurred.
**Prepared:** 2026-10-09
**Repository evaluated:** `Advanced-Geosciences-Inc/super-productivity` (fork of `super-productivity/super-productivity`), commit `baa8dab27`, app version 19.1.0
**Target infrastructure:** Cloudflare, Fly.io, Supabase, plus the existing Python AI email assistant (Grok)

---

## How to read this document

Every substantive claim is labelled:

- **[V] Verified.** Checked in this repository's code or docs, with a `path:line` reference. The repo is the source of truth for what the app does today.
- **[A] Assumption.** An external fact, such as a vendor price or third-party product behaviour, or an engineering estimate. These come from public sources or judgement and **must be confirmed** before we commit money or design to them. Prices were gathered from third-party pricing summaries in October 2026; vendor pricing pages were not reachable directly.
- **[I] Inference.** A conclusion drawn from verified facts that has not been tested by running the system.

---

## 1. Executive summary

**Verdict: Super Productivity (SP) is suitable as each employee's _personal_ task and time manager. It is not suitable, on its own, as the company's _shared, permissioned_ project system.** The recommended design keeps SP unmodified, pairs it with a team tracker that owns roles and permissions, and connects the AI email assistant to that tracker.

Key findings:

1. **There are no users, roles, permissions or shared workspaces anywhere in SP.**
   - Tasks have no owner or assignee field [V `src/app/features/tasks/task.model.ts:82-157`].
   - The sync server's `User` table holds email and auth fields only, with no org, team or role [V `packages/super-sync-server/prisma/schema.prisma:13-46`].
   - The project's stated principle is _"A personal deep-work tool, not team management or reporting"_ [V `AGENTS.md:13`].
2. **Sharing one SP dataset between employees is unsafe.**
   - The design assumes one person's devices: the vector clock is capped at 20 entries, with the comment "extremely unlikely for a personal productivity app" [V `packages/sync-core/src/vector-clock.ts:40-48`].
   - Encryption is a single shared password, so everyone on the dataset would see everything [V `docs/sync-and-op-log/supersync-encryption-architecture.md`].
   - Settings would sync and overwrite each other as well [I].
3. **Building multi-user features into the fork is not recommended.**
   - Upstream is very active: 2,585 commits in the last 6 months [V `git log`], and `src/app` churned about +380k/−115k lines against about 249k non-spec lines [V, measured by research agent].
   - Multi-user support would cut through the high-risk sync layer [V `AGENTS.md` "Sync-correctness rules"], and upstream has explicitly ruled the feature out. Rebasing the fork would become a permanent, expensive job [I].
4. **SP already integrates with team trackers that do have roles and permissions.** OpenProject is a built-in provider with status sync, worklog push, and backlog import of work packages [V `src/app/features/issue/providers/open-project/open-project-common-interfaces.service.ts:71-87`, `docs/wiki/3.07-Issue-Integration-Comparison.md:72-84`].
5. **The self-hostable SuperSync server fits Fly.io + Supabase**, with constraints:
   - single instance only [V `packages/super-sync-server/helm/supersync/templates/deployment.yaml:1-3`]
   - PostgreSQL 14+, with 16+ recommended [V `packages/super-sync-server/README.md:254-268`]
   - passwordless auth only (passkeys and magic links), no SSO [V `packages/super-sync-server/src/api.ts:149-621`]
   - mandatory end-to-end encryption [V `packages/sync-providers/src/super-sync/super-sync.ts:154`]
6. **There is no server-side task-creation API** in SP or SuperSync [V]. The AI assistant should create tasks in the team tracker's API, and SP imports them. The assistant should not write into SP directly.
7. **Estimated operating cost:** about **$85–$110/month** for infrastructure, plus **$20–$200/month** for the Grok API depending on model and email volume. That makes about **$105–$310/month in total**, or roughly **$5–$15 per employee** [A, see §9].

**Decision requested:** approve the recommended architecture (§5) and Phase 0 (§11). Phase 0 resolves the open assumptions before any spending.

---

## 2. Scope and method

Reviewed:

- the repository structure and the fork's status
- licensing and dependencies
- documentation (`AGENTS.md`, `ARCHITECTURE-DECISIONS.md`, `docs/`, the wiki and the sync docs)
- the data model
- the sync providers and the SuperSync server (stack, schema, auth, API, deployment, backups, monitoring)
- the Electron security settings
- the local REST API and the plugin API
- the issue-provider integrations

Public sources were used for infrastructure and AI pricing and for third-party product requirements.

Not done, on purpose:

- no builds, tests, deployments or account sign-ups
- no load testing
- no penetration testing

The security observations below come from reading the code. They are not the result of active testing.

---

## 3. Repository facts

### 3.1 Fork status

| Item                     | Finding                                                                                                                                                          |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Remotes                  | `origin` = `Advanced-Geosciences-Inc/super-productivity`, `upstream` = `super-productivity/super-productivity` [V]                                               |
| Company-specific changes | **None.** No commits by company authors; the fork is a mirror of upstream `master` [V `git log`]                                                                 |
| Version                  | 19.1.0 [V `package.json`]                                                                                                                                        |
| Upstream activity        | 2,585 commits in the last 6 months, 171 distinct authors in that window. Dominated by one maintainer (≈1,980 commits) [V `git log`]. Releases roughly weekly [V] |
| Stack                    | Angular 21, NgRx 21, Electron 43.5, Capacitor 8, TypeScript 5.9 [V `package.json`]                                                                               |
| Size                     | about 2,255 `.ts` files in `src/app` (893 specs), about 249k non-spec lines [V]                                                                                  |

**Implication:** the fork is free to stay a mirror. Nothing in the recommended plan requires changing SP's code. Keep company code (the AI assistant, deployment config) in a **separate repository** so the fork never has rebase conflicts [I].

### 3.2 Licensing

- **SP and the SuperSync server are MIT** [V `LICENSE`, `package.json:17`]. The server package has no separate license field, so it falls under the root MIT [V/I].
- **The SuperSync server's runtime dependencies are permissive** [V `package-lock.json`, via research agent]:
  - MIT: Fastify, `@simplewebauthn`, `jsonwebtoken`, `zod`, `@noble/ciphers`
  - Apache-2.0: Prisma
  - BSD: `bcryptjs`, `dotenv`
  - MIT-0: `nodemailer`
  - No copyleft dependencies.
- **OpenProject (recommended team tracker) is GPLv3** [A]. Running it unmodified for internal use creates no obligations for our code. Modifying and distributing it would [A, confirm with counsel if we ever modify it].
- **Upstream branding:** if we ever distribute our own SP builds, we should not present them as the official app. The recommended plan uses the official signed builds, so this does not arise [I].

### 3.3 Documentation quality

The documentation is extensive and current: architecture decisions, sync contracts, a server README, and auth, backup and capacity docs [V `docs/`, `packages/super-sync-server/docs/`]. Two small inconsistencies were found, both minor:

- The integration matrix shows OpenProject auto-import as "—" while the code implements it [V `docs/wiki/3.07-Issue-Integration-Comparison.md:17` vs. `open-project-common-interfaces.service.ts:71`].
- A plugin-API comment says node execution is "built-in only", which is outdated [V, via research agent: `packages/plugin-api/src/types.ts:792` vs. `docs/plugin-development.md:655-685`].

---

## 4. Capability assessment against requirements

| Requirement                        | SP alone                                                                                              | Evidence                                                                                             |
| ---------------------------------- | ----------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| Personal tasks per employee        | **Strong.** Tasks, projects, tags, planner, recurring tasks, time tracking, focus mode, offline-first | [V `docs/repository-map.md`]                                                                         |
| Shared company projects            | **Not supported** natively. Only through an integration (OpenProject, Jira, Plainspace, and others)   | [V §1 item 1, `src/app/features/issue/issue.model.ts:38-64`]                                         |
| Roles and permissions              | **None**                                                                                              | [V `prisma/schema.prisma:13-46`, task model]                                                         |
| Company SSO (Google/Microsoft)     | **No.** SuperSync supports passkeys and email magic links only; there is no OIDC/SAML                 | [V `packages/super-sync-server/src/api.ts:149-621`]                                                  |
| Restrict sign-ups to company staff | **Yes.** `ALLOWED_EMAILS` supports `*@company.com`                                                    | [V `packages/super-sync-server/src/email-allowlist.ts:1-29`]                                         |
| Server API to create tasks         | **None.** Only the encrypted op-log sync protocol, which is internal and versioned                    | [V `packages/shared-schema/src/supersync-http-contract.ts:118-138`, `super-sync.ts:154`]             |
| Local API to create tasks          | **Desktop only,** on `127.0.0.1:3876`, off by default, bearer-token protected                         | [V `electron/shared-with-frontend/local-rest-api.model.ts:1-2`, `default-global-config.const.ts:56`] |
| Encryption                         | **Mandatory client-side E2EE** (Argon2id + AES-256-GCM); the server cannot read task content          | [V `sync.routes.payload.ts:47-48`, `packages/sync-core/src/encryption/`]                             |
| Platforms                          | Windows, macOS, Linux, Android, iOS, web/PWA                                                          | [V `README.md`, `android/`, `ios/`]                                                                  |
| Email features                     | **None.** Drafting replies is outside SP's scope                                                      | [V]                                                                                                  |

---

## 5. Recommended architecture

### 5.1 Overview

```
                         ┌─────────────────── Cloudflare ───────────────────┐
                         │ DNS + proxy (TLS) · Zero Trust Access (admin UIs)│
                         │ Pages: optional company-hosted SP web/PWA build  │
                         └───────┬───────────────────┬──────────────────────┘
                                 │                   │
           ┌─────────────────────▼──┐      ┌─────────▼──────────────┐
 Employees │ SuperSync server       │      │ OpenProject (team      │
 (SP app on│ Fly.io, 1 machine      │      │ tracker) Fly.io,       │
 desktop / │ personal, E2E-encrypted│      │ 1 machine + volume     │
 mobile /  │ sync per employee      │      │ roles · permissions ·  │
 web)      └──────────┬─────────────┘      │ shared projects · API  │
   │  ▲               │                    └──────┬───────▲─────────┘
   │  └── personal ───┘                           │       │ REST API v3
   │      task sync                               │       │ (create work
   └──── OpenProject issue provider ──────────────┘       │  packages)
         (import assigned work, push status/time)         │
                                                 ┌────────┴────────────┐
           ┌──────── Supabase (Postgres) ──────┐ │ AI email assistant  │
           │ db: supersync   (E2E ciphertext)  │ │ (Python, Fly.io)    │
           │ db: openproject (company data)    │ │ Gmail API → Grok →  │
           │ schema: assistant (dedupe, audit) │◄┤ tasks + Gmail drafts│
           └───────────────────────────────────┘ └─────────────────────┘
```

### 5.2 Components and responsibilities

| Component                                                  | Role                                                                                                                                    | Why                                                                                                                                                                                                                 |
| ---------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Super Productivity**, official signed builds, unmodified | Each employee's personal planner: daily planning, time tracking, focus, personal tasks                                                  | Its strength [V §4]. Official builds avoid our own code signing and update distribution. There is no auto-updater in-app [V `electron/start-app.ts:526-538`]; store and package-manager channels handle updates [A] |
| **SuperSync server**, self-hosted on Fly.io                | Syncs each employee's _personal_ data across their own devices. One account per employee                                                | MIT, Postgres-backed, E2E-encrypted, domain allowlist [V]. The company cannot read personal data, which is a privacy feature and is also stated in policy (§7)                                                      |
| **OpenProject Community**, self-hosted on Fly.io           | System of record for **shared company projects**: membership, roles, permissions, assignment, audit                                     | Built-in SP provider with status, worklog and backlog import [V]. Mature RBAC and a documented REST API v3 [A]                                                                                                      |
| **Supabase Postgres**                                      | Managed database for SuperSync, OpenProject and the assistant's state                                                                   | Existing vendor. Managed backups [A]                                                                                                                                                                                |
| **Cloudflare**                                             | DNS, TLS proxy, Zero Trust Access in front of admin and web UIs, optional Pages hosting for the SP web build, R2 for off-site backups   | Existing vendor. Access is free up to 50 users [A]                                                                                                                                                                  |
| **AI email assistant** (existing Python, Grok)             | Reads Gmail, extracts action items, creates OpenProject work packages assigned to the right employee, and creates Gmail **drafts** only | Uses a stable, documented API (OpenProject) rather than SP's internal sync format [V §4]                                                                                                                            |

**Personal versus shared work.** Each employee gets a private OpenProject project ("Inbox – <name>") that only they and the assistant's service account can see. Email-derived personal action items land there. Shared work lives in company projects with role-based membership. SP imports both into the employee's backlog.

### 5.3 Alternatives considered

| Option                                                                            | Verdict                            | Reason                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------------------- | ---------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| A. SP alone, one shared sync account or folder                                    | **Rejected**                       | No permissions, a single encryption key, a 20-entry vector-clock cap, settings collisions [V §1]                                                                                                                                        |
| B. Fork SP to add users, roles and sharing                                        | **Rejected**                       | Touches the data model, op-log, sync-core, server and UI. Upstream churn and stated scope make the fork unmaintainable [V §3.1, I]                                                                                                      |
| **C. SP + self-hosted OpenProject**                                               | **Recommended**                    | Uses existing integrations; RBAC lives in a purpose-built tool; no SP code changes                                                                                                                                                      |
| D. SP + Plainspace (upstream's own collaboration layer)                           | **Evaluate in Phase 0**            | Two-way done sync, claim pool, "share project as Space" [V `docs/wiki/3.07-…:161-175`]. Hosted at plainspace.org. Its roles, pricing, data residency and self-hosting could **not** be verified [A]. Its server is not in this repo [V] |
| E. SP + custom Supabase "company tasks" backend + custom SP issue-provider plugin | **Fallback**                       | Best fit to our infrastructure: Supabase Auth with Google SSO, Row-Level Security for roles. But we would build and maintain a team task UI ourselves. Plugins are "not strongly sandboxed" [V `docs/plugin-development.md:120-125`]    |
| F. Replace SP with a single team-native tool (OpenProject alone, Plane, Vikunja)  | **Consider if SP adoption is low** | Simpler. Loses SP's personal planning and time-tracking strengths [I]                                                                                                                                                                   |

---

## 6. Detailed component findings

### 6.1 SuperSync server

| Area                    | Finding                                                                                                                                                                                                                                                                                                                               |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Stack                   | Node 24 (Alpine), Fastify 5, Prisma 5.22, Zod, JWT. Listens on port 1900 as non-root uid 1001 [V `packages/super-sync-server/Dockerfile:2,39,83-92`]                                                                                                                                                                                  |
| Database                | PostgreSQL. Minimum 14; 16+ is the supported version [V `README.md:254-268`]. Uses JSONB, GIN index on a `String[]`, `CREATE INDEX CONCURRENTLY` migrations, RepeatableRead transactions, `SELECT … FOR UPDATE` [V]. No extensions, no `LISTEN/NOTIFY`, no runtime advisory locks [V]                                                 |
| Migrations              | `scripts/migrate-deploy.sh` wraps `prisma migrate deploy` and takes advisory locks. Can run on start with `RUN_MIGRATIONS_ON_STARTUP=true` [V `scripts/migrate-deploy.sh:16-30,117-124`, `Dockerfile:83`]. **They need a direct or session-mode Postgres connection, not Supabase's transaction pooler** [I]                          |
| Scaling                 | **Single instance only.** The Helm chart refuses `replicaCount > 1` because WebSocket, auth-cache and WebAuthn-challenge state live in memory [V `helm/supersync/templates/deployment.yaml:1-3`]. Fine for 20 users; the upstream fleet runs about 8.6M ops / 6.8 GB [V `docs/production-capacity.md:42`]                             |
| Resources               | Container sized for 768 MB / 1 CPU; `--max-old-space-size=576` [V `Dockerfile:84-87`, `docker-compose.yml:130-131`]                                                                                                                                                                                                                   |
| Data per user           | 100 MB default quota [V `prisma/schema.prisma:32`]. Synced ops kept 45 days after a snapshot covers them [V `src/sync/sync.types.ts:529`]                                                                                                                                                                                             |
| Auth                    | Passkeys (WebAuthn) and email magic links (15 min). **JWTs live 365 days** [V `src/auth.ts:18`]. Revocation is per account (`tokenVersion`), not per device [V `src/auth.ts:140-166`]. A 30-second auth cache can delay revocation [V `src/auth-cache.ts`]                                                                            |
| Sign-up control         | `ALLOWED_EMAILS=*@company.com` [V]. **It is checked only on the registration routes** (`/register/passkey/*`, `/register/magic-link`) [V `src/api.ts:292,332,567`]. Removing someone from the allowlist does **not** block an existing account's login or sync; offboarding must delete the account (§7.4)                            |
| Email                   | Any SMTP relay (`SMTP_HOST/PORT/USER/PASS/FROM`). Without SMTP it falls back to a test mailbox, so real SMTP is required [V `src/config.ts:333-347`, `src/email.ts:30-39`]                                                                                                                                                            |
| Rate limits             | 500 requests / 15 min / IP globally; 50 / 15 min on auth routes [V `src/server.ts:416-421`, `src/api.ts`]. Depends on the correct client IP behind proxies: `TRUST_PROXY` defaults to loopback and unique-local addresses [V `src/config.ts:104`]. Must be verified behind Cloudflare and Fly so all users don't share one bucket [I] |
| Hardening               | Helmet with strict CSP, explicit CORS allowlist (wildcard refused in production), Zod validation, sanitised 5xx errors, 20 MB body cap, 8 KB WebSocket frames [V `src/server.ts:44-58,391-460`, `src/config.ts:288-296`]                                                                                                              |
| Server-visible metadata | Email, op/entity IDs and types, timestamps, vector clocks, device names and user agents. Task **content** is ciphertext [V `docs/architecture.md:160-172`]                                                                                                                                                                            |
| Observability           | Text logs plus JSON `AUDIT` lines; `/health` (checks the database) and `/live` endpoints [V `src/server.ts:464-489`]. **No Prometheus, Sentry or OpenTelemetry** [V]                                                                                                                                                                  |
| Backups                 | `scripts/backup.sh` (pg_dump with optional rclone). Recovery doc: restore accounts, clients re-sync [V `docs/backup-and-recovery.md:20-85`]                                                                                                                                                                                           |
| Tests                   | 74 test files plus 22 Postgres integration specs [V]                                                                                                                                                                                                                                                                                  |

### 6.2 SP client and desktop security

| Area                | Finding                                                                                                                                                                                                                                             |
| ------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Electron hardening  | `contextIsolation: true`, `nodeIntegration: false`, `webSecurity: true`, enforced by a guard; navigation and window-open guarded [V `electron/main-window.ts:209-220,643-656`, `electron/web-preferences-guard.ts:60-67`]                           |
| CSP                 | Permissive, as plugins require it: `unsafe-eval`, `unsafe-inline`, `connect-src *` [V `src/index.html:91-114`]. Electron also injects permissive CORS headers [V `electron/main-window.ts:300-310`]                                                 |
| Plugins             | Run in the main renderer via `new Function` and are **not sandboxed** [V `docs/plugin-development.md:120-125`, `src/app/plugins/plugin-runner.ts:173`]. Node execution needs a native consent dialog [V]. **Policy: only company-approved plugins** |
| Local REST API      | Off by default; binds to 127.0.0.1; checks the Host header (DNS-rebinding defence); rejects browser `Origin`s; bearer token stored in a 0600 file [V `electron/local-rest-api.ts:90,201-251,455-529,546-606`]                                       |
| Update check        | Daily banner checks upstream GitHub releases [V `src/app/core/update-check/update-check.service.ts:17-20`]. Appropriate, since we use upstream builds                                                                                               |
| Default sync server | `sync.super-productivity.com` [V `default-global-config.const.ts:257-260`]. Each employee changes it once under Sync → Advanced → Server URL [V `sync-form.const.ts:753-761`]. A company web build could preset it [I]                              |

### 6.3 OpenProject integration (SP side)

- **Connection:** host, API token and project per provider instance. SP imports up to the last 100 work packages for the configured project and scope (for example, assigned to me) [V `open-project-common-interfaces.service.ts:71-87`].
- **Two-way behaviour:** status transitions with configurable workflows, time-entry (worklog) push, start date to due date, progress percentage [V `docs/wiki/3.07-…:72-84`].
- **Limitations:**
  - One provider instance per OpenProject project. An employee in five shared projects configures five providers, unless subproject inclusion works [A, test in Phase 0].
  - No comments or subtasks [V].
  - Auto-import into the backlog is off by default per provider (`isAutoAddToBacklog: false`) [V `src/app/features/issue/issue.const.ts:122-123`].
  - The SP **web/PWA** client calls OpenProject cross-origin, so OpenProject must allow the PWA origin via CORS. Desktop is unaffected [V `main-window.ts:300-310`; A for OpenProject's CORS settings].

### 6.4 OpenProject itself [A, verify in Phase 0]

- **Requirements:** about 4 GB RAM, 4 cores and 20 GB disk for up to 200 users. PostgreSQL **16+** required from OpenProject 16.
- **Licence and features:** GPLv3. Project-level roles and permissions and REST API v3 with API tokens are in the Community edition.
- **SSO:** custom OIDC login may be **Enterprise-only**. If so, OpenProject accounts use local passwords plus 2FA, with Cloudflare Access adding Google SSO in front of the web UI. API paths need an Access bypass, because SP and the assistant authenticate with API tokens.

---

## 7. Security measures

### 7.1 Identity and access

1. **SuperSync:** set `ALLOWED_EMAILS=*@<company-domain>`. Prefer passkeys. Use the company SMTP relay so magic links reach only company mailboxes.
2. **OpenProject:** project-based roles; restrict administrator to 2 people. Personal "Inbox" projects visible only to the owner plus the assistant's service user. 2FA enforced [A, Community feature]. Cloudflare Access with Google Workspace login in front of the web UI; Access _bypass_ only for `/api/v3/*` (token-authenticated).
3. **AI assistant service account:** in OpenProject, may only create and update work packages in Inbox projects and in projects it is explicitly added to. No admin rights.
4. **Fly.io and Supabase:** organisation accounts with SSO or 2FA; at most 2 admins; secrets only in `fly secrets` or Supabase vault, never in git.

### 7.2 Network

- All public endpoints go through the Cloudflare proxy with TLS.
- Supabase accepts connections only from Fly (network restrictions) [A]. Use a separate database role per service, with no superuser for applications.
- SuperSync `CORS_ORIGINS` lists only the SP web origin(s) we use [V that wildcard is refused].
- `TRUST_PROXY` configured so the real client IP is used for rate limiting [V `config.ts:104`; I for the exact Fly/Cloudflare values].
- Fly's proxy and Cloudflare must allow WebSockets and long requests. Server timeouts are 80/90 s [V `server.ts:382-383`]; Cloudflare's proxy limit is about 100 s on standard plans [A].

### 7.3 Data protection

- **Personal SP data** is end-to-end encrypted; the company cannot read it [V]. State this in the staff policy. Employees must store their encryption password in the company password manager: **a lost password means lost synced data** [V `supersync-encryption-architecture.md`].
- **Company project data** lives in OpenProject: readable by admins, backed up, subject to company retention.
- **Email data sent to Grok:** see §8.5.

### 7.4 Joiners and leavers

- **Joiner:** the user self-registers on SuperSync (allowlisted domain); admin creates the OpenProject account and Inbox project; employee configures the SP sync URL and the OpenProject provider using a one-page guide.
- **Leaver:** this is critical because 365-day JWTs survive allowlist removal [V].
  1. Disable the Google Workspace account (blocks magic-link login).
  2. Run `npm run delete-user` (or `clear-data`) on SuperSync [V `packages/super-sync-server/scripts/delete-user.ts`]. This deletes the account and invalidates its tokens.
  3. Lock the OpenProject user and reassign their work packages.
  4. Remove them from Cloudflare Access.

  Passkey login does not depend on email, so **step 2 is mandatory** [I].

### 7.5 Supply chain and patching

- Use only official SP builds; disallow third-party plugins by policy [V that plugins are unsandboxed].
- Track GitHub security advisories for `super-productivity/super-productivity`. The project publishes GHSAs (`super-sync.ts:153` cites GHSA-9v8x-68pf-p5x7) [V].
- Update SuperSync monthly and immediately on security advisories. Pin image tags; never deploy `latest` blindly.

---

## 8. AI email assistant integration

### 8.1 Integration choice

| Path                                                   | Verdict         | Evidence                                                                                                                                                                                                                                 |
| ------------------------------------------------------ | --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **OpenProject REST API** → SP imports via its provider | **Recommended** | Stable public API; respects roles; works for all SP platforms [V §6.3; A for the API]                                                                                                                                                    |
| Custom SP plugin polling a Supabase queue              | Fallback        | Possible with `PluginAPI.request` (needs `http` permission and `allowedHosts`) and `addTask` [V `packages/plugin-api/src/types.ts:154-176,683,830-839`]. Only runs while the app is open; manual ZIP install per device; unsandboxed [V] |
| SP local REST API                                      | Not suitable    | Desktop only, localhost only [V §6.2]                                                                                                                                                                                                    |
| Inject ops into SuperSync                              | **Do not do**   | Requires each user's E2EE password and internal NgRx action names that change between releases [V `supersync-http-contract.ts:118-138`, `super-sync.ts:154`]                                                                             |
| JSON backup import                                     | Not viable      | Import replaces all data [V `src/app/op-log/backup/backup.service.ts:102-117`]                                                                                                                                                           |

### 8.2 Processing flow

1. **Ingest.** Gmail push via `users.watch` + Pub/Sub, or polling with `history.list`, for opted-in users [A]. Skip labels and senders on an exclusion list (HR, legal, medical, payroll).
2. **Deduplicate.** Key on the Gmail `Message-ID` / `threadId` in Supabase table `assistant.processed_messages`.
3. **Minimise.** Send Grok only the needed parts: latest message plus a trimmed thread, no attachments by default, signatures and quoted history stripped.
4. **Extract.** Call Grok with a **strict JSON schema**: `action_items[] {title, description, due_date?, owner_email, project_hint?, confidence}`, `needs_reply`, `draft_reply`.
5. **Validate.** Schema-validate the output, drop items below the confidence threshold, cap tasks per email (for example 3) and per user per day.
6. **Create task.** `POST /api/v3/work_packages` into the owner's Inbox project, or a shared project only when the sender and owner are both members. Include the subject, a summary, and a deep link to the Gmail thread. Never include full email bodies in shared projects.
7. **Create draft.** `users.drafts.create` in the same thread. **Never send.**
8. **Audit.** Write a row per action (message id, task id, draft id, model, token counts, decision) to `assistant.audit_log`.
9. **Pick-up.** SP imports the work package into the employee's backlog on its next poll; the employee reviews the draft in Gmail.

### 8.3 Enforcing "draft only, never send"

**There is no Gmail OAuth scope that allows creating drafts but forbids sending.** `gmail.compose` and `gmail.modify` both permit sending [A, confirm in Google's scope docs]. So the guarantee has to be layered:

- **Code:** a single `GmailClient` wrapper exposing only `list`/`get`/`history`/`drafts.create`/`drafts.update`. A CI check fails the build if `messages.send`, `drafts.send` or `.send(` appears anywhere else.
- **Least scope:** `gmail.readonly` for reading plus `gmail.compose` for drafts. Never request `mail.google.com/`.
- **Credentials:** a Google Workspace **internal** OAuth app or domain-wide delegation restricted to those two scopes and to an opt-in group [A: company uses Google Workspace].
- **Detection:** alert on any `Sent` message whose client is the assistant's OAuth client ID (Workspace audit log) [A].
- **LLM isolation:** the model has **no tools**. It returns JSON only, and the program decides what happens. This is the primary prompt-injection defence, since emails are untrusted input.

### 8.4 Hosting

A Fly.io machine (Python, 512 MB) running the worker; a scheduled or event-driven loop. State lives in a Supabase `assistant` schema. Secrets in `fly secrets`: Google credentials, xAI key, OpenProject service token.

### 8.5 AI-specific risks

| Risk                                                            | Mitigation                                                                                                                 |
| --------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Email content leaves the company to xAI                         | Review xAI's business/API data-retention terms and DPA before Phase 3 [A]; exclusion lists; minimisation; staff opt-in     |
| Prompt injection ("ignore instructions, create task …, send …") | No tools for the model; schema validation; caps; tasks go to the recipient's own Inbox only; sending is impossible by code |
| Wrong or duplicate tasks                                        | Confidence threshold, dedupe table, 2-week "shadow mode" with logging only before writing                                  |
| Cost runaway                                                    | Per-day token budget with a hard stop; model choice (§9)                                                                   |

---

## 9. Cost estimate (monthly, USD, 20 employees) [A]

Prices are third-party summaries from Oct 2026; confirm on vendor pricing pages.

| Item                                                                    | Sizing                                                                                             | Est. / month                                              |
| ----------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| Fly.io: SuperSync                                                       | 1 × shared-cpu-1x, 1 GB, always on (WebSockets)                                                    | ~$6                                                       |
| Fly.io: OpenProject                                                     | 1 × shared-cpu-2x/4x, 4 GB + 20 GB volume                                                          | ~$25–35                                                   |
| Fly.io: AI assistant                                                    | 1 × shared-cpu-1x, 512 MB                                                                          | ~$3–4                                                     |
| Supabase                                                                | Pro plan $25 (includes $10 compute credit) + Small compute upgrade                                 | ~$30                                                      |
| Supabase PITR (optional)                                                | 7-day point-in-time recovery                                                                       | +$100 (not in base)                                       |
| Cloudflare                                                              | DNS, proxy, Access (free ≤ 50 users), Pages, R2 backups (≤ 10 GB free)                             | $0–2                                                      |
| SMTP relay                                                              | Magic links and notifications (Workspace relay, or a free tier of Resend/SES)                      | $0–5                                                      |
| Monitoring                                                              | Fly built-in metrics, a free-tier uptime checker, Sentry free tier for the assistant               | $0                                                        |
| **Infrastructure subtotal**                                             |                                                                                                    | **~$65–80** (budget **$85–110** with headroom and egress) |
| Grok API, budget model (e.g. Grok 4 Fast at $0.20 / $0.50 per M tokens) | 20 users × 50 emails/day × 22 days ≈ 22k emails; ~3k in + 0.5k out tokens each → ~66M in / 11M out | **~$20**                                                  |
| Grok API, flagship model (~$2 / $6 per M tokens)                        | same volume                                                                                        | **~$200**                                                 |
| **Total**                                                               |                                                                                                    | **~$105–310 / month (~$5–15 per employee)**               |

**One-time effort** (internal engineering, not cash) [A]:

| Work                                     | Effort                  |
| ---------------------------------------- | ----------------------- |
| SuperSync deployment                     | 2–3 days                |
| OpenProject deployment and configuration | 3–5 days                |
| Assistant changes                        | 5–10 days               |
| Rollout, docs and training               | 3–5 days                |
| **Total**                                | **≈ 13–23 person-days** |

**Ongoing maintenance:** about 4–8 hours per month for updates, backup checks, joiners/leavers and monitoring [A].

For comparison, SaaS team trackers typically cost $8–15 per user per month, which is $160–300 for 20 users, before any AI costs [A].

---

## 10. Deployment steps (reference, not executed)

These are untested reference steps; validate each in Phase 1 against staging.

### 10.1 Supabase

1. Create one Pro project in the region nearest the team. Confirm the Postgres major version is **16 or later** (OpenProject needs it [A]; SuperSync needs 14+ [V]).
2. Create databases `supersync` and `openproject`, and schema `assistant` (in a separate database or in `supersync`'s server, isolated by role).
3. Create one login role per service, each owning only its database. Store the connection strings.
4. Use a **direct or session-mode (port 5432)** connection for SuperSync, including migrations, with `?connection_limit=5&pool_timeout=10` [V that the pool params are required when `REQUIRE_DATABASE_POOL_LIMITS=true`: `migrate-deploy.sh:136-152`]. Supabase direct hosts are IPv6; Fly supports IPv6 egress [A].
5. Turn on network restrictions and confirm the daily backup schedule.

### 10.2 SuperSync on Fly.io

1. Build from `packages/super-sync-server/Dockerfile` at a **pinned release commit**, ideally through a CI job in the company infra repository.
2. Create `fly.toml`:
   - `internal_port = 1900`
   - HTTP check on `/health`
   - `auto_stop_machines = "off"` and `min_machines_running = 1`
   - **exactly one machine**: never `fly scale count` > 1 [V single-instance constraint]
3. Set secrets with `fly secrets set`:
   - `DATABASE_URL`
   - `JWT_SECRET` (≥ 32 random characters [V `src/auth.ts:13`])
   - `PUBLIC_URL=https://sync.<domain>`
   - `WEBAUTHN_RP_ID=sync.<domain>`
   - `WEBAUTHN_ORIGIN=https://sync.<domain>`
   - `CORS_ORIGINS=<SP web origins>`
   - `ALLOWED_EMAILS=*@<domain>`
   - `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`
   - `RUN_MIGRATIONS_ON_STARTUP=true`
   - `TRUST_PROXY=<values verified for Fly + Cloudflare>`
4. Point Cloudflare DNS `sync.<domain>` (proxied) to the Fly app. **Do not** put Cloudflare Access in front of the sync API, because native clients cannot complete an Access login [I].
5. Smoke test:
   1. Register one account.
   2. Sync two devices.
   3. Check that a non-company email is refused.
   4. Restart the machine and confirm devices re-sync.

### 10.3 OpenProject on Fly.io

1. Deploy the official OpenProject container at a pinned version, with `DATABASE_URL` set to the Supabase `openproject` database and a Fly volume for attachments [A: confirm external-DB and attachment configuration in OpenProject's Docker docs].
2. Configure:
   - SMTP
   - 2FA
   - API token creation enabled
   - roles (Admin, Project Manager, Member, Viewer, Assistant-bot)
   - project templates
   - an "Inbox" template (private)
   - CORS for the SP web origin
3. Put Cloudflare Access (Google login) in front of `pm.<domain>`, with a bypass policy for `/api/v3/*`.

### 10.4 SP clients

1. Install official SP builds (Windows Store / winget, Mac App Store / DMG, Play Store, App Store) [A for store availability].
2. Settings → Sync: choose SuperSync, set **Server URL** to `https://sync.<domain>` [V that it is configurable], set an encryption password and store it in the company password manager.
3. Add the OpenProject provider per relevant project: host, personal API token, scope "assigned to me", auto-add to backlog **on**.

### 10.5 AI assistant

1. Create an internal Google OAuth app or domain-wide delegation limited to `gmail.readonly` + `gmail.compose`.
2. Deploy the worker to Fly with secrets.
3. Run **shadow mode** for 2 weeks: write the planned actions to `assistant.audit_log` only, create nothing.
4. Enable task creation, then draft creation, per pilot user.

### 10.6 Backups and monitoring

- **Supabase:** daily managed backups (Pro) [A]. Add a nightly `pg_dump` of both databases to Cloudflare R2 from a scheduled Fly machine, adapting `packages/super-sync-server/scripts/backup.sh` [V script exists].
- **Restore test:** restore into a scratch database **quarterly** and record the result.
- **OpenProject attachments:** volume snapshots (Fly keeps daily snapshots [A]) plus a weekly copy to R2.
- **Uptime:** checks on `https://sync.<domain>/health` and the OpenProject health endpoint, alerting to email or Slack.
- **Logs:** retained in Fly. Optionally ship them to a log service, alerting on `AUDIT` anomalies and 5xx spikes.
- **Assistant:** Sentry free tier; daily summary of tasks and drafts created, plus token spend.

---

## 11. Phased implementation plan

| Phase                                     | Weeks   | Work                                                                                                                                                                                                                                                                                                     | Exit criteria                                                                                                            |
| ----------------------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| **0. Decisions and validation** (no cost) | 1–2     | Confirm Google Workspace; xAI data terms and DPA; Supabase Postgres version; OpenProject Community feature checks (roles, API tokens, 2FA, SSO, CORS, subproject import in SP); Plainspace enquiry (roles, pricing, residency, self-hosting); approve the staff policy text (E2EE, AI processing opt-in) | All [A] items in §§6.4, 8.3, 8.5 and 9 resolved, or accepted as risks                                                    |
| **1. SuperSync pilot**                    | 2–3     | Supabase project; SuperSync on Fly (staging, then production); 3 pilot users; backups and uptime checks                                                                                                                                                                                                  | Two devices per pilot user sync for 1 week; restore test passed; offboarding runbook tested                              |
| **2. Team tracker**                       | 3–5     | OpenProject deploy; roles; 2 real shared projects; SP provider configured for pilot users                                                                                                                                                                                                                | Pilot users see assigned work in SP and status/time flows back; permissions verified (a non-member cannot see a project) |
| **3. AI assistant**                       | 5–8     | Gmail access, Grok extraction, shadow mode for 2 weeks, then tasks, then drafts                                                                                                                                                                                                                          | Precision ≥ target (for example ≥ 80% of created tasks kept); zero sends; cost within budget                             |
| **4. Rollout**                            | 8–10    | All 20 employees; training (one-page setup guide); runbooks (joiner, leaver, restore, upgrade)                                                                                                                                                                                                           | All staff onboarded; first monthly maintenance done                                                                      |
| **5. Operate**                            | ongoing | Monthly updates; quarterly restore test and access review; cost review                                                                                                                                                                                                                                   | —                                                                                                                        |

---

## 12. Risks and limitations

| #   | Risk                                                                         | Likelihood / impact                  | Mitigation                                                                                                                      |
| --- | ---------------------------------------------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| R1  | **Two systems for staff**: SP for personal work, OpenProject for shared work | Medium / Medium                      | Shared work flows _into_ SP automatically; training. If adoption is poor, consider option F                                     |
| R2  | One SP provider per OpenProject project creates configuration overhead       | Medium / Low                         | Test subproject import in Phase 0; few, larger projects                                                                         |
| R3  | **Lost encryption password = lost synced personal data** [V]                 | Medium / Medium                      | Password manager; SP local backups on desktop [V `src/app/imex/local-backup/`]                                                  |
| R4  | **365-day tokens survive allowlist removal** [V]                             | Certain / Medium                     | Mandatory `delete-user` step at offboarding (§7.4)                                                                              |
| R5  | SuperSync is single-instance: downtime during deploys or crashes             | Certain / Low                        | Offline-first clients keep working and sync later [V product principle `AGENTS.md:16`]                                          |
| R6  | Self-hosted server falls behind fast-moving clients                          | Medium / Medium                      | Monthly server updates; watch release notes and the client-version floor doc [V `docs/sync-and-op-log/client-version-floor.md`] |
| R7  | Supabase pooler incompatibility with Prisma migrations and locks             | Medium / Low                         | Use direct or session connections [I]                                                                                           |
| R8  | OpenProject resource needs (4 GB) or PG16 requirement not met                | Low / Medium                         | Verify in Phase 0; Fly machine sizing                                                                                           |
| R9  | AI misreads emails, creates noise, or is prompt-injected                     | Medium / Medium                      | Shadow mode, thresholds, no model tools, Inbox-only routing                                                                     |
| R10 | Email data processed by xAI                                                  | Certain / High if terms unacceptable | Phase 0 legal review; opt-in; exclusions; fallback to another provider                                                          |
| R11 | Unsandboxed SP plugins                                                       | Low / High                           | Policy: no third-party plugins                                                                                                  |
| R12 | Upstream direction changes (single-maintainer project)                       | Low / Medium                         | MIT licence; nothing forked; we can pin versions or migrate data via JSON export                                                |
| R13 | No built-in server metrics or APM                                            | Certain / Low                        | Uptime checks, log alerts, CLI monitor scripts [V `scripts/monitor.ts`]                                                         |

---

## 13. Decisions requested

1. Approve **Option C** (SP unmodified + self-hosted SuperSync + self-hosted OpenProject + assistant via the OpenProject API). Or choose D, E or F from §5.3.
2. Approve **Phase 0** (no spend) to resolve the open assumptions.
3. Confirm the **budget envelope**: about $110/month infrastructure plus an AI budget cap (recommend $50/month to start, with the budget model).
4. Confirm the **policies**: personal SP data is private and E2E-encrypted; AI email processing is opt-in with exclusions; no third-party SP plugins.
5. Confirm that the **fork stays a mirror of upstream**, with company code in a separate repository.

---

## 14. Sources

**Repository** (all paths relative to the repo root): see the inline `[V]` references.

**External (assumptions, Oct 2026):**

- Fly.io pricing: https://fly.io/docs/about/pricing/
- Supabase pricing summaries:
  - https://toolradar.com/blog/supabase-pricing-2026
  - https://swyftstack.com/blog/supabase-pricing-explained
  - https://metacto.com/blogs/the-true-cost-of-supabase-a-comprehensive-guide-to-pricing-integration-and-maintenance
- xAI Grok API pricing summaries:
  - https://www.morphllm.com/grok-api-pricing
  - https://anotherwrapper.com/llm-pricing/grok-4-fast
- Cloudflare Zero Trust free plan: https://costbench.com/software/business-vpn/cloudflare-zero-trust/free-plan/
- OpenProject system requirements: https://www.openproject.org/docs/installation-and-operations/system-requirements/
- OpenProject licence and editions: https://en.wikipedia.org/wiki/OpenProject, https://www.openproject.org/de/community-edition/
- Plainspace: https://plainspace.org (no public detail retrievable at time of writing)
