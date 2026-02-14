# Factor 16: Prompt and Context Engineering

> Treat prompts as versioned software artifacts and context as a managed resource — with intentional design for prompt structure, context window utilization, RAG pipelines, and token budgeting.

## Motivation

Prompts are the most influential code in an AI application. A single word change in a system prompt can dramatically alter application behavior. Yet prompts are often treated as casual strings — edited ad hoc, untested, and deployed without review. Similarly, context window management — deciding what information to include in the model's context and how to structure it — is often an afterthought that becomes a scaling bottleneck.

Prompt engineering is software engineering. Context management is resource management. Both deserve the same rigor, tooling, and processes that we apply to traditional code.

## What This Replaces

**New — no direct predecessor.** The original 12/15-factor methodology had no concept of prompts or context windows, as these are unique to AI applications.

## How AI Changes This

This factor *is* the AI change. It addresses:

- **Prompt versioning and lifecycle**: Prompts are code that changes application behavior — they need version control, review, and testing.
- **Context window as a resource**: The context window is finite and expensive. What you include (and exclude) directly impacts quality, cost, and latency.
- **RAG pipeline design**: Retrieval-augmented generation is an architectural pattern that requires intentional design, not ad hoc assembly.
- **Token budgeting**: Every token in the context window has a cost and displaces other potential context. Budget allocation is an engineering decision.

## In Practice

### Prompt Architecture

Design prompts with clear structure and separation of concerns:

```markdown
# prompts/customer-support/system.md

## Role and Identity
You are a customer support assistant for {{company_name}}.
You help customers with questions about their accounts, orders, and products.

## Core Instructions
- Always be helpful, accurate, and concise.
- If you don't know the answer, say so — never fabricate information.
- Refer to the knowledge base context provided below for factual answers.

## Knowledge Base Context
{{retrieved_context}}

## Conversation Guidelines
- Greet returning customers by name.
- For billing questions, always verify the account first.
- For technical issues, gather system information before troubleshooting.

## Output Format
- Use clear, short paragraphs.
- Use bullet points for lists of steps.
- Include relevant links from the knowledge base when available.

## Safety Boundaries
- Never share other customers' information.
- Never process refunds over ${{refund_threshold}} without escalation.
- If the customer is upset, acknowledge their frustration before solving.
```

### Prompt Versioning

```
prompts/
├── customer-support/
│   ├── system.md                 # Current production prompt
│   ├── system.v2-candidate.md    # Under evaluation
│   ├── few-shot-examples.jsonl   # Few-shot examples
│   └── eval-config.yaml          # Evaluation configuration for this prompt
├── summarization/
│   ├── system.md
│   ├── chain-of-thought.md       # CoT template
│   └── eval-config.yaml
└── classification/
    ├── system.md
    └── labels.json                # Valid classification labels
```

### Context Window Budget

```yaml
# context-budget.yaml — allocate the context window intentionally
context_budget:
  model: claude-sonnet-4-5-20250929
  max_context_tokens: 200000
  max_output_tokens: 4096

  allocation:
    system_prompt:
      budget: 2000
      priority: 1              # Always included
      source: prompts/system.md

    safety_instructions:
      budget: 500
      priority: 1              # Always included
      source: prompts/safety.md

    few_shot_examples:
      budget: 3000
      priority: 2              # Included when budget allows
      source: prompts/few-shot-examples.jsonl
      strategy: select_most_relevant  # Pick examples similar to current query

    retrieved_context:
      budget: 8000
      priority: 2
      source: rag_pipeline
      strategy: rerank_and_truncate

    conversation_history:
      budget: 4000
      priority: 3              # Trimmed first when budget is tight
      strategy: keep_recent    # Keep most recent turns, summarize older ones

    user_message:
      budget: 2000
      priority: 1              # Always included in full

  overflow_strategy: trim_lowest_priority
  reserved_for_output: 4096    # Always reserve space for model output
```

### RAG Pipeline Design

