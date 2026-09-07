# Week 2 Day 1 — Agent Foundations: Short Write-up

## 1. Agent mental model and ReAct

A chatbot mainly generates a response to a user message, while a workflow follows a predefined sequence of steps. An agent is different because the model can choose what action to take next, call an external tool, observe the result, and decide whether another step is necessary.

The core loop is **Reason → Act → Observe → repeat**. The model receives the user request and available tool schemas. If a tool is needed, it emits a `tool_use` block with structured arguments. Python executes the selected tool and returns a `tool_result`. The model receives that observation and either calls another tool or produces the final answer. A maximum-iteration limit prevents an accidental infinite loop.

An agent is unnecessary when a task is deterministic and has no meaningful branching. A normal function, script, or fixed workflow is usually simpler, cheaper, and easier to test.

## 2. Tool schemas

Two tools were implemented:

- **`calculator`** — safely evaluates basic arithmetic from `expression: string`. It uses an AST allowlist rather than arbitrary `eval()`.
- **`get_weather`** — returns deterministic demo weather data from `city: string`. It is explicitly a learning stub, not a live weather API.

Tool descriptions matter because the model uses the name, description, and JSON schema to decide whether and how to call a tool. Precise descriptions reduce ambiguity, while strict schemas and application-side validation prevent malformed arguments from reaching the implementation.

## 3. Multi-step agent test

The agent was tested with:

> Look up the weather in Lahore and London using the weather tool. Then tell me which city is warmer and by how many degrees Celsius.

This requires at least two weather observations. After receiving both results, the model compares the temperatures and produces a final response.

The loop preserves the assistant response, sends tool results using the Anthropic `tool_result` content block, and stops after a configurable maximum number of iterations.

## 4. Memory and state

Conversation memory is the message history sent to the model. It contains user requests, assistant responses, and tool results.

Working memory is application-level state tracked during the current task. Examples include the iteration counter, tool observations, cached intermediate values, and completion flags. Keeping these concepts separate makes debugging and later framework migration easier.

The notebook logs each iteration, tool selection, arguments, observation, stop reason, and final answer.

## 5. Failure modes and guardrails

1. **Infinite/repeating loop:** bounded with `max_iterations`.
2. **Wrong arguments:** reduced with JSON schemas and server-side validation.
3. **Unknown tool:** handled by an allowlisted dispatcher that returns a structured error.
4. **Silent tool error:** failures are returned as structured results and marked with `is_error=True`.
5. **Unsafe execution:** the calculator uses an AST allowlist instead of `eval()`.
6. **Unavailable/stale data:** the weather tool explicitly reports when data is unavailable instead of inventing a value.

The deliberate failure test requests weather for `Atlantis`, which is not in the stub. The tool returns an error observation, allowing the model to explain the limitation rather than fabricate a temperature.

## 6. Why frameworks exist

LangChain, LangGraph, CrewAI, and similar frameworks exist because manually maintaining tool schemas, loops, state, retries, routing, persistence, tracing, approvals, and multi-agent coordination becomes repetitive as applications grow. The raw Python implementation exposes the primitives underneath those abstractions: messages, tools, state, control flow, observations, and guardrails.

## Conclusion

An agent is not magic. At its simplest, it is an LLM inside a controlled program loop that can select tools, receive observations, maintain state, and continue until a stopping condition is reached. Frameworks make this easier to build and operate, but understanding the raw loop makes their abstractions much easier to reason about.
