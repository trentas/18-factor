# Factor 7: Responsible AI by Design

> Build safety, fairness, and accountability into the architecture from the start — not as an afterthought or a compliance checkbox.

## Motivation

Traditional application security focuses on protecting the system from external threats: injection attacks, unauthorized access, data breaches. AI systems introduce a new class of risks that come from within the system itself. A model can generate harmful content, leak private information from its training data, produce biased outputs that discriminate against protected groups, or be manipulated through adversarial prompts.

These aren't bugs in the traditional sense — they're emergent properties of systems that learn from data and generate novel outputs. You can't fix them with a patch. They require architectural patterns: guardrails, monitoring, human oversight, and continuous evaluation. Responsible AI isn't a feature — it's a cross-cutting architectural concern, like security or observability.

## What This Replaces

**New — no direct predecessor.** The original 12/15-factor methodology predates the era where applications could generate harmful, biased, or misleading content as a normal part of their operation.

The closest analogue is security best practices, but responsible AI encompasses fairness, transparency, privacy, and safety concerns that go beyond traditional security.

## How AI Changes This

This factor *is* the AI change. It exists because AI systems can:

- **Generate harmful content**: Toxic, violent, sexual, or otherwise harmful outputs.
- **Leak private information**: Models can memorize and reproduce training data, including PII.
- **Produce biased outputs**: Systematic discrimination based on protected characteristics.
- **Be manipulated**: Prompt injection, jailbreaking, and adversarial inputs can subvert intended behavior.
- **Hallucinate**: Confidently state false information, including fabricated citations and non-existent URLs.
- **Act beyond intended scope**: Agents may take actions that exceed their intended authority.

## In Practice

### Safety Layers Architecture
Implement defense in depth — multiple layers, each catching different issues:

```
┌──────────────────────────────────────────────┐
│              INPUT GUARDRAILS                 │
│  Prompt injection detection                  │
│  Input validation and sanitization           │
│  PII detection and redaction                 │
│  Content policy pre-screening                │
├──────────────────────────────────────────────┤
│              MODEL LAYER                      │
│  System prompt with safety instructions      │
│  Constrained output schemas                  │
│  Temperature and sampling controls           │
├──────────────────────────────────────────────┤
│              OUTPUT GUARDRAILS               │
│  Content safety classification               │
│  PII detection in outputs                    │
│  Hallucination detection                     │
│  Fact-checking against sources               │
├──────────────────────────────────────────────┤
│              MONITORING LAYER                │
│  Safety metric tracking                      │
│  Bias detection and alerting                 │
│  Human review queues                         │
│  Incident response triggers                  │
└──────────────────────────────────────────────┘
```

### Prompt Injection Defense
Protect against attempts to override system instructions:

```python
# Multi-layer prompt injection defense
class InputGuardrail:
    def check(self, user_input: str) -> GuardrailResult:
        results = []

        # Layer 1: Pattern matching for known injection patterns
        results.append(self.pattern_detector.scan(user_input))

        # Layer 2: ML-based injection classifier
        results.append(self.injection_classifier.classify(user_input))

        # Layer 3: Input/output boundary enforcement
        results.append(self.boundary_enforcer.check(user_input))

        return GuardrailResult.aggregate(results)
```

### PII Handling
Detect and handle personally identifiable information at system boundaries:

```yaml
pii_policy:
  input:
    detection_enabled: true
    action: redact_and_log           # redact_and_log | block | allow_and_flag
    entity_types:
      - email
      - phone_number
      - ssn
      - credit_card
      - address
      - date_of_birth

  output:
    detection_enabled: true
    action: block                     # Stricter on output — never leak PII
    fallback_response: "I can't include personal information in my response."

  storage:
    redacted_in_logs: true
    redacted_in_traces: true
    retention_days: 30
```

### Bias Monitoring
Continuously monitor for systematic biases in AI outputs:

```python
# Bias monitoring across demographic dimensions
class BiasMonitor:
    def evaluate(self, requests, responses):
        # Segment by demographic indicators
        segments = self.segment_by_demographics(requests)

        for dimension in ["gender", "ethnicity", "age_group", "language"]:
            metrics = {}
            for segment_value, segment_data in segments[dimension].items():
                metrics[segment_value] = {
                    "quality_score": self.evaluate_quality(segment_data),
                    "refusal_rate": self.measure_refusal_rate(segment_data),
                    "response_length": self.measure_response_length(segment_data),
                    "sentiment": self.measure_sentiment(segment_data),
                }

            # Alert on significant disparities
            if self.detect_disparity(metrics):
                self.alert(dimension, metrics)
```

### Human-in-the-Loop Gates
Define clear criteria for when human oversight is required:

```yaml
human_review_triggers:
  # Content-based triggers
  - condition: safety_score < 0.7
    action: block_and_queue_for_review
    sla_minutes: 30

  # Action-based triggers
  - condition: agent_action in [delete, publish, send_email, financial_transaction]
    action: require_approval_before_execution
    sla_minutes: 5

  # Confidence-based triggers
  - condition: model_confidence < 0.5
    action: flag_for_review_before_serving
    sla_minutes: 15

  # Volume-based triggers
  - condition: user_requests_per_hour > 100
    action: sample_and_review
    sample_rate: 0.10
```

### Transparency and Explainability
Make AI decisions auditable:

- **Disclosure**: Clearly indicate when content is AI-generated.
- **Attribution**: When RAG is used, cite the source documents.
- **Reasoning traces**: Log the chain-of-thought or reasoning steps for important decisions.
- **Confidence indicators**: Surface confidence scores to users and downstream systems.
- **Appeal mechanisms**: Provide paths for users to contest AI decisions.

### Incident Response for AI
AI incidents require specific response procedures:

```yaml
ai_incident_playbook:
  severity_1_harmful_output:
    - immediately: disable_affected_feature
    - within_1h: identify_root_cause_and_affected_scope
    - within_4h: deploy_fix_or_guardrail
    - within_24h: conduct_retrospective
    - within_1w: add_to_eval_suite

  severity_2_bias_detected:
    - immediately: flag_for_investigation
    - within_24h: quantify_impact_and_scope
    - within_1w: implement_mitigation
    - ongoing: enhanced_monitoring
```

## Compliance Checklist

- [ ] Input guardrails detect and handle prompt injection attempts
- [ ] PII is detected at input and output boundaries with configurable handling policies
- [ ] Content safety classifiers screen AI outputs before serving to users
- [ ] Bias monitoring runs continuously across demographic dimensions
- [ ] Human-in-the-loop gates are defined for high-risk actions and low-confidence outputs
- [ ] AI-generated content is clearly disclosed to users
- [ ] RAG outputs include source attribution
- [ ] Reasoning traces are logged for auditability
- [ ] An AI incident response playbook exists and is practiced
- [ ] Safety evaluations are part of the CI pipeline (Factor 5) and production monitoring (Factor 14)
