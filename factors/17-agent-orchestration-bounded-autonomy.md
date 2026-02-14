# Factor 17: Agent Orchestration and Bounded Autonomy

> Design AI agents with explicit capabilities, clear boundaries, execution budgets, and human-in-the-loop gates — orchestrated through well-defined patterns.

## Motivation

AI agents — systems that can plan, use tools, and take actions autonomously — represent the most powerful and most dangerous capability in AI applications. An agent that can browse the web, write code, send emails, and modify databases can accomplish in seconds what would take a human hours. It can also cause damage in seconds that takes days to undo.

The temptation is to give agents broad capabilities and rely on the model's judgment to use them wisely. This is the architectural equivalent of running everything as root. Bounded autonomy means agents have explicit, enforced limits on what they can do, how much they can spend, how long they can run, and when they must ask for human approval. These boundaries are architectural, not prompt-based — they're enforced by code, not by instructions the model might ignore.

> **Relationship with Factor 8 (Identity, Access, and Trust)**: Factor 8 defines *who* the agent is and *what* it's allowed to do — identity, permissions, and trust boundaries. This factor defines *how* the agent operates within those boundaries — orchestration patterns, execution budgets, checkpointing, and runtime guardrails. Factor 8 is the authorization model; Factor 17 is the execution model.

## What This Replaces

**New — no direct predecessor.** The original 12/15-factor methodology had no concept of autonomous AI agents, as these are a recent architectural pattern in AI applications.

## How AI Changes This

This factor *is* the AI change. It addresses:

- **Agent architecture patterns**: How to structure agents for reliability, observability, and control.
- **Tool permission management**: What tools an agent can use, with what parameters, and under what conditions.
- **Execution budgets**: Hard limits on time, tokens, cost, and actions per agent invocation.
- **Human-in-the-loop design**: When and how to involve humans in agent decision-making.
- **Multi-agent orchestration**: Patterns for coordinating multiple agents.

## In Practice

### Agent Architecture Patterns

**Pattern 1: Simple Tool-Use Agent**
Single agent with a fixed set of tools:

```
User → Agent → [Tool A, Tool B, Tool C] → Response
```

**Pattern 2: Router Agent**
A routing agent delegates to specialized sub-agents:

```
User → Router Agent → Specialist Agent A → Response
                   → Specialist Agent B → Response
```

**Pattern 3: Pipeline Agent**
Agents connected in a processing pipeline:

```
User → Research Agent → Analysis Agent → Writing Agent → Response
```

**Pattern 4: Hierarchical Agent**
A supervisor agent coordinates worker agents:

```
User → Supervisor Agent → Worker Agent 1 ─┐
                        → Worker Agent 2 ──┤→ Supervisor → Response
                        → Worker Agent 3 ─┘
```

### Bounded Autonomy Definition

```yaml
# agent-definition.yaml
agents:
  research-assistant:
    purpose: "Research topics using web search and knowledge base"
    model: claude-sonnet-4-5-20250929

    tools:
      - name: web_search
        permission: autonomous     # Can use freely
        rate_limit: 10/minute

      - name: knowledge_base_search
        permission: autonomous
        rate_limit: 20/minute

      - name: send_email
        permission: human_approval  # Must ask first
        approval_timeout_seconds: 300   # consistent with Factor 7 triggers and Factor 8 policy

      - name: write_to_database
        permission: denied          # Cannot use this tool

    execution_budget:
      max_steps: 25                 # Maximum tool calls per invocation
      max_tokens: 50000             # Maximum total tokens (input + output)
      max_cost_usd: 1.00            # Maximum cost per invocation (Factor 18 defines org-wide budget hierarchy)
      max_duration_seconds: 120     # Maximum wall-clock time
      max_retries: 3                # Maximum retries on failure

    safety:
      require_reasoning: true       # Agent must explain its plan before acting
      checkpoint_interval: 5        # Checkpoint state every 5 steps
      rollback_on_failure: true     # Revert partial actions if agent fails
```

### Execution Budget Enforcement

```python
class AgentExecutor:
    """Execute an agent with enforced boundaries."""

    def __init__(self, agent: AgentConfig, tools: ToolRegistry):
        self.agent = agent
        self.tools = tools
        self.budget = ExecutionBudget(agent.execution_budget)

    async def run(self, task: str) -> AgentResult:
        messages = [{"role": "user", "content": task}]
        steps = []

        while not self.budget.exhausted:
            # Get next action from model
            response = await self.llm.complete(
                messages=messages,
                tools=self.get_allowed_tools(),
            )

            if response.is_final_answer:
                return AgentResult(answer=response.content, steps=steps)

            if response.has_tool_call:
                tool_call = response.tool_call

                # Enforce tool permissions
                permission = self.check_tool_permission(tool_call)
                if permission == "denied":
                    messages.append(self.deny_message(tool_call))
                    continue
                if permission == "human_approval":
                    approved = await self.request_human_approval(tool_call)
                    if not approved:
                        messages.append(self.deny_message(tool_call))
                        continue

                # Execute tool with budget tracking
                self.budget.record_step()
                result = await self.tools.execute(tool_call)
                self.budget.record_tokens(response.token_usage)
                self.budget.record_cost(response.cost)

                steps.append(AgentStep(tool_call=tool_call, result=result))
                messages.append({"role": "tool", "content": result})

                # Checkpoint if interval reached
                if len(steps) % self.agent.safety.checkpoint_interval == 0:
                    await self.checkpoint(steps)

        # Budget exhausted
        return AgentResult(
            answer=None,
            steps=steps,
            status="budget_exhausted",
            budget_report=self.budget.report(),
        )
```

