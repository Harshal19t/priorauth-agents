# Prior-Authorization Multi-Agent System: Project Document

Nov 24, 2026 · @Harshal

## Overview

This project builds a team of small AI agents on Cloudflare that prepares prior-authorization requests for a clinic, with a human approving every submission. It runs on synthetic (fake) patient data, costs $0 on free tiers, and is sized for one person to build in seven days.

**Goals**

- Show an industry-standard multi-agent design: one orchestrator, several narrow agents, and guardrails enforced in code rather than in prompts.
- Show full automation: tests, evals, security scans, staged deploys and rollback in a CI/CD pipeline.
- Produce a public GitHub repository with a README, diagrams and a demo you can show employers.

**Non-goals**

- No real patient data, no diagnosis and no clinical decisions.
- No live connection to real insurers. The payer is simulated.
- Not a certified medical product and not a HIPAA-compliant deployment (see the Security section).

## Problem and scope

Prior authorization is the step where an insurer must approve a test or treatment before it happens, and clinics still prepare each request by hand. The system automates the preparation and leaves the decision to submit with a person.

Every case runs through the same steps:

1. A clinic sends a referral (synthetic patient reference, payer name, requested service, clinical notes) to the API.
2. The Intake agent extracts symptom duration, past treatments, past imaging and red-flag symptoms.
3. The Coding agent suggests a procedure code (CPT) and a diagnosis code (ICD-10).
4. The Policy agent marks each rule of the payer's policy as met, not met or unknown. Code, not the model, turns those marks into likely approve, needs more info, or likely deny.
5. The packet builder assembles a FHIR Claim marked as a preauthorization request.
6. The workflow pauses until a human reviewer approves or rejects.
7. On approval the packet goes to a simulated payer. A denial triggers the Appeal agent, which drafts a letter for the reviewer to edit.

The synthetic rulebook has two policies. A lumbar spine MRI (CPT 72148) needs 6 or more weeks of symptoms and a conservative treatment such as physical therapy, unless a red flag is present. A knee MRI (CPT 73721) needs 4 or more weeks of symptoms and a prior X-ray.

| In scope | Out of scope |
| --- | --- |
| Two services: lumbar MRI and knee MRI | Other procedures and drug authorizations |
| Synthetic referrals only | Real patient data |
| Simulated payer | Real insurer APIs and payer portals |
| Human approval before submission | Autonomous submission |
| Drafted appeal letters | Clinical advice or diagnosis |

## Architecture

A single Cloudflare Worker exposes the API and runs one durable Workflow per case; the agents are plain TypeScript functions called as workflow steps, so the model gathers evidence and code makes every decision.

&#91;embedded content: architecture · 1 API, 5 workflow steps, 3 supporting services\]

The Policy step is highlighted because it is where code, not the model, turns the evidence into a decision.

- **Worker API (Hono):** checks the bearer token, validates input with Zod, starts a workflow and delivers the human's decision to it.
- **Workflow:** runs the steps in order, retries failed steps, and pauses at the approval step until a reviewer responds.
- **Agents:** Intake, Coding, Policy and Appeal call the model through one `Llm` interface; the packet builder is deterministic code.
- **LLM layer:** removes personal identifiers from every prompt, then calls Workers AI through AI Gateway, or uses offline rules when no model is configured.
- **D1 database:** stores each case and an audit entry for every step.
- **Payer simulator:** returns approved or denied from the Policy result, so the appeal path can be tested without a real insurer.

## Tech stack

Everything runs on Cloudflare's free tier with TypeScript, so there are no servers to manage and no cost while you build.

