# Contains Install Instructions to the MCP-RPC Projects and Details About the Approach

## Projects

https://github.com/jx-codes/mcp-rpc-runtime
https://github.com/jx-codes/mcp-rpc-bridge

## Background

This project emerged from exploring Cloudflare's "Code Mode" concept, which argues that LLMs are fundamentally better at writing regular code than making tool calls. Tool calls rely on synthetic tokens that aren't well-represented in training data, while code generation leverages the vast amount of programming examples LLMs have been trained on.

### The Problem with Traditional MCP

The Model Context Protocol (MCP) typically requires LLMs to make sequential, individual tool calls. This approach has several limitations:

- Sequential calls are slow and token-intensive
- Tool-calling tokens are synthetic and less familiar to LLMs
- Complex workflows require multiple round trips
- Context gets polluted with repetitive tool-call patterns

### The Code Mode Solution

Instead of calling MCP servers directly, Code Mode lets LLMs write regular TypeScript code in a secure sandbox. This enables:

- **Parallel operations** - kick off multiple requests concurrently with `Promise.all()`
- **Complex workflows** - map, filter, and transform data using familiar programming patterns
- **Better performance** - fewer tokens and round trips
- **Natural programming** - LLMs write code they've been extensively trained on

### Evolution to MCP-RPC

While implementing a Code Mode wrapper for MCP, a simpler pattern emerged: why translate MCP schemas into TypeScript APIs when we could just write TypeScript functions directly?

MCP-RPC takes this approach:

1. Write plain TypeScript functions with proper types
2. Functions are automatically discovered and exposed as RPC endpoints
3. Type information is extracted and provided to the LLM
4. The LLM writes normal TypeScript code against a typed client
5. Code runs in a secure Deno sandbox with controlled permissions

This approach treats MCP servers, databases, APIs, and any other integration as just another TypeScript function. The LLM doesn't need to know about protocols or schemas - it just calls functions with types, making the entire system more flexible and natural to use.

## Installation Instructions

Each repo contains an install.sh script (be sure to read the contents before running the scripts).

### RPC Server

```shell
curl -fsSL https://raw.githubusercontent.com/jx-codes/mcp-rpc-runtime/main/install.sh | bash
```

### Bridge

```shell
curl -fsSL https://raw.githubusercontent.com/jx-codes/mcp-rpc-bridge/main/install.sh | bash
```

## Configuration

After installing both the runtime and bridge, you'll need to configure your MCP client to use the bridge.

### Claude Code Configuration

Add this to your Claude Code MCP configuration file (typically `~/.config/claude/claude_desktop_config.json` or similar):

```json
{
  "mcpServers": {
    "codemode": {
      "command": "mcp-rpc-bridge",
      "env": {
        "RPC_WS_URL": "ws://localhost:8080/ws",
        "RPC_HTTP_URL": "http://localhost:8080"
      }
    }
  }
}
```

**Note:** The default URLs assume your RPC runtime is running on `localhost:8080`. Adjust these if you've configured different ports or remote connections.

### Starting the RPC Runtime

Before using the bridge, ensure the RPC runtime server is running:

```shell
mcp-rpc-runtime -r ./test-rpc -p 8080
```

This starts the WebSocket server that executes your TypeScript code in a secure Deno sandbox.

**Command Options:**

- `-r ./test-rpc` - Specifies the directory containing your RPC function files
- `-p 8080` - Sets the port for the WebSocket server (default: 8080)

## Available Tools

Once configured, Claude will have access to three main tools through the `codemode` MCP server:

### 1. `get_available_rpc_tools`

Discovers and retrieves all available RPC function definitions with their type signatures. This allows Claude to understand what functions it can call in its TypeScript code.

**Usage:** Claude will typically call this first to understand what functions are available to work with.

### 2. `run_script`

Executes TypeScript code in the secure Deno sandbox. This is where the magic happens - Claude writes regular TypeScript code using the `rpc` object to call functions, and the runtime executes it.

**Key Features:**

- Full TypeScript support with type checking
- Access to the `rpc` object for calling server-side functions
- Async/await support for handling promises
- Parallel execution with `Promise.all()` for performance
- Controlled permissions in Deno sandbox

**Example:** Claude can write code like:

