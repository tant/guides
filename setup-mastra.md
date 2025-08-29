# 🏗️ Step-by-Step: Building Mastra Agents with Centralized LLM Configuration

## Overview

This comprehensive guide will walk you through creating a scalable Mastra agent system with centralized LLM configuration using the proven architecture from your current working codebase.

## Project Setup Using Mastra CLI

### Create Project Using Mastra CLI

Instead of manually creating the folder structure, use the Mastra CLI to set up your project. The CLI will create the project structure and initialize everything for you:

```bash
# Create a new Mastra project (this includes initialization)
pnpm create mastra@latest

# The CLI will ask you some questions:
# - Project name: Choose a name for your project (e.g., "my-mastra-agent")
# - Project structure: The CLI will create the basic structure for you
# - Dependencies: The CLI will install all required dependencies
# - Mastra initialization: The CLI will initialize Mastra automatically
```

Example CLI interaction:
```
┌   Mastra Create
│
◇  What do you want to name your project?
│  my-mastra-agent
│
◇  Project structure created
│
◇  pnpm dependencies installed
│
◇  mastra installed
│
◇  Mastra dependencies installed
│
◇  .gitignore added
│
└  Project created successfully


┌   Mastra Init
│
◇  Where should we create the Mastra files? (default: src/)
│  src/
│
◇  Select default provider:
│  OpenAI
│
◇  Enter your openai API key?
│  Skip for now
│
◇  Make your AI IDE into a Mastra expert? (installs Mastra docs MCP server)
│  VSCode
│
◑  Initializing MastraPPP WARN  1 deprecated subdependencies found: node-domexception @1.0.0
Already up to date
Progress: resolved 630, rProgress: resolved 630, reused 579, downloaded 0, added 0, done
◑  Initializing Mastra...
dependencies:
+ @mastra/loggers 0.10.9
◒  Initializing Mastra...
Done in 2.4s using pnpm v10.14.0
◇
│
◇   ─────────────────────────────────────────────────────────╮
│                                                            │
│        Mastra initialized successfully!                    │
│                                                            │
│        Add your OPENAI_API_KEY as an environment variable  │
│        in your .env file                                   │
│                                                            │
├────────────────────────────────────────────────────────────╯
```

### Open Project in Code Editor

After the CLI finishes, simply navigate to your project directory and open it in your code editor:

```bash
# Go to your project directory
cd my-mastra-agent

# Open the project in VS Code
code .
```

That's it! The Mastra CLI has already created the project structure and initialized everything for you. You're ready to start coding.

## Create Environment Configuration

### Create .env File

Create `.env` file in your project root:

```bash
# .env
# Your vLLM endpoint
VLLM_BASE_URL=http://yourip:yourport/v1
VLLM_API_KEY=your-api-key-here  # Optional if no authentication required

# Model names (change these to match your vLLM models)
GENERATE_MODEL=gpt-oss-20b
REASONING_MODEL=gpt-oss-20b
SMALL_GENERATE_MODEL=gpt-oss-20b

# Default timeout and retry settings
LLM_TIMEOUT_MS=30000
LLM_RETRIES=2
LLM_BACKOFF_MS=1000

# Database configuration
DATABASE_URL=file:./mastra.db
```

### Install dotenv

```bash
# If not already installed by Mastra CLI
pnpm add dotenv
```

## Create Centralized LLM Configuration

### What We're Building

We're creating a single configuration file that all agents will use:

```
src/mastra/llm/
└── config.ts    ← This file
```

### Create the Configuration File

Create `src/mastra/llm/config.ts`:

```typescript
/**
 * This file contains the centralized configuration for all LLM services.
 * Instead of each agent creating its own configuration, they all use this one.
 */

// First, let's define what our configuration looks like
export interface LLMConfig {
  // The URL of your vLLM server
  baseUrl: string;
  
  // API key for authentication (optional)
  apiKey?: string;
  
  // Different models for different tasks
  models: {
    generate: string;    // For content generation
    reasoning: string;   // For complex thinking
    small: string;       // For quick responses
  };
  
  // Default settings for all LLM calls
  default: {
    timeoutMs: number;   // How long to wait before giving up
    retries: number;     // How many times to retry on failure
    backoffMs: number;   // Wait time between retries
  };
}

/**
 * This is the default configuration that will be used by all agents.
 * It reads values from environment variables, with fallbacks.
 */
export const defaultLLMConfig: LLMConfig = {
  // Get the base URL from environment variable, or use localhost as fallback
  baseUrl: process.env.VLLM_BASE_URL || process.env.OPENAI_API_BASE_URL || 'http://localhost:8000/v1',
  
  // Get the API key from environment variable (can be undefined)
  apiKey: process.env.VLLM_API_KEY || process.env.OPENAI_API_KEY,
  
  // Model configuration - read from environment with fallbacks
  models: {
    generate: process.env.GENERATE_MODEL || 'gpt-oss-20b',
    reasoning: process.env.REASONING_MODEL || 'gpt-oss-20b',
    small: process.env.SMALL_GENERATE_MODEL || 'gpt-oss-20b'
  },
  
  // Default settings - read from environment with number parsing
  default: {
    timeoutMs: parseInt(process.env.LLLM_TIMEOUT_MS || '15000'),
    retries: parseInt(process.env.LLM_RETRIES || '1'),
    backoffMs: parseInt(process.env.LLM_BACKOFF_MS || '500')
  }
};
```

