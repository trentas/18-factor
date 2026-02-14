# Factor 1: Declarative Codebase

> Every artifact — application code, infrastructure, configuration, and AI prompts — lives in version control as a declarative, reproducible specification.

## Motivation

A codebase is the single source of truth for a system. In the original 12-Factor App, this meant "one codebase tracked in revision control, many deploys." That principle remains, but the scope of what constitutes a "codebase" has expanded dramatically. Infrastructure-as-Code (IaC), GitOps pipelines, and now prompt-as-code mean the declarative codebase encompasses far more than application source files.

When AI is both a tool used in development and a component of the running system, the boundary of "code" blurs. A system prompt is as much a part of the application's behavior as a controller or service class. If it isn't versioned, reviewed, and deployed through the same pipeline, it's shadow configuration — invisible, unauditable, and unreproducible.

## What This Replaces

**Original Factor #1: Codebase** — "One codebase tracked in revision control, many deploys."

The original factor focused on the relationship between a codebase and its deploys. This update retains that principle and extends it to cover:

- Infrastructure definitions (Terraform, Pulumi, CloudFormation)
- GitOps manifests (Kubernetes YAML, Helm charts, Kustomize overlays)
- AI prompts, system instructions, and agent tool definitions
- Evaluation datasets and quality benchmarks
- Pipeline definitions (CI/CD as code)

## How AI Changes This

### AI-Assisted Development
- AI coding assistants (Copilot, Claude Code, Cursor) generate code that must still pass the same review, lint, and test gates as human-written code. The codebase is the arbiter, not the generation method.
- AI-generated code should be indistinguishable from human-written code in the repository — no special markers or second-class treatment.
- Context files (`.cursorrules`, `CLAUDE.md`, `.github/copilot-instructions.md`) that guide AI coding assistants are themselves part of the codebase and should be versioned.

### AI-Native Applications
- **Prompt-as-code**: System prompts, few-shot examples, and chain-of-thought templates are versioned alongside application code. They go through pull requests, code review, and CI checks.
- **Agent tool schemas**: Tool definitions (JSON Schema, OpenAPI specs) that define what an AI agent can do are declarative specifications that belong in the codebase.
- **Evaluation datasets**: The golden datasets used to evaluate AI output quality are versioned artifacts, analogous to test fixtures.
- **Model configuration**: Model selection, temperature, token limits, and other inference parameters are declared in config files, not buried in application code.

## In Practice

### Repository Structure
Organize AI artifacts alongside traditional code:

```
repo/
├── src/                    # Application source
├── infra/                  # IaC definitions
├── k8s/                    # GitOps manifests
├── prompts/                # Versioned prompt templates
│   ├── system.md
│   ├── few-shot-examples/
│   └── chains/
├── evals/                  # Evaluation datasets and configs
│   ├── golden-set.jsonl
│   └── eval-config.yaml
├── tools/                  # Agent tool schemas
│   └── tool-definitions.json
├── .cursorrules            # AI coding assistant context
├── CLAUDE.md               # AI coding assistant context
└── pipeline.yaml           # CI/CD definition
```

### Prompt Versioning
Treat prompts as first-class code artifacts:
- Store prompts as template files with clear variable interpolation syntax.
- Use pull requests for prompt changes — diffs are meaningful and reviewable.
- Tag prompt versions that correspond to production deployments.
- Include prompt changes in changelog and release notes.

### GitOps for AI
- Declare model versions, endpoint configurations, and feature flags in Git.
- Use Git as the trigger for deployment — changes merged to `main` are automatically applied.
- Rollback is a `git revert`, not a manual infrastructure change.

### Monorepo vs. Multi-repo
The original factor's "one codebase, many deploys" still applies. Whether using a monorepo or multi-repo strategy, each deployable unit has a single codebase. Shared libraries are dependencies (Factor 3), not copy-pasted code.

## Compliance Checklist

- [ ] All application code is in version control with a clear branching strategy
- [ ] Infrastructure is defined as code (Terraform, Pulumi, CDK, etc.) and versioned
- [ ] Deployment manifests are declarative and versioned (Helm, Kustomize, etc.)
- [ ] CI/CD pipelines are defined as code, not configured through UIs
- [ ] AI system prompts and templates are versioned alongside application code
- [ ] Agent tool definitions and schemas are in the repository
- [ ] Evaluation datasets and benchmarks are versioned artifacts
- [ ] AI coding assistant context files are maintained and versioned
- [ ] Model configuration (selection, parameters) is declared in config files
- [ ] Every production deployment is traceable to a specific commit