```typescript
const results = await Promise.all([
  rpc.getUserData(123),
  rpc.getOrderHistory(123),
  rpc.getPreferences(123),
]);
```

### 3. `help`

Provides runtime status and usage examples. Use this to get quick guidance on how to structure code and what capabilities are available.

## How It Works

The typical workflow when interacting with Claude looks like this:

1. **Discovery:** Claude calls `get_available_rpc_tools` to see what functions are available
2. **Code Generation:** Claude writes TypeScript code using the discovered functions
3. **Execution:** Claude calls `run_script` to execute the code in the runtime
4. **Results:** The runtime returns results, which Claude uses to answer your question or complete the task

## Usage Examples

### Basic Usage

Simply ask Claude to perform tasks that would normally require multiple sequential operations:

```
"Fetch the user profile, their recent orders, and calculate their total spending"
```

Claude will automatically:

- Discover available RPC functions
- Write TypeScript code to fetch all data in parallel
- Execute the code
- Present the results

### Complex Data Processing

```
"Get all active users, filter those who haven't ordered in 30 days, and send them a reminder email"
```

Claude can write complex map/filter/reduce operations using familiar JavaScript patterns, all executed efficiently in the sandbox.

### Parallel Operations

```
"Check the status of these 10 API endpoints and report which ones are down"
```

Instead of 10 sequential tool calls, Claude writes a single TypeScript script with `Promise.all()` to check all endpoints concurrently.

## Advantages Over Traditional MCP

- **Performance:** Parallel operations are orders of magnitude faster than sequential tool calls
- **Token Efficiency:** One code execution vs. dozens of individual tool calls
- **Flexibility:** Full programming language capabilities (loops, conditions, data transformations)
- **Familiarity:** LLMs are extensively trained on JavaScript/TypeScript code
- **Type Safety:** Full TypeScript type checking catches errors before execution

## Architecture

```
┌─────────────┐         ┌──────────────┐         ┌───────────────────┐
│   Claude    │ ◄─────► │  MCP Bridge  │ ◄─────► │ MCP-RPC-Runtime   │
│   (Client)  │   MCP   │  (Protocol)  │   WS    │     (Deno)        │
└─────────────┘         └──────────────┘         └─────────┬─────────┘
                                                         spawns
                                                           |
                                                  ┌─────────────────────┐
                                                  │    LLM Sandbox      │
                                                  │  (Net Permissions)  │
                                                  └─────────────────────┘
                                                           (and)
                                                  ┌─────────────────────┐
                                                  │   RPC Functions     │
                                                  │  (All Permissions)  │
                                                  └─────────────────────┘
```

- **Claude Client:** Makes standard MCP protocol calls
- **MCP Bridge:** Translates between MCP and WebSocket RPC protocol
- **RPC Runtime:** Executes TypeScript in secure Deno sandbox with access to registered functions
- **TypeScript Functions:** Your actual business logic, automatically exposed as RPC endpoints

## Writing Your Own RPC Functions

To extend the system with your own functions, add TypeScript files to the directory you specify with the `-r` flag (e.g., `./test-rpc`). The runtime will automatically:

1. Discover exported functions from TypeScript files in that directory
2. Extract type information
3. Expose them through the RPC interface
4. Make them available to Claude through the `rpc` object

**Example structure:**

```
./test-rpc/
  ├── users.ts          # User management functions
  ├── orders.ts         # Order processing functions
  └── notifications.ts  # Notification functions
```

Each file should export typed functions that will become available as RPC endpoints.

## Troubleshooting

### Bridge can't connect to runtime

Ensure:

- The RPC runtime is running (`mcp-rpc-runtime -r ./test-rpc -p 8080`)
- URLs in your MCP config match the runtime's address and port
- No firewall blocking WebSocket connections on the specified port
- The RPC directory path (`-r` flag) exists and contains your function files

### TypeScript execution errors

Check:

- Function signatures match what's available (call `get_available_rpc_tools`)
- Async functions are properly awaited
- Required permissions are granted in Deno sandbox

### Claude not using code mode

Try being explicit:

- "Write a TypeScript script to..."
- "Use code mode to fetch..."
- "Execute code that will..."

## License

Check individual repositories for license information.