### Human-in-the-Loop Patterns

This factor defines the *execution mechanisms* for human approval. Factor 7 defines *when* approval is triggered (safety scores, confidence thresholds), and Factor 8 defines *who* is authorized to approve.

**Approval Gate**: Agent pauses and waits for human approval:

```python
class ApprovalGate:
    async def request_approval(self, agent_id: str, action: ToolCall) -> bool:
        # Send approval request to human
        request = ApprovalRequest(
            agent_id=agent_id,
            action=action.tool_name,
            parameters=action.parameters,
            reasoning=action.reasoning,
            timestamp=now(),
        )
        await self.notification_service.send(request)

        # Wait for response with timeout
        try:
            response = await asyncio.wait_for(
                self.approval_queue.get(request.id),
                timeout=self.agent.tools[action.tool_name].approval_timeout_seconds,
            )
            return response.approved
        except asyncio.TimeoutError:
            # Default deny on timeout
            return False
```

**Supervised Mode**: Human reviews every action before execution:

```python
class SupervisedExecutor:
    """Every agent action requires human confirmation."""

    async def execute_step(self, agent_id: str, tool_call: ToolCall) -> ToolResult:
        # Show the human what the agent wants to do
        await self.ui.show_pending_action(
            agent=agent_id,
            action=tool_call.tool_name,
            params=tool_call.parameters,
            reasoning=tool_call.reasoning,
        )

        # Wait for human decision
        decision = await self.ui.get_decision()  # approve / modify / reject / stop

        match decision:
            case "approve":
                return await self.tools.execute(tool_call)
            case "modify":
                modified = await self.ui.get_modified_params()
                return await self.tools.execute(tool_call.with_params(modified))
            case "reject":
                return ToolResult.rejected("Human rejected this action")
            case "stop":
                raise AgentStoppedByHuman()
```

### Multi-Agent Orchestration

```python
class SupervisorOrchestrator:
    """Supervisor pattern: one agent coordinates multiple specialists."""

    def __init__(self, supervisor: AgentConfig, workers: dict[str, AgentConfig]):
        self.supervisor = supervisor
        self.workers = workers

    async def run(self, task: str) -> OrchestratorResult:
        # Supervisor plans the work
        plan = await self.supervisor.plan(task)

        results = {}
        for step in plan.steps:
            worker = self.workers[step.agent]

            # Each worker has its own bounded execution
            result = await AgentExecutor(worker).run(step.task)

            if result.status != "success":
                # Supervisor handles worker failures
                recovery = await self.supervisor.handle_failure(step, result)
                if recovery.action == "retry":
                    result = await AgentExecutor(worker).run(recovery.revised_task)
                elif recovery.action == "skip":
                    continue
                elif recovery.action == "escalate":
                    result = await self.escalate_to_human(step, result)

            results[step.id] = result

        # Supervisor synthesizes final result
        return await self.supervisor.synthesize(results)
```

### Agent Observability

```json
{
  "trace_id": "agent-trace-123",
  "agent_id": "research-assistant",
  "task": "Research Q3 market trends",
  "status": "completed",
  "steps": 12,
  "budget": {
    "steps_used": 12,
    "steps_limit": 25,
    "tokens_used": 28500,
    "tokens_limit": 50000,
    "cost_usd": 0.42,
    "cost_limit_usd": 1.00,
    "duration_seconds": 45,
    "duration_limit_seconds": 120
  },
  "tools_used": {
    "web_search": 5,
    "knowledge_base_search": 7
  },
  "approvals_requested": 0,
  "errors": 0,
  "quality_score": 0.88
}
```

### Anti-Patterns
- **Unbounded agents**: Agents with no step limit, cost limit, or time limit. They can run indefinitely and spend without constraint.
- **Prompt-only boundaries**: "Don't use this tool unless necessary" in the prompt is not a boundary — it's a suggestion. Enforce boundaries in code.
- **God agents**: A single agent with every tool and permission. Prefer specialized agents with minimal tool sets.
- **No checkpoint/resume**: If an agent fails after 20 steps, it has to start over. Checkpoint state to enable resume.
- **Silent agents**: Agents that act without logging. Every tool call, every decision, every error should be traced.

## Compliance Checklist

- [ ] Every agent has a defined purpose, tool set, and execution budget
- [ ] Tool permissions are enforced architecturally (code), not just through prompts
- [ ] Execution budgets (steps, tokens, cost, time) are enforced with hard limits
- [ ] Human-in-the-loop gates exist for high-risk actions
- [ ] Agent actions are fully observable with distributed tracing (Factor 14)
- [ ] Multi-agent orchestration uses defined patterns (router, pipeline, supervisor)
- [ ] Agents checkpoint state periodically to enable resume after failures
- [ ] Failed agent actions can be rolled back where possible
- [ ] Agent identities and permissions follow Factor 8 (Identity, Access, Trust)
- [ ] Agent execution patterns and budget usage are monitored for optimization
