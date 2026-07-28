# Cloud Cost & Infrastructure Optimization Agent

> **Status: in progress** — planning is complete and implementation is underway, beginning with the AWS and Terraform foundations the rest of the system builds on.

An **agentic** tool that connects to an AWS account, scans for cost waste, reasons about what it finds, and **proposes** — but never applies — Terraform fixes. It produces two outputs from the same underlying findings: a technical report for engineers and a plain-English savings one-pager for a business or customer audience.

This is a project built to demonstrate the technical depth to design and build a real AWS agentic system, along with the communication range to translate that system's output into a business case.

## Why this project exists

Cloud cost waste (idle compute, unattached storage, fixed fleets that never scale down) is one of the most common and most fixable problems in any AWS account, and it sits at the exact intersection of technical infrastructure work and business value conversations. This project builds a tool that finds that waste automatically, explains it in language an engineer can act on and a buyer can understand, and does so safely: it can look at an account and suggest changes, but a human always stays in control of what actually happens.

## What it does

- Scans EC2, EBS, RDS, and Auto Scaling Groups for waste patterns: idle or over-provisioned instances, unattached EBS volumes, over-sized databases, fixed fleets with no autoscaling.
- Grounds every savings estimate in real AWS data such as Cost Explorer, Compute Optimizer, and the Pricing API, never in an LLM guess.
- Runs an LLM tool-calling agent loop to prioritize findings by impact and confidence and explain them in plain English.
- Generates Terraform remediations for each finding and runs `terraform plan` to produce a real, safe preview of the change — with no auto-apply.
- Produces two audience-specific outputs from the same structured findings: a technical report and a customer savings one-pager.

## Architecture

```
                          ┌───────────────────────────────────────────────┐
                          │            YOU (human-in-the-loop)             │
                          │   review proposals · run `terraform apply`     │
                          └───────────────▲───────────────────────────────┘
                                          │ proposals + reports
   ┌──────────────────┐   read-only   ┌───┴─────────────────────────────────────┐
   │  AWS SANDBOX      │◄──────────────│           THE AGENT (your app)          │
   │  (Terraform-      │  boto3/       │                                         │
   │   provisioned,    │  Describe*    │  ┌────────────┐   ┌──────────────────┐  │
   │   deliberately    │──────────────►│  │  SCANNER   │──►│  FINDINGS SCHEMA  │  │
   │   wasteful)       │  metrics,     │  │ (boto3 +   │   │  (normalized      │  │
   │                   │  cost data    │  │  Cost tools)│   │   objects)       │  │
   │  EC2 · EBS · RDS  │               │  └────────────┘   └────────┬─────────┘  │
   │  ASG · CloudWatch │               │                            ▼            │
   └──────────────────┘               │  ┌──────────────────────────────────┐   │
                                       │  │  ORCHESTRATOR + LLM (agent loop) │   │
                                       │  │  reason · call tools · prioritize│   │
                                       │  └───────────────┬──────────────────┘   │
                                       │                  ▼                       │
                                       │  ┌──────────────────────────────────┐   │
                                       │  │  REMEDIATION GENERATOR (Terraform)│   │
                                       │  └───────────────┬──────────────────┘   │
                                       │                  ▼                       │
                                       │  ┌───────────────┐   ┌───────────────┐   │
                                       │  │ TECH REPORT   │   │ CUSTOMER      │   │
                                       │  │ (engineers)   │   │ ONE-PAGER ($) │   │
                                       │  └───────────────┘   └───────────────┘   │
                                       │            served via DEMO UI (hosted)   │
                                       └──────────────────────────────────────────┘
```

Data flows left to right to down: the sandbox is read via read-only boto3 calls, raw resource state is normalized into a typed findings schema, the agent reasons over and prioritizes those findings, remediations are generated as Terraform, and two audience-specific outputs are produced and served through a hosted demo UI. Nothing flows back into the AWS account except through the human at the top of the diagram.

**Division of labor by design:** facts  (metrics, prices, savings dollars) come from deterministic Python calling AWS's own cost tools. Judgment (what matters, how to explain it, what to prioritize) comes from the LLM. Arithmetic is deliberately kept out of the model.

## Tech stack

- **Language:** Python
- **AWS SDK:** boto3, restricted to read-only `Describe*` / `Get*` / `List*` calls
- **AWS services in scope:** EC2, EBS, RDS, Auto Scaling Groups, CloudWatch (metrics and evidence), plus Cost Explorer, Compute Optimizer, Trusted Advisor, and the Pricing API for grounded cost data
- **IaC:** Terraform (HCL) both for the demo sandbox and for generated remediations
- **LLM orchestration:** a small, hand-written tool-calling agent loop (Claude API by default, written to be swappable rather than framework-locked)
- **Demo UI:** a thin Streamlit or FastAPI app, deployed to a free or low-cost host

## Key design decisions

- **Read-only, always.** The agent never gets write, modify, or delete permissions on the AWS account. This is enforced technically by a scoped IAM role limited to `Describe*`/`Get*`/`List*` actions plus cost-tool reads, not by convention or prompt instruction. There is no code path where the agent can mutate the account.
- **Human-in-the-loop by design.** The agent reads AWS state and proposes Terraform changes; it stops there. A human reviews the generated Terraform and the `terraform plan` preview, and decides whether to run `terraform apply`. This mirrors how real platform and SRE teams manage infrastructure change, and it's what makes the project safe to demo against a real or sandbox account.
- **Savings are grounded, never guessed.** Every dollar figure traces back to a real AWS data source. Cost Explorer for actual spend, Compute Optimizer for AWS's own rightsizing ecommendations, and the Pricing API for real unit prices. Each estimate is explicitly labeled "estimated" and states its assumptions (hours per month, region, on-demand pricing).
- **Terraform, not console clicks or live API calls, is the unit of change.** A generatedfix is a reviewable, version-controllable diff, paired with a `terraform plan` preview that shows exactly what would happen before anything does.
- **Facts from code, judgment from the model.** Keeping arithmetic deterministic and out of the LLM's hands means the numbers are auditable and testable independent of the model's behavior.

## The two outputs

**Technical report (for engineers):** each finding presented with its supporting evidence (the underlying metrics), the recommendation, the generated Terraform diff, the `terraform plan` output, and the estimated savings with assumptions spelled out. Built for someone who will act on it.

**Customer savings one-pager (for business audiences):** the same findings translated into plain, jargon-free language. A headline total estimated savings figure, a ranked summary of what was found and what it's costing, and business framing throughout (e.g. "you're paying roughly $X/month to run a dev database that's idle 90% of the time"). Built for someone who needs to understand the value of the fix, not the mechanics of it.

Both outputs render from the same structured findings, the difference is entirely in audience and language, not in the underlying facts.

## Project status

This project is under active, in-progress development. Planning is complete; implementation is proceeding phase by phase (sandbox and read-only setup, scanner and cost enrichment, the agentic core, remediation generation, the two report outputs, and finally a hosted demo UI). Expected completion is **late September 2026**. A live demo link, screenshots, and setup instructions will be added once the demo UI phase is complete.