## Create Centralized LLM Provider

### What We're Building

We're creating a single provider that all agents will use:

```
src/mastra/llm/
├── config.ts     ← Already created
└── provider.ts   ← This file
```

### Create the Provider File

Create `src/mastra/llm/provider.ts`:

```typescript
/**
 * This file creates the centralized LLM provider.
 * Instead of each agent creating its own provider, they all use this one.
 */

// Import the OpenAI provider from Mastra (works with vLLM too!)
import { createOpenAI } from '@ai-sdk/openai';

// Import our configuration
import { defaultLLMConfig, type LLMConfig } from './config';

/**
 * This function creates an LLM provider with the given configuration.
 * Think of it as a factory that makes LLM providers.
 */
export const createLLMProvider = (config: Partial<LLMConfig> = {}) => {
  // Merge the provided config with our default config
  // If no config is provided, use the default
  const mergedConfig = { ...defaultLLMConfig, ...config };
  
  // Create and return the OpenAI-compatible provider
  // This works with vLLM because vLLM provides an OpenAI-compatible API
  return createOpenAI({
    baseURL: mergedConfig.baseUrl,    // The vLLM endpoint URL
    apiKey: mergedConfig.apiKey,      // API key (can be undefined)
  });
};

/**
 * This is our default provider instance.
 * All agents will use this unless they need something special.
 */
export const llmProvider = createLLMProvider();

/**
 * This function creates model instances from our provider.
 * For example: llmProviderFactory('gpt-oss-20b')
 */
export const llmProviderFactory = (modelName: string) => {
  return llmProvider(modelName);
};
```

## Enhance the Existing LLM Adapter

### What We're Building

We're improving the existing LLM adapter with better error handling and documentation:

```
src/mastra/llm/
├── config.ts     ← Already created
├── provider.ts   ← Already created
└── adapter.ts    ← This file (enhanced version)
```

### Update the Adapter File

Update `src/mastra/llm/adapter.ts` with enhanced features:

