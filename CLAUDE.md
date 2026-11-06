# CLAUDE.md

Operating guide for Claude Code in this repository. Read it fully before planning or changing anything. When this file and a user instruction conflict, follow the user and flag the conflict. Items marked **[TO CONFIRM]** are assumptions the BRD does not settle; confirm them before building on them.

---

## 1. Project Context

- **Project:** Patient Management Application (web-based), built for a single general physician in a small clinic.
- **Repository:** `ClaudeCodeShravan` (local path: `D:\Claude Code\ClaudePractical\ClaudeCodeShravan\ClaudeCodeShravan`).
- **Branches on origin:** `main` (production-facing, release only), `development` (integration).
- **Source documents:**
  - `BRD/Doc_BRD.md`: the Business Requirements Document. It is the source of truth for product behaviour.
  - `Reference/AI Training.docx`: the learning plan that defines the 8-step workflow used in this repo (see section 15).
- **Current state (2026-11-06):** No application code exists yet. The working tree has `README.md` deleted and uncommitted, and the `BRD/` and `Reference/` folders are untracked. Do not restore or commit these changes without the user's approval.
- **Tech stack (mandated by the project owner):**
  - Frontend: Angular
  - Backend: ASP.NET Core (C#)
  - Database: SQL Server **[TO CONFIRM]** (the BRD does not name an engine)
  - Hosting, CI/CD, and PDF library: **[TO CONFIRM]**

---

## 2. Product Goal

Build a simple, fast, web-based system that lets a general physician run daily clinic work: appointments, patient records, complaints, diagnosis, medication, and printed prescriptions.

The goals are to:
- Cut manual paperwork (target: at least 80% reduction in paper usage).
- Make a consultation record take 2–3 minutes.
- Keep patient history accurate and easy to find.

---

## 3. BRD Summary

**Users:** One general physician (single user). Receptionist access is not in Phase 1.
**Stakeholders:** Clinic owner (doctor), Product Owner, Development Team.

**Functional requirements:**

| Area | Requirement |
|---|---|
| Patients | Add, edit, view. Capture name, age/DOB, gender, contact details. Search by name or phone. |
| Appointments | Schedule. View daily list. Status: Scheduled, Completed, Cancelled, No-show. |
| Consultation | **Mandatory vitals** (temperature, BP, pulse) on every consultation. Free-text complaints. Diagnosis notes. |
| Prescriptions | Medicines with name, dosage, frequency, duration, instructions. Printable prescription with clinic/doctor header, patient details, vitals, diagnosis, medications, and footer (notes and signature area). |
| History | View previous visits with vitals, complaints, diagnosis, and prescriptions. Filter by date. |
| Navigation | Quick search, recent patients, easy movement between patient profile and visits. |
| Export | Patient or visit data as CSV and PDF. |

**Non-functional requirements:**
- Usability: minimal UI optimised for fast entry during consultations.
- Performance: page load under 2 s; patient search and history retrieval within 2–5 s.
- Reliability: no data loss; regular automated backups.
- Security: secure single-user login; encryption at rest and in transit.
- Scale: one clinic, moderate patient volume.
- Browsers: current Chrome, Edge, and Safari.

**Success criteria:** consultation record in 2–3 min; search in 2–5 s; at least 80% less paper; prescriptions print smoothly; CSV/PDF export works; minimal training needed.

---

## 4. Scope (Phase 1)

- Browser-based access.
- Patient registration and profile management.
- Appointment scheduling and status tracking.
- Consultation: vitals (mandatory), complaints, diagnosis.
- Medication and prescription management, with printable prescriptions.
- Patient visit history with date filter.
- Basic search and recent-patients navigation.
- Data export (CSV, PDF).
- Secure single-user login.

---

## 5. Out of Scope

Do **not** build, stub, or scaffold these in Phase 1. If a task seems to need one, stop and ask.

- Receptionist or any multi-user / role-based access.
- Billing and invoicing.
- Insurance processing.
- Lab or pharmacy integrations.
- AI-based diagnosis or recommendations.
- Offline mode.
- Mobile application (the web app must still be responsive; see section 6).
- Advanced analytics and reporting.
- Multi-doctor or multi-clinic support.
- Follow-up alerts and reminders.

If a feature is only "nice to have," record it as a backlog idea in `docs/backlog.md`. Do not implement it.

---

## 6. Architecture Rules

**Layers and dependency direction:**

```
Angular SPA  →  ASP.NET Core API  →  Application services  →  Domain  
                                          ↓
                                 Data access (Dapper)  →  SQL Server
```

1. Angular talks to the API only, over HTTP. It never touches the database or connection strings.
2. Controllers are thin: they validate input, call an application service, and map the result. No business logic in controllers or components.
3. Business rules (for example, mandatory vitals before a consultation can be completed, or appointment status transitions) live in the application/domain layer and are unit-tested there.
4. API contracts use DTOs. Never expose database row types or domain entities over the wire.
5. The schema is defined in versioned SQL migration scripts (see section 9). Schema changes happen only through those scripts, never by ad-hoc DDL against a shared database. **[TO CONFIRM: migration tool, recommended DbUp]**
6. Keep one clinic and one doctor in the model. Do not add multi-tenant columns or abstractions. If clinic header details are needed for prescriptions, store them as configuration, not as a tenant model.
7. Prefer simple, boring solutions. Add a pattern only when a concrete requirement needs it.
8. Cross-cutting concerns (logging, error handling, auth, validation) are set up once, centrally, and reused.
9. Any significant architectural decision is recorded as an ADR in `docs/adr/` (see section 13).

**Repository layout (target):**

```
/src
  /Web                 Angular application
  /Api                 ASP.NET Core Web API
  /Application         Use cases, services, DTOs, validation
  /Domain              Entities and business rules (no infrastructure references)
  /Infrastructure      Dapper repositories, SQL migration scripts, connection factory, PDF/CSV export, external services
/tests
  /Domain.Tests
  /Application.Tests
  /Api.Integration.Tests
  /Web.Tests
  /e2e
/docs
  /adr                 Architecture Decision Records
  backlog.md
/BRD                   Business Requirements (read-only source of truth)
/Reference             Training material (do not modify)
```

---

## 7. Angular Standards

- **Version:** use the latest stable Angular and keep it consistent across the project. Do not mix versions.
- **Language:** TypeScript with `strict: true`. No `any`. Use explicit types for service and API models.
- **Components:** standalone components. Use `ChangeDetectionStrategy.OnPush` by default.
- **Dependency injection:** use the `inject()` function in components and services.
- **Forms:** typed reactive forms. Template-driven forms are not allowed for data entry.
- **State and data:** services wrap `HttpClient`. Components do not call `HttpClient` directly. Use `async` pipe or signals; avoid manual subscriptions. Unsubscribe or use `takeUntilDestroyed`.
- **Routing:** lazy-load feature routes. Each feature gets its own route file.
- **Folder structure:** feature-based (for example, `features/patients`, `features/appointments`, `features/consultation`, `features/prescriptions`), with shared UI in `shared/`.
- **UI rules:**
  - Fast data entry: keyboard-friendly forms, sensible tab order, autofocus on the first field, and Enter to submit where appropriate.
  - Consultation screen must show vitals, complaints, diagnosis, and medications on one screen, without extra navigation.
  - Responsive layout that works in Chrome, Edge, and Safari.
  - Accessibility: WCAG 2.1 AA as the baseline. Labels on every input, visible focus states, and colour contrast that passes.
- **Printing:** prescription print view uses a dedicated print stylesheet (`@media print`). Verify the output with a print preview.
- **Tooling:** ESLint and Prettier. Lint and format must pass before commit.
- **Performance:** keep the initial bundle small. Lazy-load heavy features such as PDF preview. Page load target is under 2 s.

---

## 8. ASP.NET Core Standards

- **Target:** latest LTS version of .NET. Use `<Nullable>enable</Nullable>` and `<ImplicitUsings>enable</ImplicitUsings>` in every project.
- **API style:** controllers or minimal APIs, used consistently. Choose one per project and do not mix them.
- **Layering:** controllers → application services → Dapper repositories. Application services take and return DTOs. Repositories map rows to Domain entities or DTOs; no row type leaks upward.
- **Async:** every I/O method is `async` and returns `Task` or `Task<T>`. Accept and pass `CancellationToken`. No `.Result` or `.Wait()`.
- **Validation:** validate requests at the boundary (data annotations or FluentValidation). Business-rule validation lives in the application layer and returns clear errors.
- **Errors:** return RFC 7807 `ProblemDetails` for all errors. Do not leak stack traces or internal details to clients. Use a central exception handler.
- **Responses:** use the right HTTP status codes (201 on create, 404 on missing, 409 on conflict, 422 on business-rule failures).
- **Configuration:** `appsettings.json` for non-secret defaults; environment variables or user-secrets/Key Vault for secrets. Bind options to typed classes with validation.
- **Logging:** structured logging (`ILogger<T>` with message templates). Never log patient data (see section 10).
- **API docs:** OpenAPI/Swagger enabled in Development only.
- **Dates:** store and transmit UTC. Convert to the clinic's local time zone only in the UI or the print/export layer.
- **Exports:** CSV and PDF generation is behind an interface (`IExportService`) so it can be tested and replaced.
- **Health:** expose a health-check endpoint that includes a database check.
- **Code style:** follow the `.editorconfig`. Treat analyzer warnings as errors in CI.

---

## 9. SQL Standards

- **Engine:** SQL Server **[TO CONFIRM]**. Write T-SQL that is valid for the chosen engine only.
- **Naming:**
  - Tables: PascalCase, plural (`Patients`, `Appointments`, `Visits`, `Prescriptions`, `PrescriptionItems`).
  - Columns: PascalCase, singular (`FullName`, `DateOfBirth`).
  - Primary keys: `Id` (identity `bigint`). Foreign keys: `<Parent>Id` (for example, `PatientId`).
  - Constraints and indexes: `PK_<Table>`, `FK_<Table>_<Parent>`, `IX_<Table>_<Columns>`, `CK_<Table>_<Rule>`.
- **Types:**
  - Timestamps: `datetime2` in UTC, with `CreatedAtUtc` and `UpdatedAtUtc` on every table.
  - Money or dosage values: `decimal` with explicit precision. No `float`.
  - Free text: `nvarchar` with a length limit. Use `nvarchar(max)` only for notes that genuinely need it.
- **Integrity:**
  - Enforce foreign keys, NOT NULL, and CHECK constraints in the database, not only in code.
  - Appointment status is constrained to the BRD values: Scheduled, Completed, Cancelled, No-show.
  - Vitals columns are NOT NULL for completed visits.
- **Deletion:** medical records are not hard-deleted. Use soft-delete flags (`IsDeleted`, `DeletedAtUtc`) for patients, visits, and prescriptions. Record the reason where relevant.
- **Queries:**
  - No `SELECT *`. Name every column.
  - Parameterised queries only. No string concatenation of user input.
  - Index the columns used by search (patient name, phone) and by the daily appointment list (date, status).
  - Use paging for lists. Keep search under the 2–5 s target on realistic data sizes.
- **Migrations:** versioned SQL scripts, one per logical change, with descriptive names (for example, `0003_AddPrescriptionItems.sql`). Review each script before applying. Never edit an applied script; add a new one. Dapper does not generate schema, so the scripts are hand-written and reviewed.
- **Dapper rules:** use parameterised `DynamicParameters` or anonymous-object parameters only. Keep SQL in the repository layer, not in controllers or services. Use explicit column lists in every query. Pass `CancellationToken` through `CommandDefinition`. Open connections through a factory, and never hold a connection in a singleton.
- **Seed data:** only synthetic data. Never commit real patient data, even for testing.
- **Backups:** automated daily backups with a documented restore procedure. Restore must be tested before release.

---

## 10. Security Standards

This system stores identifiable health information. Treat all patient data as sensitive.

- **Authentication:** single-user login. Passwords are hashed with a modern algorithm (ASP.NET Core Identity defaults or equivalent). Enforce a strong password policy and lockout after repeated failures.
- **Sessions:** secure, HTTP-only cookies. Session idle timeout. Logout invalidates the session.
- **Transport:** HTTPS everywhere, with HSTS in production. Redirect HTTP to HTTPS.
- **Encryption at rest:** database and backups are encrypted (for example, TDE or equivalent). Choose the mechanism in an ADR **[TO CONFIRM]**.
- **Authorisation:** every endpoint requires authentication unless it is explicitly public (for example, the health check). Default-deny.
- **Input and output:**
  - Validate all input server-side. The client is not a security boundary.
  - Encode output in Angular (default). Do not use `innerHTML` or `bypassSecurityTrust*` with user data.
  - Use parameterised queries (section 9).
- **CSRF:** protect state-changing requests (anti-forgery tokens or SameSite cookies plus token validation).
- **CORS:** restrict to the known Angular origin. No wildcard origins.
- **Secrets:** never commit secrets, connection strings, or keys. Use user-secrets locally and a secret store in deployed environments. Scan commits for secrets before pushing.
- **Logging and PHI:** logs and telemetry must not contain patient names, phone numbers, DOB, diagnosis, or prescriptions. Log IDs and event types only.
- **Dependencies:** keep NuGet and npm packages current. Review new dependencies before adding them. Run `dotnet list package --vulnerable` and `npm audit` in CI.
- **Audit trail:** record who viewed or changed a patient record, and when. (Recommended; confirm with the Product Owner if it is in scope for Phase 1.)
- **Exports:** CSV and PDF exports are generated on demand and are not stored in publicly reachable locations. Exports are authenticated downloads.
- **Error handling:** generic messages to clients; detailed errors only in server logs.
- **Security review:** use `/security-review` on any change that touches auth, data access, exports, or the API surface.

---

## 11. Testing Standards

**Principle:** every BRD functional requirement has at least one automated test that proves it. Tests are part of the deliverable, not an afterthought.

- **Test pyramid:**
  - **Domain and Application unit tests (xUnit):** business rules such as mandatory vitals, appointment status transitions, prescription validation. Fast, no I/O.
  - **Integration tests (xUnit + real SQL Server via Testcontainers or LocalDB):** SQL migration scripts, Dapper repository queries, search performance on sample data, API endpoints end to end.
  - **Angular unit tests:** components, services, and forms. Use the project's configured runner (Jest or Jasmine/Karma **[TO CONFIRM]**). Mock `HttpClient` with `HttpTestingController`.
  - **End-to-end (Playwright):** critical user journeys: register patient → schedule → consult with vitals → add medicines → print prescription → export CSV/PDF. Run on Chrome, Edge, and WebKit (Safari engine).
- **Naming:** test names describe behaviour, for example `Consultation_CannotComplete_WithoutVitals`.
- **Structure:** Arrange / Act / Assert. One behaviour per test.
- **Determinism:** no real clock, network, or random data in unit tests. Inject `TimeProvider` (or an equivalent clock abstraction). Use fixed seeds.
- **Test data:** synthetic data only (see section 9).
- **Coverage:** Domain and Application layers target 80% line coverage or better. Coverage is a guide, not a goal; a test must assert behaviour.
- **Performance checks:** automated checks that patient search and page loads stay within the BRD targets on a realistic dataset (for example, 10,000 patients).
- **Regression:** every bug fix adds a test that fails before the fix and passes after.
- **CI gate:** all tests, lint, format, and vulnerability checks must pass before a PR can be merged.
- **Manual checks:** print output of prescriptions and PDF/CSV exports must be checked by a person on at least one supported browser before release.

---

## 12. Git Standards

**Ownership:** Claude owns all Git operations in this repository. This includes branch creation, worktree management, commits, pushes, and PR creation. The user approves merges and branch cleanup.

**Branches:**

| Branch | Purpose | Rules |
|---|---|---|
| `main` | Released, production-ready code | Changed only through a release PR from `development`. Never committed to directly. |
| `development` | Integration branch | Receives merged feature branches. Never committed to directly. |
| `feature/<short-slug>` | One unit of work | Created from `development`. Named in lowercase with hyphens (for example, `feature/consultation-vitals`). |

**Worktrees:**
- Each feature is developed in its own worktree, created from `development`:
  `git worktree add ../ClaudeCodeShravan-worktrees/<branch-slug> -b feature/<slug> development`
- Work is never done directly in the main checkout on `main` or `development`.
- Each worktree is used for one feature only.

**Commits:**
- Conventional Commits format: `type(scope): summary`, with types `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `build`.
- Keep commits small and focused. Each commit builds and passes tests where practical.
- Subject line: imperative mood, 72 characters or fewer. Body explains *why* when it is not obvious.
- Every commit made by Claude ends with this trailer:
  ```
  Co-Authored-By: Claude Haiku 5.5 <noreply@anthropic.com>
  ```
- Never use `--no-verify` or skip signing unless the user explicitly asks.

**Integration flow:**
1. Claude creates the feature branch and worktree from `development`.
2. Claude implements, tests, and commits on the feature branch.
3. Claude pushes the feature branch and opens a PR into `development`.
4. Claude requests the user's approval before merging. **Claude does not merge without explicit approval.**
5. After approval, Claude merges into `development`.
6. **Merged feature branches are retained.** Do not delete them after merge.
7. Release: Claude opens a release PR from `development` to `main`. Merge to `main` requires explicit approval.

**Branch cleanup:** removing branches (local or remote) or removing worktrees requires explicit user approval. Claude lists the candidates and waits for a yes.

**Pull requests:**
- Title follows the commit convention.
- Description includes: purpose, linked BRD requirement(s), what changed, how it was tested, screenshots for UI changes, and known gaps.
- Every PR description ends with:
  ```
  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  ```
- Never push directly to `main` or `development`. Never force-push a shared branch.

**Safety:**
- Before any destructive Git operation (reset, rebase of shared branches, force-push, deleting refs), stop and ask.
- Before committing, check `git status` and make sure no secrets, `bin/`, `obj/`, `node_modules/`, or personal files are staged. Keep `.gitignore` current.
- The uncommitted `README.md` deletion and the untracked `BRD/` and `Reference/` folders are the user's work in progress. Do not stage or commit them without approval.

---

## 13. Documentation Standards

- **Source of truth:** `BRD/Doc_BRD.md` defines behaviour. If the code and the BRD disagree, stop and ask. Do not silently change either.
- **README.md:** setup instructions (prerequisites, restore, run, test, migrate, seed), project structure, and links to the docs below. Keep it accurate in the same PR as the change that affects it.
- **ADRs:** significant decisions (database engine, auth approach, encryption, PDF library, hosting) go in `docs/adr/NNNN-short-title.md` with: context, decision, alternatives, and consequences. Number ADRs sequentially. Do not rewrite accepted ADRs; supersede them.
- **API documentation:** generated from OpenAPI. Endpoints without XML/summary docs are not complete.
- **Code comments:** explain *why*, not *what*. Do not comment obvious code. Public domain and application APIs have XML doc comments.
- **Changelog:** `CHANGELOG.md` updated on each release PR, grouped by Added, Changed, Fixed, Security.
- **Backlog:** out-of-scope or deferred ideas go in `docs/backlog.md`.
- **CLAUDE.md:** keep it current. When a standard changes, update this file in the same PR.
- **Writing style:** plain English, short sentences, active voice. Use tables for comparisons and lists for steps.

---

## 14. Definition of Done

A task is done only when **all** of these are true:

1. **Requirement traced:** the work maps to a BRD requirement or an approved plan item. Any deviation is recorded and approved.
2. **Plan approved:** an implementation plan (steps and test approach) was reviewed and approved before coding started.
3. **Built in a worktree:** the work was done on a `feature/*` branch in its own worktree, not on `main` or `development`.
4. **Tests written and passing:** unit, integration, and (for user-facing flows) E2E tests. Existing tests still pass.
5. **Static checks clean:** lint, format, analyzers, and vulnerability checks pass with zero new warnings.
6. **Verified:** the feature was run and checked against its acceptance criteria. For UI, checked in a real browser.
7. **Gap analysis done:** the result was compared to the BRD requirement and success criteria. All gaps are either fixed or logged in `docs/backlog.md` with user agreement.
8. **Security reviewed:** changes touching auth, data, exports, or the API passed `/security-review` and the standards in section 10.
9. **Code reviewed:** review comments are addressed within the agreed timeline.
10. **Docs updated:** README, ADRs, API docs, and CHANGELOG are current where affected.
11. **Committed cleanly:** Conventional Commits, trailer included, no secrets, no generated artifacts.
12. **PR opened:** description complete (section 12), and approval requested from the user.
13. **Merged only after approval**, and the feature branch is retained.
14. **Worktree status reported:** the worktree is kept until the user approves cleanup.

For a **release** (development → main), also confirm: all Phase 1 success criteria are checked, backup restore has been tested, the CHANGELOG is updated, and the user has approved the release PR.

---

## 15. Learning Guidelines

This repository is also a training environment. The developer follows the plan in `Reference/AI Training.docx`.

**Phase 1: Prompt-based execution.** Work through all eight modules in order. Each module is a checkpoint.

| # | Module | Objective | Exit check |
|---|---|---|---|
| 1 | Brainstorming | Understand requirements, scope, constraints, options | Requirement understanding reviewed by mentor/lead |
| 2 | Planning | Detailed implementation and test plan | Plan approved before implementation |
| 3 | Worktree | Isolated development | Worktree created and managed; `main` untouched |
| 4 | Implementation | Execute the approved plan step by step | Assigned work complete per plan |
| 5 | Verification | Run tests, check outputs against requirements | Issues found are fixed before proceeding |
| 6 | Gap Analysis | Compare solution to original requirements | Requirement coverage measured; all gaps addressed |
| 7 | Code Review | Improve quality and consistency | Comments resolved within the agreed timeline |
| 8 | Finishing | PR, merge readiness, worktree cleanup | PR or merge-ready; cleanup completed on approval |

**How Claude should teach in this mode:**
- Explain the *reason* behind each step, not only the command. Name the module you are in.
- Do not jump ahead. Brainstorm first, plan second, and wait for plan approval before implementing.
- Show options with trade-offs and recommend one. The developer makes the decision.
- Ask the developer to predict the result before running a risky command (for example, a migration or a merge), then compare.
- After each module, summarise what was done and what remains, in a short checklist.
- When a mistake happens, explain the cause and the fix. Do not just silently correct it.
- Keep a progress log in `docs/learning-log.md`: module, date, status, and open items. This supports the weekly assessment.

**Phase 2: Agent-driven execution (advanced).** Only after all eight Phase 1 modules are complete, the same lifecycle is redone using agents, sub-agents, skills, and commands, with the same quality, verification, and tracking standards. Until then, do not use sub-agents or custom skills to shortcut the workflow the developer is learning.

---

## 16. Working Rules for Claude

- **Ask before acting** when a requirement is ambiguous, a BRD item conflicts with another, or an action is hard to reverse (merge, delete, force-push, schema drop, changing `main`).
- **Stay in scope.** If an out-of-scope item appears necessary, stop and ask. Do not implement it.
- **Show evidence.** Report test results, command output, and gap-analysis outcomes. Say plainly when something failed or was skipped.
- **Keep changes small.** Prefer several small, reviewable PRs over one large one.
- **Do not guess at security or medical logic.** Ask for a decision and record it as an ADR.
