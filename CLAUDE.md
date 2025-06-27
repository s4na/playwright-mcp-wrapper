# playwright-mcp-wrapper Codebase Summary for Claude

This file contains a concise summary of the playwright-mcp-wrapper codebase to provide context for Claude when working with this project.

## Project Overview

**playwright-mcp-wrapper** is a session-aware wrapper for Playwright MCP (Model Context Protocol) that solves the critical problem of browser context isolation when multiple Claude Code sessions interact with the same Playwright MCP instance.

### Problem Statement
- When multiple Claude Code sessions connect to a single Playwright MCP instance on the same port, browser contexts get mixed up between different conversations
- This leads to session interference and unpredictable behavior

### Solution
- A transparent HTTP proxy wrapper that automatically spawns isolated Playwright MCP instances per Claude Code session
- Each session gets its own dedicated Playwright MCP process with a unique port
- Zero additional user configuration required - just use `claude mcp add` as normal

## Architecture

### Core Components

1. **Wrapper MCP (Session Router)**
   - HTTP endpoint at `POST /mcp` (JSON-RPC) and `GET /mcp` (SSE)
   - Manages session lifecycle using `Mcp-Session-Id` header from Streamable-HTTP transport
   - Spawns Playwright MCP child processes on-demand
   - Routes requests to appropriate Playwright MCP instances based on session ID

2. **Session Registry**
   - In-memory map: `sessionId → {port, childPid, lastSeen}`
   - Tracks active sessions and their corresponding Playwright MCP processes

3. **Port Allocation Algorithm**
   - Deterministic port assignment: `20000 + (SHA256(sessionId)[0:4] % 20000)`
   - Range: 20000-39999 to avoid conflicts

### Request Flow

1. Claude Code sends initial `initialize` request
2. Wrapper generates UUID session ID if not present
3. Wrapper spawns Playwright MCP on calculated port
4. Wrapper returns `Mcp-Session-Id` header in response
5. Claude Code automatically includes this header in subsequent requests
6. Wrapper routes all requests to the appropriate Playwright MCP instance

### Lifecycle Management

- Processes are spawned lazily on first request
- Keep-alive: 60 seconds after last activity
- Clean shutdown: SIGTERM → SIGKILL sequence
- Automatic cleanup of idle sessions

## Technical Stack

- **Language**: TypeScript/Node.js
- **Web Framework**: Fastify
- **HTTP Proxy**: @fastify/http-proxy
- **Process Management**: Node.js child_process
- **Session ID**: UUID v4
- **Port Hashing**: SHA-256

## Security Considerations

- Binds only to 127.0.0.1 (localhost)
- Origin header validation for DNS rebinding protection
- No TLS required for local-only deployment
- For external exposure: Use Nginx + Mutual TLS

## Installation & Usage

```bash
# Install wrapper
npm i -g wrapper-mcp

# Start wrapper
wrapper-mcp --port 4000 &

# Register with Claude Code
claude mcp add --transport http browser http://127.0.0.1:4000/mcp
```

## Implementation Status

**Current State**: Design phase - no implementation code exists yet

### Next Steps for Implementation

1. **Core HTTP Server**
   - Set up Fastify server with proper middleware
   - Implement JSON-RPC and SSE endpoints
   - Add session ID header handling

2. **Session Management**
   - Create session registry with Map structure
   - Implement port calculation algorithm
   - Add session timeout logic

3. **Process Management**
   - Implement child process spawning for playwright-mcp
   - Add health checking and retry logic
   - Handle graceful shutdown

4. **Proxy Logic**
   - Set up HTTP proxy for JSON-RPC requests
   - Implement SSE streaming proxy
   - Add error handling and fallback

5. **Testing**
   - Unit tests for port calculation
   - Integration tests for session isolation
   - Load tests for concurrent sessions

## Key Design Decisions

1. **Why HTTP Transport?**
   - Simplest integration with Claude Code
   - Built-in session support via headers
   - No additional configuration needed

2. **Why Dynamic Port Allocation?**
   - Avoids port conflicts
   - Deterministic for debugging
   - Large enough range (20k ports) for practical use

3. **Why 60-second Timeout?**
   - Balances resource usage vs responsiveness
   - Allows for brief interruptions
   - Configurable for different use cases

## Dependencies

- `playwright-mcp`: The underlying browser automation MCP
- `fastify`: High-performance web framework
- `@fastify/http-proxy`: HTTP proxy plugin
- Standard Node.js libraries (crypto, child_process)

## Monitoring & Debugging

- Session registry provides real-time session visibility
- Child process PIDs tracked for system monitoring
- Structured logging recommended for production

This wrapper ensures clean session isolation for browser automation through Claude Code, making it safe to use Playwright MCP in multi-session environments without any additional user configuration.

## Claude Code Rules

- ルールを追加して欲しいとユーザーから要求された場合は、このCLAUDE.mdファイルにルールを追加すること