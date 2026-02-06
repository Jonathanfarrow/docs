# minns-sdk

A high-performance TypeScript SDK for the Event Graph service. Designed for autonomous agents and LLM-powered applications with a focus on **minimal overhead**, **local intent extraction**, and **comprehensive memory management**.

## Core Features

- **Fluent Event Builder**: Easily construct complex events with rich context and causality.
- **Auto-Batching**: Group events into efficient batch requests based on interval or queue size.
- **Low-Latency Processing**: Enqueue events locally to keep your agent loop fast.
- **Sidecar Intent Parsing**: Local extraction of structured data from LLM responses with no network round-trips.
- **Defensive Design**: Built-in protection against circular references, dangerous keys, and oversized payloads.
- **Type Safe**: First-class TypeScript support with generics and `unknown` types for safe payloads.

---

## Installation

```bash
npm install minns-sdk
```

---

## Quick Start

### 1. Initialize the Client

```typescript
import { createClient } from 'minns-sdk';

const client = createClient({
  baseUrl: "https://your-event-graph.api",
  enableDefaultTelemetry: true, 
  defaultAsync: true,           // processEvent() returns local receipt by default
  autoBatch: true,              // Buffer events locally for network efficiency
  batchInterval: 100,           // Flush queue every 100ms
  batchMaxSize: 20,             // Or every 20 events
  maxQueueSize: 1000            // Maximum local queue depth
});
```

### 2. Build and Process Events (Fluent API)

```typescript
// send() waits for the server response
const response = await client.event("research-agent", { agentId: 123, sessionId: 456 })
  .action("search_database", { query: "quantum computing" })
  .outcome({ resultsFound: 10 }) // Requires action() earlier in chain
  .state({ mode: "strict" })     // Add runtime state variables
  .goal("Research quantum trends", 5) 
  .send();

// enqueue() returns a local acknowledgement immediately
const receipt = await client.event("research-agent")
  .observation("web_page", { target: "https://github.com" })
  .enqueue();
```

---

## API Reference

### `EventGraphDBClient`

#### Event Processing
- `event(agentType: string, config?: EventBuilderConfig): EventBuilder`: Start building a rich event.
- `processEvent(event: Event, options?): Promise<ProcessEventResponse>`: Submit a single event.
- `processEvents(events: Event[], options?): Promise<ProcessEventResponse>`: Submits multiple events; chunks into requests of up to `batchMaxSize`.
- `flush(): Promise<void>`: Manually trigger a flush of the auto-batch buffer. **Crucial for clean shutdowns.**

#### Memory & Knowledge
- `getAgentMemories(agentId: AgentId, limit?: number): Promise<MemoryResponse[]>`: Retrieve semantic memories for an agent.
- `getContextMemories(context: EventContext, request?): Promise<MemoryResponse[]>`: Find memories relevant to a specific context.
- `searchClaims(request: ClaimSearchRequest): Promise<ClaimSearchResponse[]>`: Semantic search over extracted knowledge/claims.

#### Strategies & Policy
- `getAgentStrategies(agentId: AgentId, limit?: number): Promise<StrategyResponse[]>`: Get learned behavioral patterns.
- `getSimilarStrategies(request: StrategySimilarityRequest): Promise<SimilarStrategyResponse[]>`: Find strategies similar to a given signature.
- `getActionSuggestions(contextHash: ContextHash, lastActionNode?: number, limit?: number): Promise<ActionSuggestionResponse[]>`: Get policy-guided next steps (Policy Guide).

#### Graph Exploration
- `getGraph(query?): Promise<GraphResponse>`: Get the full graph structure for visualization.
- `getGraphByContext(query: GraphContextQuery): Promise<GraphResponse>`: Get a context-anchored subgraph.
- `queryGraphNodes(request: GraphNodeQueryRequest): Promise<GraphNodeQueryResponse>`: Direct search for nodes by properties/filters.
- `traverseGraph(query: GraphTraverseQuery): Promise<GraphTraverseResponse>`: Navigate relationships from a starting node.

---

### `EventBuilder` (Fluent API)

- `action(name: string, params: Record<string, unknown>)`: Define an Action event.
- `outcome(result: unknown)`: Attach an outcome to the previously defined action. **Requires `action()` first.**
- `observation(type: string, data: unknown, options?)`: Define an Observation event. `data` is stored verbatim; the SDK does not validate payload fields.
- `context(text: string, type?)`: Define a Context event for claim extraction.
- `state(variables: Record<string, unknown>)`: Add environmental state (runtime variables).
- `goal(text, priority?, progress?)`: Add an active goal (Priority 1-5).
- `causedBy(parentId: string)`: Link to a previous event ID (causality).
- `build(): Event`: Return the raw Event object.
- `send(): Promise<ProcessEventResponse>`: Build and submit the event. Waits for server.
- `enqueue(): Promise<LocalAck>`: Build and queue for background processing. Returns receipt immediately.

---

## Advanced: LLM Sidecar Intent Parsing

The "Sidecar" pattern allows you to extract structured intents from LLM responses locally with no network round-trips. All sidecar utilities and types are exported from the main entry point.

```typescript
import { buildSidecarInstruction, extractIntentAndResponse } from 'minns-sdk';

// 1. Generate prompt instructions
const instruction = buildSidecarInstruction(spec);

// 2. Parse LLM output locally
const { intent, assistantResponse } = extractIntentAndResponse(modelOutput, userMsg, spec);
```

## Security & Resilience

The SDK is designed to be resilient against erratic LLM outputs and large data structures:
- **Circular Reference Protection**: Automatically detects and replaces circular objects with `"[Circular]"` in payloads.
- **Prototype Pollution Protection**: Strips dangerous JS keys (`__proto__`, `constructor`) during serialization.
- **Size Guarding**: Rejects payloads exceeding `maxPayloadSize` (default 1MB) checked **after** serialization.
- **Queue Management**: Local queue handles up to `maxQueueSize` (default 1000) events; `enqueue()` throws if exceeded.

## Telemetry

The SDK includes fire-and-forget telemetry to monitor performance and LLM token usage.
- **Collected Data**: Latency, status codes, estimated token counts, and error messages.
- **Privacy**: Telemetry sanitizes metadata and **never includes request bodies** or raw event content.
- **Opt-out**: Set `enableDefaultTelemetry: false` in the client configuration.

## License

MIT © 2026