```typescript
/**
 * Enhanced LLM adapter with improved error handling and documentation.
 * This builds upon the existing working adapter with better features.
 */

// Import types from Mastra
import type { CoreMessage } from "@mastra/core";

// Import our provider factory (this should match your existing import)
import { llmProviderFactory } from "./provider"; // Or from your agent file

/**
 * Options for LLM calls with better defaults
 */
export type LLMCallOptions = {
  timeoutMs?: number;    // Custom timeout (default: 15000ms)
  retries?: number;      // Custom retry count (default: 1)
  backoffMs?: number;    // Custom backoff time (default: 500ms)
};

/**
 * Type guard to check if an object is a ProviderClient
 */
type ProviderClient = {
  generate?: (opts: { messages: CoreMessage[] }) => Promise<unknown>;
  doGenerate?: (opts: { messages: CoreMessage[] }) => Promise<unknown>;
  createCompletion?: (opts: { messages: CoreMessage[] }) => Promise<unknown>;
};

function isProviderClient(obj: unknown): obj is ProviderClient {
  if (!obj || typeof obj !== "object") return false;
  const r = obj as Record<string, unknown>;
  return (
    typeof r.generate === "function" ||
    typeof r.doGenerate === "function" ||
    typeof r.createCompletion === "function"
  );
}

/**
 * Type guard to check if an object is a FunctionClient
 */
type FunctionClient = (opts: unknown) => Promise<unknown>;

function isFunctionClient(obj: unknown): obj is FunctionClient {
  return typeof obj === "function";
}

/**
 * Extracts text from LLM responses
 * This handles different response formats from different models
 */
export function extractText(resp: unknown): string {
  if (!resp) return "";
  if (typeof resp === "string") return resp;
  if (typeof resp === "object") {
    const r = resp as Record<string, unknown>;
    if (typeof r.text === "string") return r.text;
    if (Array.isArray(r.data) && r.data.length > 0) {
      const d0 = r.data[0] as Record<string, unknown> | undefined;
      if (d0 && typeof d0.text === "string") return d0.text;
    }
    if (Array.isArray(r.choices) && r.choices.length > 0) {
      const c0 = r.choices[0] as Record<string, unknown> | undefined;
      if (c0 && typeof c0.text === "string") return c0.text;
      if (c0 && typeof c0.message === "object") {
        const m = c0.message as Record<string, unknown> | undefined;
        if (m && typeof m.content === "string") return m.content;
      }
    }
  }
  try {
    return JSON.stringify(resp);
  } catch {
    return String(resp);
  }
}

/**
 * Promise with timeout helper
 * This adds a timeout to any promise
 */
function timeoutPromise<T>(promise: Promise<T>, ms: number): Promise<T> {
  if (ms <= 0) return promise;
  return Promise.race([
    promise,
    new Promise<T>((_, reject) =>
      setTimeout(() => reject(new Error("LLM call timeout")), ms),
    ),
  ]);
}

/**
 * Main function to call LLM models with enhanced features
 * This is the improved version of your existing callModel function
 */
export async function callModel(
  modelName: string,              // Model name to call
  messages: CoreMessage[],        // The conversation messages
  opts: LLMCallOptions = {}       // Custom options
): Promise<{ text: string; raw: unknown }> {
  // Extract options with defaults (matching your existing logic)
  const { timeoutMs = 15000, retries = 1, backoffMs = 500 } = opts;
  
  // Get the client from our provider (using the same pattern as your code)
  const client = llmProviderFactory(modelName) as unknown;
  
  // Validate inputs (added for better error handling)
  if (!modelName || !messages.length) {
    throw new Error("Model name and messages are required");
  }

  // Retry logic (same as your existing code)
  let attempt = 0;
  while (attempt <= retries) {
    try {
      // This is the actual call to the LLM (same as your existing code)
      const run = async () => {
        if (isProviderClient(client)) {
          if (typeof client.generate === "function") {
            return client.generate({ messages });
          }
          if (typeof client.doGenerate === "function") {
            return client.doGenerate({ messages });
          }
          if (typeof client.createCompletion === "function") {
            return client.createCompletion({ messages });
          }
        }

        if (isFunctionClient(client)) {
          return (client as FunctionClient)({
            inputFormat: "messages",
            prompt: { messages },
          });
        }

        throw new Error(
          "Provider client does not expose a supported generate method",
        );
      };

      // Execute with timeout (same as your existing code)
      const resp = await timeoutPromise(run(), timeoutMs);
      return { text: extractText(resp), raw: resp };
    } catch (error) {
      attempt++;

      // If we've exhausted retries, throw the error (same as your existing code)
      if (attempt > retries) {
        throw error;
      }

      // Wait before retrying with exponential backoff (same as your existing code)
      await new Promise((resolve) => setTimeout(resolve, backoffMs * attempt));
    }
  }

  // This should never be reached due to the throw above, but TypeScript needs it
  throw new Error("Unexpected error in callModel");
}
```

## Create Centralized Exports

### What We're Building

We're creating a single point to export all LLM functionality:

```
src/mastra/llm/
├── config.ts     ← Already created
├── provider.ts   ← Already created
├── adapter.ts    ← Already created
└── index.ts      ← This file
```

### Create the Index File

Create `src/mastra/llm/index.ts`:

```typescript
/**
 * This file exports all LLM functionality for easy importing.
 * Instead of importing from multiple files, you can import from here.
 */

// Export the main adapter function
export { callModel } from './adapter';

// Export the provider functions
export { llmProviderFactory, createLLMProvider } from './provider';

// Export the configuration
export { defaultLLMConfig } from './config';

// Export the types
export type { LLMConfig } from './config';
export type { LLMCallOptions } from './adapter';
```

## Update Your Agent to Use Centralized LLM

### What We're Building

Update your existing agent to use the centralized LLM configuration:

```
src/mastra/agents/
└── your-existing-agent.ts    ← Update this file
```

### Update Your Agent File

Update your existing agent file (e.g., `src/mastra/agents/index.ts` or similar):

```typescript
/**
 * Updated agent using centralized LLM configuration
 */

// Import required Mastra components (keep your existing imports)
import { Agent } from "@mastra/core/agent";
// ... other imports

// Import our centralized LLM provider and configuration
import { llmProviderFactory } from '../llm/provider';
import { defaultLLMConfig } from '../llm/config';

// Keep your existing llmFactory if needed for backward compatibility
// export const llmFactory = llmProviderFactory;

// Update your agent configuration to use the centralized provider
export const yourAgent = new Agent({
  name: "Your Agent",
  instructions: "Your agent instructions here...",
  // Use the centralized provider with environment variable model
  model: llmProviderFactory(process.env.GENERATE_MODEL || 'gpt-oss-20b'),
  tools: {
    // ... your tools
  },
  // ... other configuration
});
```