| Layer | Technology | Role | Why |
| --- | --- | --- | --- |
| Language | TypeScript (strict mode) | All application code | Catches errors before runtime; native to Workers |
| Runtime | Cloudflare Workers | API and agent code | Free tier, global, no servers |
| Web framework | [Hono](https://hono.dev/getting-started/cloudflare-workers) | Routing and auth middleware | Small and built for Workers |
| Orchestration | [Cloudflare Workflows](https://developers.cloudflare.com/workflows/) | Durable steps, retries, wait for human approval | Resumes after failure; `waitForEvent` fits the approval gate |
| Database | [Cloudflare D1](https://developers.cloudflare.com/d1/get-started/) | Cases and audit log | SQLite, free tier, versioned migrations |
| AI access | Workers AI through AI Gateway | Model calls, logging, rate limits, DLP | One place to swap providers later |
| Validation | [Zod](https://zod.dev/basics) | Schemas for inputs and every agent output | Rejects malformed model output |
| Testing | [Vitest](https://vitest.dev/guide/) | Unit tests, evals, leak and injection tests | Fast and TypeScript-native |
| CLI | Wrangler | Local dev, migrations, deploy, rollback | Official Cloudflare tool |
| CI/CD | GitHub Actions with `wrangler-action` | Checks and deployments | Free for public repos |
| Security scans | Gitleaks, `npm audit`, Dependabot | Secrets and dependency risks | Automated on every change |
| Data standard | HL7 FHIR R4 Claim | Submission format | Industry format for prior authorization |

## Repository

Use one public GitHub repository named `priorauth-agents`, with a single Worker project at the root and no monorepo tooling.

```text
priorauth-agents/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml             # checks on every pull request
│   │   └── deploy.yml         # staging, approval, production
│   └── dependabot.yml
├── src/
│   ├── index.ts               # Hono API; exports the workflow
│   ├── workflow.ts            # PriorAuthWorkflow (durable steps)
│   ├── agents.ts              # intake, coding, policy, appeal
│   ├── packet.ts              # FHIR Claim builder
│   ├── policies.ts            # payer rules as code
│   ├── llm.ts                 # Llm interface: offline and Workers AI
│   ├── heuristics.ts          # offline rules used instead of a model
│   ├── redact.ts              # PHI redaction
│   ├── schemas.ts             # Zod schemas
│   ├── auth.ts                # bearer-token check
│   └── env.ts                 # bindings and variables types
├── migrations/
│   └── 0001_init.sql          # cases and audit_log tables
├── evals/
│   └── golden.json            # six golden cases
├── test/
│   ├── redact.test.ts
│   ├── agents.test.ts
│   └── evals.test.ts
├── samples/                   # example referral JSON files
├── scripts/
│   └── demo.sh                # runs one case end to end
├── docs/
│   └── architecture.md
├── wrangler.jsonc             # bindings, staging and production
├── package.json
├── tsconfig.json
├── .gitignore                 # includes .dev.vars
└── README.md
```

**Conventions**

- Trunk-based work: short-lived feature branches, pull requests into `main`, and branch protection that requires the CI checks to pass.
- Conventional commit messages, such as `feat: add policy agent`.
- Two Cloudflare environments, `staging` and `production`, defined in `wrangler.jsonc` with their own D1 databases and secrets.
- Secrets never enter the repository. Local values live in `.dev.vars` (ignored by Git); deployed values are Wrangler secrets and GitHub environment secrets.

## Agents, data model and API

Four agents call the model and one step is plain code; every model output is validated against a Zod schema before anything uses it.

| Agent | Input | Output | Uses a model | Guardrail |
| --- | --- | --- | --- | --- |
| Intake | Referral | Duration, treatments, imaging, red flags | Yes | Zod schema; notes are treated as untrusted data |
| Coding | Intake result | CPT and ICD-10 suggestions | Yes | Zod schema; suggestions only, a human reviews |
| Policy | Intake and coding results | Met, not met or unknown per rule | Yes | Returned rule ids must match the policy; code computes the decision |
| Packet builder | All results | FHIR Claim (preauthorization) | No | Deterministic code |
| Appeal | Packet and denial reason | Draft appeal letter | Yes | A human edits and sends it |

The database has two tables. Statuses below are a proposal: received, analyzing, awaiting\_approval, rejected, submitted, approved, denied, needs\_info and expired.

| Table | Columns |
| --- | --- |
| `cases` | `id`, `status`, `created_at`, `updated_at`, `result_json` |
| `audit_log` | `id`, `case_id`, `ts`, `actor`, `action`, `detail_json` |

| Method | Path | Purpose | Auth |
| --- | --- | --- | --- |
| GET | `/health` | Liveness check used by smoke tests | None |
| POST | `/cases` | Validate a referral, start a workflow, return the case id | Bearer token |
| GET | `/cases/:id` | Status, results and audit trail | Bearer token |
| POST | `/cases/:id/approval` | Send approve or reject and the reviewer's name to the waiting workflow | Bearer token |

## Security, privacy and compliance

No model output can cause an action on its own, and personal identifiers are stripped from text before it reaches a model. This is a learning project on synthetic data, not a compliant system.

**Controls built into the design**

- **Data minimization:** referrals carry a synthetic patient reference, never a name.
- **Redaction:** one wrapper around every model call removes SSNs, phone numbers, emails, dates, record numbers and titled names. It is a baseline, not full de-identification.
- **Untrusted input:** clinical notes sit inside delimiters, the prompt says to ignore instructions inside them, and code makes the decision, so injected text cannot approve a case.
- **Output validation:** Zod rejects any model response that does not match its schema.
- **Authentication:** every route except `/health` needs a bearer token held as a secret.
- **Audit trail:** every step writes who or what acted, what happened and when.
- **Human-in-the-loop:** nothing is submitted until a reviewer approves.
- **Secret hygiene:** Wrangler and GitHub environment secrets, plus Gitleaks in CI.

| Standard | What it asks | How this project responds | Gap |
| --- | --- | --- | --- |
| [HIPAA de-identification](https://www.hhs.gov/hipaa/for-professionals/special-topics/de-identification/index.html) | Remove 18 listed identifier types (Safe Harbor) or use expert determination | Regex redaction of common identifiers | No name recognition; not a certified method |
| FHIR R4 and [Da Vinci PAS](https://hl7.org/fhir/us/davinci-pas/specification.html) | Use a standard claim format | FHIR Claim with `use` set to `preauthorization` | Simplified; not the full PAS bundle |
| OWASP Top 10 for LLM Applications | Defend against prompt injection, excessive agency and data disclosure | Untrusted-input handling, human gate, redaction | No adversarial red-teaming beyond the tests |
| GDPR and UK data residency | Minimize data and control where it is processed | Synthetic data only | Cloudflare's [compatibility table](https://developers.cloudflare.com/data-localization/compatibility/index.md) shows Workers AI is not covered by Regional Services |

**Before any real patient data**

- A HIPAA business associate agreement is needed, and Cloudflare signs these only with Enterprise customers. Which developer products a BAA covers must be confirmed with Cloudflare in writing.
- Workflow state stores case data, so retention and access need review.
- Replace regex redaction with a proper de-identification step and have the design reviewed by a compliance specialist.

## CI/CD and automation

GitHub Actions checks every change, deploys to staging automatically, and needs a human approval before production, with an automatic rollback if the production smoke test fails.

&#91;embedded content: release pipeline · 2 lanes, 2 checks, 1 approval gate\]

A failed pull-request check loops back to the author; a failed production smoke test triggers the rollback.

**On every pull request**

1. Type-check with `tsc`.
2. Run unit tests and the golden-set evals with Vitest.
3. Scan for committed secrets with Gitleaks.
4. Audit dependencies with `npm audit`.

**On merge to main**

1. Apply D1 migrations to staging, deploy, and call `GET /health`.
2. Pause for a reviewer through a protected GitHub environment named `production`.
3. Apply migrations and deploy to production, then call `GET /health` again.
4. If the check fails, run `wrangler rollback` to restore the previous version.

**Setup notes**

- Store `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` as GitHub environment secrets, and the API token as a Wrangler secret per environment.
- Turn on branch protection so nothing merges without green checks.
- Dependabot opens weekly dependency updates.
- Required reviewers on GitHub Free work only for public repositories. The Gitleaks Action needs a license key for organization accounts; a personal account avoids this.
- Cloudflare also supports [gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/) as a later upgrade to the production step.

## Testing and evals

Six golden cases and four kinds of automated test run on every pull request, and the build fails if decision accuracy drops below the threshold. Offline rules stand in for the model, so these tests are free and repeatable; on Day 7 the same evals run against the real model for comparison.

| Layer | Tool | What it proves |
| --- | --- | --- |
| Unit | Vitest | Redaction patterns, schemas and the packet builder behave as specified |
| Agent | Vitest with the offline LLM | Each agent returns output that passes its schema |
| Evals | Vitest on the golden set | Codes and decisions match the expected values |
| PHI leak | Vitest with a spy LLM | No identifier reaches any prompt across all cases |
| Prompt injection | Vitest | Notes that say "approve this" do not change the decision |
| Smoke | `curl` after deploy | The deployed Worker is up |

| Golden case | Key facts | Expected decision |
| --- | --- | --- |
| Lumbar MRI, approve | 8 weeks of pain; physical therapy and ibuprofen tried | Likely approve |
| Lumbar MRI, deny | 2 weeks of pain; no treatment tried | Likely deny |
| Lumbar MRI, missing data | Physical therapy mentioned; no duration | Needs more info |
| Lumbar MRI, red flag | 3 days of pain; saddle anesthesia and bladder dysfunction | Likely approve |
| Knee MRI, approve | 6 weeks of pain; X-ray done | Likely approve |
| Knee MRI, deny | 6 weeks of pain; no imaging | Likely deny |

Seed some golden notes with fake identifiers, such as a made-up date of birth and phone number, so the PHI-leak test has something to catch.

## Seven-day plan

The plan has seven days and three gates, and each day ends with something that runs, so you always hold a working version.

&#91;embedded content: plan · 7 days, 3 gates\]

Do not start the next day's work until the gate before it passes; the table below lists each day's deliverable and what done means.

| Day | Goal | Deliverable | Done when |
| --- | --- | --- | --- |
| 1 | Skeleton and data | Worker with `/health`, repo on GitHub, `evals/golden.json` with 6 cases | `wrangler dev` returns ok and the repo is pushed |
| 2 | The agents | Zod schemas, Intake, Coding and Policy agents, the `Llm` interface with offline rules, policies as code | All six golden notes produce valid structured output |
| 3 | Privacy and tests | `redact.ts`, PHI-leak and injection tests, eval threshold | `npm test` is green and a planted identifier fails the leak test |
| 4 | Workflow and database | D1 migration, `PriorAuthWorkflow` with retries, audit log | A case runs through all steps locally and the audit log shows each one |
| 5 | API and approval | Routes, bearer auth, FHIR Claim, Appeal agent, `demo.sh` | `demo.sh` takes a case from referral to approval and appeal |
| 6 | CI/CD | `ci.yml`, `deploy.yml`, environments, branch protection, Dependabot | A pull request runs checks and a merge deploys to staging |
| 7 | Real AI and docs | AI Gateway model option, eval comparison, README, diagram, demo GIF | Evals pass in model mode and the README is complete |

If a day slips, cut scope in this order: the appeal agent, the preview deploys, the model comparison. Never cut the tests.

## Cost and free-tier limits

Expect $0: every service in the plan has a free tier, and Cloudflare's free tier stops working at its limit instead of billing you. Pricing was checked on 2026-11-24 and can change.

| Service | Free-tier fact | What it means here |
| --- | --- | --- |
| [Workers](https://developers.cloudflare.com/workers/platform/pricing/) | 100,000 requests a day | Far above what a demo uses |
| [Workflows](https://developers.cloudflare.com/workflows/reference/pricing) | Included in the free plan; billed as Workers; state kept 3 days | An approval left waiting more than 3 days can lose its state |
| [D1](https://developers.cloudflare.com/d1/platform/pricing/) | 500 MB per database, 10 databases | Ample for fake cases and an audit log |
| [Workers AI](https://developers.cloudflare.com/workers-ai/platform/pricing/) | 10,000 Neurons a day; no usage beyond that without the paid plan | Enough for a few test cases a day on Day 7 |
| [AI Gateway](https://developers.cloudflare.com/ai-gateway/reference/pricing) | Available on all plans; log storage is capped | Fine for a portfolio project |

Costs appear only if you upgrade to Workers Paid (minimum $5 a month), use a third-party model provider that bills per token, or add a custom domain. GitHub Actions is free for public repositories on standard runners; check GitHub's pricing page for private repositories.

## Risks, limitations and stretch goals

The biggest risks are scope creep and trusting unverified details, so keep to two services and check every code and API name against current documentation.

| Risk | Mitigation |
| --- | --- |
| Scope creep into more procedures or real insurer APIs | Hold to two services and a simulated payer |
| Medical codes in examples are wrong or outdated | Treat codes as examples and verify them against the current official CPT and ICD-10 code sets |
| Cloudflare APIs or flags changed since this document | Check the Workflows, D1 and Wrangler docs before each day |
| Offline rules pass but the real model fails | Run the same evals in model mode on Day 7 and set a separate threshold |
| Free-tier limits interrupt testing | Do most work offline and check the Workers AI usage dashboard |
| Workflow state expires after 3 days on the free plan | Use short approval windows in demos |

**Stretch goals**

- Terraform for the D1 databases and other infrastructure.
- Integration tests with `@cloudflare/vitest-pool-workers`.
- A small reviewer dashboard.
- A cost and latency report per agent.
- Gradual production deployments.

## Reading list by day

Read each day's links the evening before you do that day's work; all were checked on 2026-11-24.

| Day | Topic | Source |
| --- | --- | --- |
| 1 | Your first Worker | [Cloudflare: create your first Worker](https://developers.cloudflare.com/workers/get-started/guide/) |
| 1 | Hono on Workers | [Hono: Cloudflare Workers](https://hono.dev/getting-started/cloudflare-workers) |
| 1 | Synthetic patient data | [Synthea on GitHub](https://github.com/synthetichealth/synthea) |
| 1 | Golden datasets | [Langfuse: golden dataset evaluation](https://langfuse.com/resources/engineering/golden-dataset-evaluation.md) |
| 2 | Agent design patterns | [Anthropic: building effective agents](https://www.anthropic.com/research/building-effective-agents) |
| 2 | Orchestrator-workers | [Anthropic cookbook: orchestrator-workers](https://platform.claude.com/cookbook/patterns-agents-orchestrator-workers) |
| 2 | When to use many agents | [Claude blog: building multi-agent systems](https://www.claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them) |
| 2 | Schema validation | [Zod: basic usage](https://zod.dev/basics) |
| 2 | Prior-auth standard | [Da Vinci PAS specification](https://hl7.org/fhir/us/davinci-pas/specification.html) |
| 3 | De-identification | [HHS: de-identification guidance](https://www.hhs.gov/hipaa/for-professionals/special-topics/de-identification/index.html) |
| 3 | LLM security risks | [OWASP Top 10 for LLM Applications (2023 PDF)](https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-2023-v1_1.pdf) |
| 3 | Testing | [Vitest: getting started](https://vitest.dev/guide/) |
| 3 | Evals in CI | [freeCodeCamp: AI evaluation handbook](https://www.freecodecamp.org/news/ai-evaluation-engineering-build-a-production-grade-llm-evaluation-platform-handbook/) |
| 4 | Durable workflows | [Cloudflare Workflows](https://developers.cloudflare.com/workflows/) |
| 4 | Human approval step | [Workflows example: waitForEvent](https://developers.cloudflare.com/workflows/examples/wait-for-event/) |
| 4 | Sending events | [Workflows: events and parameters](https://developers.cloudflare.com/workflows/build/events-and-parameters/) |
| 4 | Database | [D1: get started](https://developers.cloudflare.com/d1/get-started/) and [migrations](https://developers.cloudflare.com/d1/reference/migrations/) |
| 5 | API and middleware | [Hono docs](https://hono.dev/docs) |
| 5 | Claim format | [Da Vinci PAS: Claim profile](https://hl7.org/fhir/us/davinci-pas/en/StructureDefinition-profile-claim-base-definitions.html) |
| 6 | Deploy from CI | [Cloudflare: GitHub Actions](https://developers.cloudflare.com/workers/ci-cd/external-cicd/github-actions/) |
| 6 | Approval gates | [GitHub: managing environments](https://docs.github.com/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment) |
| 6 | Safe releases | [Gradual deployments](https://developers.cloudflare.com/workers/versions-and-deployments/gradual-deployments/) and [rollbacks](https://developers.cloudflare.com/workers/versions-and-deployments/rollbacks/) |
| 6 | Secret scanning | [Gitleaks Action](https://github.com/marketplace/actions/gitleaks) |
| 7 | Model access | [AI Gateway with Workers AI](https://developers.cloudflare.com/ai-gateway/integrations/aig-workers-ai-binding/) |
| 7 | Binding reference | [AI Gateway Workers binding](https://developers.cloudflare.com/ai-gateway/usage/worker-binding-methods/) |
| 7 | Data loss prevention | [AI Gateway DLP](https://developers.cloudflare.com/ai-gateway/features/dlp) |
| 7 | Data use and residency | [Workers AI data usage](https://developers.cloudflare.com/workers-ai/privacy/) and [Data Localization Suite](https://developers.cloudflare.com/data-localization/) |
| 7 | Cloud and HIPAA | [HHS: HIPAA and cloud computing](https://www.hhs.gov/hipaa/for-professionals/special-topics/cloud-computing) |
