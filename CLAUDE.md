# CLAUDE.md

> This file is auto-loaded by Claude Code at the start of every session. It is the
> project's working memory. Read it fully before doing anything, and keep it up to date
> as the project evolves.

## Project: Cloud Cost & Infrastructure Optimization Agent

An **agentic** tool that connects to an AWS account, scans for cost waste, reasons about
what it finds, and **proposes** (never applies) Terraform fixes — then produces two
outputs: a technical report for engineers and a plain-English savings one-pager for a
business/customer audience.

**Why this project exists:** a portfolio project targeting cloud **solutions engineer /
cloud architect / cloud sales** roles. Design and communication choices should serve both
technical depth *and* business translation.

---

## Current status

- ✅ **Planning complete.** Concept, scope, stack, and a 10-week timeline are locked (see `docs/`).
- ▶️ **Next up: Phase 0 — Foundations & Setup** (read-only IAM role + a Terraform-provisioned,
  deliberately-wasteful sandbox + boto3 read "hello world"). See `docs/02_Project_Timeline.md`.
- Nothing has been built yet in this repo beyond docs.

Update this section as phases complete.

---

## Locked decisions (see docs/DECISIONS.md for rationale)

- **IaC tool:** **Terraform** (not CloudFormation).
- **Agent scope:** **read-only + human-in-the-loop.** The agent reads AWS state and
  *proposes* Terraform. It **never mutates the account.** A human reviews and runs
  `terraform apply`. This is enforced technically by a **read-only IAM role**, not by
  convention.
- **LLM layer:** **Claude API tool-calling** by default, written **swappable**. For current
  model IDs / pricing / limits, check https://docs.claude.com — do NOT hard-code a model or
  price from memory.
- **Savings must be grounded in real data:** dollar figures come from AWS **Cost Explorer**,
  **Compute Optimizer**, and the **Pricing API** — not LLM guesses. Every estimate carries its
  assumption (hours/month, region, on-demand) and is labeled "estimated."
- **Everything is IaC**, including the sandbox test environment itself.
- **Effort:** ~20 hrs/week, ~10-week plan.

---

## Tech stack

- **Language:** Python.
- **AWS SDK:** boto3 (read-only `Describe*`/`Get*`/`List*` calls only).
- **IaC:** Terraform (HCL).
- **AWS services in scope:** EC2, EBS, RDS, Auto Scaling Groups, CloudWatch (metrics/evidence),
  plus cost tooling: Cost Explorer, Compute Optimizer, Trusted Advisor (support-tier
  permitting), Pricing API. S3 lifecycle is an optional stretch.
- **LLM orchestration:** a small, hand-written tool-calling agent loop (prefer our own
  orchestrator over a heavy framework).
- **Demo UI:** thin Streamlit or FastAPI app, deployed to a free/cheap host (Streamlit Cloud /
  Render / Railway / Fly.io) in Phase 5.

---

## Architecture (summary — full detail in docs/01_Concepts_and_Process_Guide.md §5–6)

```
AWS sandbox (Terraform-provisioned, deliberately wasteful)
   │  read-only boto3 + cost APIs
   ▼
SCANNER  → normalizes to → FINDINGS SCHEMA (typed objects)
   ▼
ORCHESTRATOR + LLM (agent loop: reason · call tools · prioritize · explain)
   ▼
REMEDIATION GENERATOR → Terraform + `terraform plan` preview  (no apply)
   ▼
OUTPUTS:  technical report (engineers)  +  customer one-pager ($)
   ▼
DEMO UI (hosted link)  ──►  human reviews & runs `terraform apply`
```

**Division of labor to preserve:** *facts (metrics, dollars) come from deterministic Python;
judgment (prioritization, explanation) comes from the LLM.* Keep arithmetic out of the model.

---

## Planned repo layout (create as we go)

```
.
├── CLAUDE.md                  # this file
├── README.md                  # in-progress project readme (fleshed out in Phase 5)
├── docs/
│   ├── 01_Concepts_and_Process_Guide.md   # concepts + build walkthrough
│   ├── 02_Project_Timeline.md             # week-by-week plan
│   └── DECISIONS.md                        # decision log (ADR-style)
├── terraform/
│   ├── sandbox/               # the deliberately-wasteful demo environment (Phase 0)
│   └── remediations/          # generated fix snippets (Phase 3)
├── src/
│   ├── scanners/              # boto3 collectors → findings (Phase 1)
│   ├── cost/                  # Cost Explorer / Compute Optimizer / Pricing (Phase 1)
│   ├── schema.py              # the Finding data model (Phase 1)
│   ├── agent/                 # orchestrator + tool schemas + prompts (Phase 2)
│   ├── remediation/           # Terraform generation + `plan` (Phase 3)
│   └── reports/               # technical report + customer one-pager (Phase 4)
├── tests/                     # unit + integration tests vs the seeded sandbox
├── app/                       # demo UI (Phase 5)
├── .env.example               # names of required env vars (NEVER real secrets)
└── .gitignore
```

---

## Guardrails & conventions (do not violate)

- **Read-only always.** Never write, propose, or generate code that gives the agent
  write/delete/modify AWS permissions or that calls `terraform apply` automatically. Only
  `terraform plan` / `validate` run programmatically.
- **No secrets in git.** Credentials via env vars / local AWS profile only. Maintain
  `.env.example` and `.gitignore`. If a secret is about to be committed, stop and flag it.
- **Least privilege IAM:** scope the role to only the `Describe*`/`Get*`/`List*` actions we
  actually call, plus cost-tool reads.
- **Cost discipline:** the sandbox uses smallest/free-tier-friendly resource sizes; a billing
  alarm is set; `terraform destroy` the sandbox when idle for a while.
- **Findings are typed:** everything flows through the `Finding` schema (see Doc 1 §1.5). Don't
  pass around loose dicts once the schema exists.
- **Estimates are honest:** every dollar figure is "estimated," shows its assumption, and
  traces to a real source. Never invent numbers.
- **Test the risky parts:** scanners get unit tests against the known seeded waste (the
  "answer key"); generated Terraform must pass `terraform validate`.
- **Style:** clear names, small functions, docstrings that explain non-obvious AWS/agent
  behavior.

---

## Definition of done (per phase)

A phase is done only when: (1) the milestone artifact from `docs/02_Project_Timeline.md`
works and is demoable, and (2) code is committed with a tag/note so we always have a
fallback state to demo.

---

## Key references

- `docs/01_Concepts_and_Process_Guide.md` — concepts (agentic, tool-calling, AWS services,
  FinOps, Terraform) + component-by-component build explanation. Source of truth for the design.
- `docs/02_Project_Timeline.md` — the phased plan, hours, and per-phase milestones.
- `docs/DECISIONS.md` — why we chose what we chose.

---

<!-- Personal, machine-local instructions. -->
@~/.claude/aws-cost-agent-local.md
