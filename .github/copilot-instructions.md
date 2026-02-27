# Copilot Instructions for Figma Console MCP

## Build, Test, and Lint

```bash
# Install dependencies
npm install

# Build (all targets)
npm run build

# Build local mode only
npm run build:local

# Run development server (local mode)
npm run dev:local

# Run all tests
npm test

# Run single test file
npx jest tests/basic.test.ts

# Run single test by name
npx jest -t "test name pattern"

# Watch mode
npm run test:watch

# Coverage (requires 70% threshold)
npm run test:coverage

# Format (Biome)
npm run format

# Lint and fix
npm run lint:fix

# Type check
npm run type-check
```

## Architecture

This is an MCP (Model Context Protocol) server that bridges AI assistants with Figma.

### Two Deployment Modes

1. **Local Mode** (`src/local.ts`) — Full read/write via Desktop Bridge plugin. Uses stdio transport.
2. **Cloudflare Mode** (`src/index.ts`) — Read-only REST API. Deployed as Cloudflare Worker.

### Core Components

```
src/
├── local.ts              # Local MCP server entry point
├── index.ts              # Cloudflare Workers entry point
├── core/
│   ├── figma-tools.ts    # MCP tool definitions (main tool registration)
│   ├── figma-api.ts      # Figma REST API client
│   ├── websocket-server.ts    # WebSocket server for Desktop Bridge
│   ├── websocket-connector.ts # WebSocket client abstraction
│   ├── figma-desktop-connector.ts # Desktop Bridge communication
│   └── types/            # TypeScript definitions
├── apps/                 # MCP Apps (interactive UI components)
│   ├── token-browser/
│   └── design-system-dashboard/
└── browser/              # Browser automation (Puppeteer)

figma-desktop-bridge/     # Figma plugin (runs in Figma Desktop)
├── code.js               # Plugin main code (Figma Plugin API)
├── ui.html               # Plugin UI (WebSocket client)
└── manifest.json
```

### Data Flow

- **REST API tools**: MCP Server → Figma REST API → Response
- **Desktop Bridge tools**: MCP Server → WebSocket (:9223-9232) → Plugin UI → Plugin Code → Figma Plugin API

### Transport Detection

The server uses WebSocket transport via Desktop Bridge plugin on ports 9223–9232 (multi-instance support). All 56+ tools work through WebSocket when the plugin is connected.

## Key Conventions

### MCP Tool Registration Pattern

Tools are registered using the MCP SDK with Zod schemas:

```typescript
import { z } from "zod";
import type { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

server.tool(
    "tool_name",
    "Tool description",
    { param: z.string().describe("Parameter description") },
    async (args) => {
        // Implementation
        return {
            content: [{ type: "text", text: JSON.stringify(result) }]
        };
    }
);
```

### Response Format

All tools return MCP content structure:

```typescript
return {
    content: [{ type: "text", text: JSON.stringify(data) }]
};
```

For images:
```typescript
return {
    content: [{ type: "image", data: base64String, mimeType: "image/png" }]
};
```

### Logging

Use the child logger pattern:

```typescript
import { createChildLogger } from "./core/logger.js";
const logger = createChildLogger({ component: "my-component" });
logger.info("message");
```

### WebSocket Communication

Desktop Bridge commands use request/response correlation:

```typescript
const response = await connector.sendCommand({
    type: "command_type",
    // command-specific fields
});
```

### File Imports

Use `.js` extensions in imports (ESM):

```typescript
import { FigmaAPI } from "./figma-api.js";
```

### Environment Variables

- `FIGMA_ACCESS_TOKEN` — Personal Access Token for Figma API
- `FIGMA_WS_PORT` — Override WebSocket port (default: 9223)
- `FIGMA_WS_HOST` — WebSocket bind address (default: localhost)
- `ENABLE_MCP_APPS` — Enable interactive MCP Apps (default: false)

### Testing

- Jest with ts-jest preset
- Tests in `tests/` directory and `src/**/*.test.ts`
- Coverage threshold: 70% (branches, functions, lines, statements)
- Use `forceExit: true` due to WebSocket event listeners
