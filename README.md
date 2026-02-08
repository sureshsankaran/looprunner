# Loop Runner

A web-based interface for running continuous AI agent loops using OpenCode. Configure prompts, monitor command output, and manage working/persistent memory across iterations.

## What It Does

Loop Runner repeatedly prompts an AI model in a configurable loop, allowing you to:

- **Run continuous agent loops** - Execute prompts at regular intervals with automatic iteration tracking
- **Monitor external commands** - Watch log files or run shell commands, feeding output to the AI
- **Manage memory** - Use working memory (cleared each iteration) and persistent memory (maintained across loops)
- **Track history** - View all past iterations with prompts and responses

## Requirements

- [Bun](https://bun.sh/) runtime
- [OpenCode](https://github.com/anomalyco/opencode) CLI installed

## Installation

```bash
bun install
```

## Usage

Start the development server:

```bash
bun run dev
```

The web interface will be available at `http://localhost:3456`.

### Configuration Options

| Option | Description |
|--------|-------------|
| **Model** | Select the AI model/provider to use |
| **System Prompt** | Fixed instructions given to the model |
| **User Prompt** | The prompt sent each iteration |
| **Interval** | Milliseconds between iterations (minimum 1000) |
| **Max Iterations** | Stop after N iterations (0 = unlimited) |
| **Monitor Command** | Shell command to run periodically (e.g., `tail -n 20 /var/log/app.log`) |
| **Monitor Interval** | How often to run the monitor command |

### Memory Sections

- **Working Memory**: Temporary notes for the current iteration only. Cleared after each loop cycle.
- **Persistent Memory**: Long-term context maintained across all iterations. Use for goals, instructions, or accumulated knowledge.

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/events` | GET | SSE stream for real-time updates |
| `/api/state` | GET | Current loop state and config |
| `/api/models` | GET | Available AI models |
| `/api/config` | POST | Update loop configuration |
| `/api/start` | POST | Start the loop |
| `/api/stop` | POST | Stop the loop |
| `/api/working` | POST | Update working memory |
| `/api/persistent` | POST | Update persistent memory |
| `/api/history` | GET | Get iteration history |

## Running Tests

```bash
# Run E2E tests
bun run test:e2e

# Run E2E tests with UI
bun run test:e2e:ui
```

## Environment Variables

| Variable | Description |
|----------|-------------|
| `OPENCODE_EXECUTABLE` | Path to the OpenCode binary (optional, auto-detected) |

## Project Structure

```
looprunner/
├── src/
│   ├── server.ts    # Bun server with API endpoints
│   └── index.html   # Web interface
├── e2e/
│   └── loop-runner.spec.ts  # Playwright tests
├── playwright.config.ts
└── package.json
```

## Architecture Design

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Browser (Client)                            │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                     index.html (Web UI)                      │   │
│  │  - Configuration forms                                       │   │
│  │  - Real-time output display                                  │   │
│  │  - History viewer                                            │   │
│  └──────────────────────────┬──────────────────────────────────┘   │
└─────────────────────────────┼───────────────────────────────────────┘
                              │ HTTP/SSE
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Bun Server (server.ts)                         │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐   │
│  │  REST API    │  │  SSE Stream  │  │     Loop Controller     │   │
│  │  Endpoints   │  │  Broadcast   │  │  - Iteration mgmt       │   │
│  │              │  │              │  │  - Prompt building      │   │
│  └──────┬───────┘  └──────┬───────┘  │  - Session handling     │   │
│         │                 │          └───────────┬─────────────┘   │
│         └─────────────────┴──────────────────────┼─────────────────│
│                                                  │                  │
│  ┌───────────────────┐              ┌────────────┴────────────┐    │
│  │  Monitor Process  │              │    State Management     │    │
│  │  (Shell commands) │              │  - LoopConfig           │    │
│  └───────────────────┘              │  - LoopState            │    │
│                                     │  - History              │    │
│                                     └─────────────────────────┘    │
└─────────────────────────────┬───────────────────────────────────────┘
                              │ SDK Client
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   OpenCode Server (SDK)                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    Session Management                        │   │
│  │  - Create sessions                                           │   │
│  │  - Execute prompts                                           │   │
│  │  - Model provider routing                                    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      AI Model Providers                             │
│          (Anthropic, OpenAI, Google, etc.)                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Core Components

#### 1. Web Server (`server.ts`)

The Bun-based HTTP server handles:

- **Static file serving**: Serves the `index.html` web interface
- **REST API**: Handles configuration, control, and state endpoints
- **SSE (Server-Sent Events)**: Real-time updates to connected clients via the `/api/events` endpoint

#### 2. Loop Controller

The central orchestrator that manages the continuous AI loop:

```typescript
interface LoopState {
  running: boolean;      // Whether the loop is active
  iteration: number;     // Current iteration count
  sessionID: string;     // Current OpenCode session
  lastOutput: string;    // Most recent AI response
  monitorOutput: string; // Output from monitor command
  history: Array<...>;   // Past iteration records
}
```

**Loop Execution Flow:**
1. Create new OpenCode session
2. Build prompt (combining memories, monitor output, previous response)
3. Send prompt to AI model via OpenCode SDK
4. Extract and broadcast response
5. Clear working memory
6. Sleep for configured interval
7. Repeat until stopped or max iterations reached

#### 3. State Management

Two distinct data structures manage the application:

- **LoopConfig**: User-configurable settings (model, prompts, intervals, etc.)
- **LoopState**: Runtime state (iteration count, outputs, history)

#### 4. Monitor Process

An optional subprocess runner that:
- Executes shell commands at configurable intervals
- Captures stdout/stderr output
- Feeds results into the AI prompt context
- Runs independently from the main loop

#### 5. OpenCode Integration

Uses the `@opencode-ai/sdk` to:
- Spawn and manage the OpenCode server process
- Create isolated sessions for each iteration
- Route prompts to configured AI model providers
- Retrieve available models from all configured providers

### Data Flow

```
User Input (Config/Memory)
        │
        ▼
┌───────────────────┐
│   buildPrompt()   │ ◄── Persistent Memory
│                   │ ◄── Working Memory  
│                   │ ◄── Monitor Output
│                   │ ◄── Previous Response
│                   │ ◄── User Prompt
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│  OpenCode SDK     │
│  session.prompt() │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│   AI Response     │
└─────────┬─────────┘
          │
          ├──► State Update
          ├──► History Record
          ├──► SSE Broadcast
          └──► Clear Working Memory
```

### Real-Time Communication

The SSE (Server-Sent Events) system enables push-based updates:

```typescript
// Event types broadcast to clients:
{ type: "connected", state, config }  // Initial connection
{ type: "iteration", iteration }       // New iteration started
{ type: "prompt", prompt, iteration }  // Prompt sent to AI
{ type: "response", response, iteration } // AI response received
{ type: "monitor", output }            // Monitor command output
{ type: "config", config }             // Configuration changed
{ type: "state", state }               // State updated
{ type: "stopped" }                    // Loop stopped
{ type: "error", message }             // Error occurred
```

### Key Design Decisions

1. **Stateless Sessions**: Each iteration creates a fresh OpenCode session, preventing context accumulation issues

2. **Dual Memory Model**: Working memory (ephemeral) + Persistent memory (durable) allows both short-term and long-term context management

3. **SSE over WebSockets**: Simpler unidirectional push model sufficient for real-time updates

4. **Bun Runtime**: Chosen for native TypeScript support, fast startup, and built-in HTTP server

5. **AbortController Pattern**: Clean cancellation of async loops and monitor processes

## License

MIT
