# Factor 5: Immutable Build Pipeline

> Build once, deploy everywhere — and extend the pipeline to compile prompts, pin model versions, and gate releases on evaluation results.

## Motivation

The original factor emphasized strict separation of build, release, and run stages with immutable releases. Every release is a unique, immutable artifact tagged with a timestamp or version. You never patch a running deployment — you build a new release and deploy it.

AI applications raise the stakes. A "build" now includes more than compiling code and bundling assets. It may include compiling prompt templates, resolving model version references, bundling evaluation datasets, and running quality gates that go beyond traditional tests. The non-deterministic nature of AI outputs makes the pipeline more complex: you can't just assert that a function returns the expected value — you need statistical evaluation of output quality.

## What This Replaces

**Original Factor #4 / Beyond 15 #4: Build, Release, Run** — "Strictly separate build and run stages."

This update retains the immutability principle and extends the pipeline to include:

- Prompt template compilation and validation
- Model version resolution and pinning
- Evaluation gates (quality, safety, cost) as CI checks
- AI artifact bundling alongside traditional build outputs

## How AI Changes This

### AI-Assisted Development
- AI can assist in pipeline authoring (generating CI configs, Dockerfiles, deployment manifests) but pipeline definitions must be reviewed and versioned like any other code (Factor 1).
- AI-assisted code review can be a stage in the pipeline — automated review comments, security scanning, and style checking.

### AI-Native Applications
- **Prompt compilation**: Prompt templates with variables, includes, and conditional sections are compiled into concrete prompts at build time. This catches template errors before runtime.
- **Model version pinning**: The build stage resolves model references (e.g., `claude-sonnet-latest`) to specific, immutable model versions and records them in the release manifest.
- **Evaluation gates**: Before a release is promoted, automated evaluations run against golden datasets. Releases that fail quality thresholds (accuracy, safety, cost) are blocked.
- **Artifact composition**: A release artifact includes the application binary, compiled prompts, model version manifest, evaluation results, and deployment configuration — all immutable and tied to a single version.

## In Practice

### The Extended Pipeline

```
┌─────────┐    ┌─────────┐    ┌──────────┐    ┌─────────┐    ┌─────┐
│  BUILD   │───▶│  EVAL   │───▶│ RELEASE  │───▶│ DEPLOY  │───▶│ RUN │
│          │    │         │    │          │    │         │    │     │
│ Compile  │    │ Quality │    │ Tag +    │    │ Roll out│    │Serve│
│ Test     │    │ Safety  │    │ Seal     │    │         │    │     │
│ Bundle   │    │ Cost    │    │          │    │         │    │     │
└─────────┘    └─────────┘    └──────────┘    └─────────┘    └─────┘
```

### Build Stage
The build stage produces a deterministic artifact:

```yaml
# ci-pipeline.yaml
build:
  steps:
    - name: Install dependencies
      run: pip install -r requirements.lock

    - name: Compile prompt templates
      run: |
        prompt-compiler compile \
          --input prompts/templates/ \
          --output dist/prompts/ \
          --validate-schemas

    - name: Resolve model versions
      run: |
        model-resolver resolve \
          --config ai-config.yaml \
          --output dist/model-manifest.json \
          --pin-versions

    - name: Run unit tests
      run: pytest tests/unit/

    - name: Build container image
      run: docker build -t app:$SHA .

    - name: Scan for vulnerabilities
      run: trivy image app:$SHA
```

### Evaluation Stage
Evaluations are the AI equivalent of integration tests — they gate the pipeline:

```yaml
evaluate:
  needs: build
  steps:
    - name: Run quality evaluations
      run: |
        eval-runner run \
          --model-manifest dist/model-manifest.json \
          --eval-suite evals/quality/ \
          --threshold accuracy=0.92

    - name: Run safety evaluations
      run: |
        eval-runner run \
          --eval-suite evals/safety/ \
          --threshold toxicity=0.01 \
          --threshold pii_leak=0.001

    - name: Run cost evaluations
      run: |
        eval-runner run \
          --eval-suite evals/cost/ \
          --threshold avg_cost_per_request=0.05 \
          --threshold p99_tokens=4096
```

### Release Manifest
The release artifact is immutable and self-describing:

```json
{
  "version": "2025.06.15-abc123f",
  "build_sha": "abc123f",
  "timestamp": "2025-06-15T14:30:00Z",
  "artifacts": {
    "container": "registry.example.com/app:abc123f",
    "prompts": "s3://releases/abc123f/prompts.tar.gz"
  },
  "models": {
    "summarization": "claude-sonnet-4-5-20250929@20250610",
    "embedding": "text-embedding-3-small@v2"
  },
  "evaluations": {
    "quality_accuracy": 0.94,
    "safety_toxicity": 0.003,
    "avg_cost_per_request": 0.032
  },
  "config_hash": "sha256:def456..."
}
```

### Immutability Rules
- **Never mutate a released artifact**. If a prompt needs changing, create a new release.
- **Never change a model version** in a running deployment. Change the configuration and trigger a new release.
- **Evaluation results are part of the release**. They're not just CI output — they're the evidence that this release meets quality standards.
- **Rollback is deployment of a previous release**, not manual surgery on a running system.

### Progressive Delivery
For AI applications, progressive delivery is essential because behavior changes are harder to predict:

- **Canary deployments**: Route a small percentage of traffic to the new release and monitor quality metrics.
- **Shadow deployments**: Run the new release alongside production, compare outputs, but don't serve the new outputs to users.
- **Feature flags for AI features**: Gate AI-powered features behind flags that can be toggled without redeployment.

## Compliance Checklist

- [ ] Build, evaluation, release, and run stages are strictly separated
- [ ] Build artifacts are immutable — never patched in place
- [ ] Each release has a unique, traceable version identifier
- [ ] Prompt templates are compiled and validated during the build stage
- [ ] Model versions are resolved and pinned at build time, recorded in the release manifest
- [ ] Automated evaluations gate the pipeline — failed evals block the release
- [ ] Evaluation results (quality, safety, cost) are recorded as part of the release metadata
- [ ] Container images are scanned for vulnerabilities before release
- [ ] Rollback is achieved by deploying a previous immutable release
- [ ] Progressive delivery mechanisms (canary, shadow) are used for AI feature releases