## Create Test Scripts

### What We're Building

We're creating test scripts to verify everything works:

```
├── test-llm-connection.ts    ← This file
└── test-agent.ts            ← This file
```

### Create LLM Test Script

Create `test-llm-connection.ts` in your project root:

```typescript
/**
 * Test script to verify LLM connection works with enhanced adapter
 */

// Import dotenv to load environment variables
import dotenv from 'dotenv';
dotenv.config();

// Import our centralized LLM adapter
import { callModel } from './src/mastra/llm';

// Import Mastra types
import type { CoreMessage } from '@mastra/core';

async function testLLMConnection() {
  try {
    console.log('🧪 Testing centralized LLM configuration...');
    
    // Create test messages
    const messages: CoreMessage[] = [
      { role: 'user', content: 'Hello, how are you?' }
    ];
    
    // Test with environment variable model
    console.log('📝 Testing with environment model...');
    const modelName = process.env.GENERATE_MODEL || 'gpt-oss-20b';
    const response = await callModel(modelName, messages);
    console.log('✅ Model response:', response.text);
    
    // Test with custom options
    console.log('⚡ Testing with custom timeout...');
    const quickResponse = await callModel(
      modelName, 
      messages, 
      { timeoutMs: 5000, retries: 0 }
    );
    console.log('✅ Quick response:', quickResponse.text);
    
    console.log('🎉 All LLM connection tests passed!');
    
  } catch (error) {
    console.error('❌ LLM connection test failed:', error);
    process.exit(1);
  }
}

// Run the test
testLLMConnection();
```

### Create Agent Test Script

Create `test-agent.ts` in your project root:

```typescript
/**
 * Test script to verify agent works with centralized configuration
 */

// Import dotenv to load environment variables
import dotenv from 'dotenv';
dotenv.config();

// Import your agent (update import path as needed)
import { yourAgent } from './src/mastra/agents';

async function testAgent() {
  try {
    console.log('🤖 Testing agent with centralized LLM configuration...');
    
    // Test the agent
    const response = await yourAgent.generate("Hello, what can you help me with?");
    console.log('✅ Agent response:', response.text);
    
    console.log('🎉 Agent test passed!');
    
  } catch (error) {
    console.error('❌ Agent test failed:', error);
    process.exit(1);
  }
}

// Run the test
testAgent();
```

## Update Package.json Scripts

Update your `package.json` to add test scripts:

```json
{
  "scripts": {
    "test-llm": "npx ts-node test-llm-connection.ts",
    "test-agent": "npx ts-node test-agent.ts",
    "validate-env": "npx ts-node validate-env.ts"
  }
}
```

## Benefits of This Enhanced Architecture

### 🔧 Maintenance Benefits
- **One place to change** LLM configuration
- **Consistent behavior** across all agents
- **Easy to update** when you change vLLM servers
- **Simple to test** with mock configurations

### 🚀 Development Benefits
- **Faster development** - no need to configure LLM for each agent
- **Better collaboration** - team members use same configuration
- **Easier debugging** - consistent error handling
- **Scalable** - easy to add more agents

### 🛡️ Production Benefits
- **Centralized monitoring** - track all LLM usage in one place
- **Consistent performance** - same timeout/retry settings
- **Easy A/B testing** - switch models for all agents at once
- **Better security** - API keys in one place

## Troubleshooting Common Issues

### 1. "Connection Refused" Error
```bash
# Check if your vLLM server is running
curl http://yourip:yourport/v1/models

# Check network connectivity
ping yourip
```

### 2. "Model not found" Error
```bash
# List available models
curl http://yourip:yourport/v1/models

# Update your .env with correct model names
```

### 3. "Timeout" Errors
```bash
# Increase timeout values in .env
LLM_TIMEOUT_MS=60000
LLM_RETRIES=3
```

## Next Steps

1. **✅ Completed**: Set up centralized LLM configuration using proven architecture
2. **✅ Completed**: Enhanced existing adapter with better documentation
3. **✅ Completed**: Updated agents to use shared configuration
4. **Now**: Add custom tools to your agents
5. **Next**: Implement database storage for conversation history
6. **Later**: Add more agents with different personalities
7. **Finally**: Deploy to production environment

This architecture provides a solid foundation that builds upon your existing working code while adding the centralized configuration benefits you need for scalability.