```python
class RAGPipeline:
    """Structured retrieval-augmented generation pipeline."""

    def __init__(self, embedder, vector_store, reranker, llm):
        self.embedder = embedder
        self.vector_store = vector_store
        self.reranker = reranker
        self.llm = llm

    async def generate(self, query: str, context_budget: int) -> RAGResponse:
        # Stage 1: Query understanding
        search_queries = await self.expand_query(query)

        # Stage 2: Retrieval (cast a wide net)
        candidates = []
        for search_query in search_queries:
            embedding = await self.embedder.embed(search_query)
            results = await self.vector_store.query(
                vector=embedding,
                top_k=20,  # Retrieve more than needed
            )
            candidates.extend(results)

        # Stage 3: Deduplication
        candidates = self.deduplicate(candidates)

        # Stage 4: Reranking (narrow to most relevant)
        reranked = await self.reranker.rerank(
            query=query,
            documents=candidates,
            top_k=5,
        )

        # Stage 5: Context assembly (fit within budget)
        context = self.assemble_context(
            documents=reranked,
            token_budget=context_budget,
        )

        # Stage 6: Generation
        response = await self.llm.complete(
            system=self.system_prompt,
            context=context,
            query=query,
        )

        return RAGResponse(
            answer=response.text,
            sources=[doc.metadata for doc in context.documents],
            token_usage=response.token_usage,
        )

    def assemble_context(self, documents, token_budget):
        """Fit documents into the token budget, preserving the most relevant."""
        assembled = []
        tokens_used = 0

        for doc in documents:  # Already sorted by relevance
            doc_tokens = self.count_tokens(doc.content)
            if tokens_used + doc_tokens <= token_budget:
                assembled.append(doc)
                tokens_used += doc_tokens
            else:
                # Try to include a truncated version
                remaining = token_budget - tokens_used
                if remaining > 200:  # Minimum useful context
                    truncated = self.truncate_to_tokens(doc.content, remaining)
                    assembled.append(doc.with_content(truncated))
                break

        return ContextBundle(documents=assembled, total_tokens=tokens_used)
```

### Prompt Testing

```python
# Test prompts like code
class PromptTests:
    def test_system_prompt_within_budget(self):
        prompt = load_prompt("customer-support/system.md")
        tokens = count_tokens(prompt.render(default_vars))
        assert tokens <= CONTEXT_BUDGET["system_prompt"]

    def test_prompt_variables_all_defined(self):
        prompt = load_prompt("customer-support/system.md")
        required_vars = prompt.extract_variables()
        for var in required_vars:
            assert var in KNOWN_VARIABLES, f"Undefined variable: {var}"

    def test_few_shot_examples_valid(self):
        examples = load_examples("customer-support/few-shot-examples.jsonl")
        for example in examples:
            assert "input" in example
            assert "output" in example
            assert count_tokens(example["output"]) <= MAX_EXAMPLE_TOKENS

    async def test_prompt_produces_valid_output(self):
        prompt = load_prompt("customer-support/system.md")
        response = await llm.complete(prompt.render(test_vars))
        assert is_valid_format(response)
        assert not contains_pii(response)
```

### Prompt Optimization Strategies

- **Prompt compression**: Remove redundant instructions. Models understand concise prompts well.
- **Dynamic few-shot selection**: Don't include all examples — select the most relevant ones for the current query.
- **Conversation summarization**: Instead of passing full conversation history, summarize older turns.
- **Structured output over prose**: JSON schemas for output reduce tokens and improve reliability.
- **Chain-of-thought budgeting**: If using CoT, budget tokens for reasoning steps that won't be shown to the user.

## Compliance Checklist

- [ ] All prompts are stored in version control as named, reviewable files
- [ ] Prompt changes go through pull request review and evaluation (Factor 6)
- [ ] Context window utilization is budgeted with explicit allocations per component
- [ ] RAG pipelines have defined stages (retrieval, reranking, assembly) with quality metrics
- [ ] Token budgets account for system prompt, context, conversation history, and output reservation
- [ ] Few-shot examples are curated, versioned, and selected dynamically based on relevance
- [ ] Prompt templates are validated at build time (variables defined, within budget)
- [ ] Conversation history management has a defined strategy (truncation, summarization)
- [ ] Prompt effectiveness is measured through evaluations (Factor 6) and production monitoring (Factor 14)
- [ ] Context assembly strategies handle overflow gracefully (truncation, prioritization)
