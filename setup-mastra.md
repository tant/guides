# 🏗️ Step-by-Step: Building Mastra Agents with Centralized LLM Configuration

## Overview

This comprehensive guide will walk you through creating a scalable Mastra agent system with centralized LLM configuration. Even if you're new to Mastra, you'll be able to follow along and understand each step.



## Step 1: Understanding the Problem

### Why Centralized LLM Configuration?

**Current Problem:**
```
Agent 1 → Creates its own LLM provider
Agent 2 → Creates its own LLM provider  
Agent 3 → Creates its own LLM provider
```

**Problems with this approach:**
- Code duplication
- Hard to maintain (change config in 10 places)
- Inconsistent configurations
- Difficult to test

**Solution:**
```
All Agents → Shared LLM Provider ← Centralized Configuration
```

**Benefits:**
- One source of truth
- Easy maintenance
- Consistent behavior
- Better testability

## Step 2: Project Setup Using Mastra CLI

### 2.1 Create Project Using Mastra CLI

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

### 2.2 Open Project in Code Editor

After the CLI finishes, simply navigate to your project directory and open it in your code editor:

```bash
# Go to your project directory
cd my-mastra-agent

# Open the project in VS Code
code .
```

That's it! The Mastra CLI has already created the project structure and initialized everything for you. You're ready to start coding.

## Step 3: Create Environment Configuration

### 3.1 Create .env File

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

### 3.2 Install dotenv

```bash
npm install dotenv
```

## Step 4: Create Centralized LLM Configuration

### 4.1 What We're Building

We're creating a single configuration file that all agents will use:

```
src/mastra/llm/
└── config.ts    ← This file
```

### 4.2 Create the Configuration File

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
    timeoutMs: parseInt(process.env.LLM_TIMEOUT_MS || '15000'),
    retries: parseInt(process.env.LLM_RETRIES || '1'),
    backoffMs: parseInt(process.env.LLM_BACKOFF_MS || '500')
  }
};
```

### 4.3 What This Does

This configuration file:
1. **Defines the structure** of our LLM configuration
2. **Provides default values** from environment variables
3. **Has fallback values** if environment variables aren't set
4. **Is used by all agents** instead of each creating their own

## Step 5: Create Centralized LLM Provider

### 5.1 What We're Building

We're creating a single provider that all agents will use:

```
src/mastra/llm/
├── config.ts     ← Already created
└── provider.ts   ← This file
```

### 5.2 Create the Provider File

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

### 5.3 What This Does

This provider file:
1. **Creates a factory function** to make LLM providers
2. **Provides a default provider instance** for all agents
3. **Creates a model factory** to get specific models
4. **Uses our centralized configuration**

## Step 6: Create Enhanced LLM Adapter

### 6.1 What We're Building

We're creating an enhanced adapter with better error handling:

```
src/mastra/llm/
├── config.ts     ← Already created
├── provider.ts   ← Already created
└── adapter.ts    ← This file
```

### 6.2 Create the Adapter File

Create `src/mastra/llm/adapter.ts`:

```typescript
/**
 * This file contains an enhanced LLM adapter with better error handling,
 * timeout management, and retry logic.
 */

// Import types from Mastra
import type { CoreMessage } from "@mastra/core";

// Import our provider
import { llmProviderFactory } from "./provider";

// Import our configuration
import { defaultLLMConfig, type LLMConfig } from "./config";

/**
 * Options for LLM calls
 */
export type LLMCallOptions = {
  timeoutMs?: number;    // Custom timeout
  retries?: number;      // Custom retry count
  backoffMs?: number;    // Custom backoff time
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
 * This is what you'll use instead of calling the provider directly
 */
export async function callModel(
  messages: CoreMessage[],                              // The conversation messages
  taskType: 'generate' | 'reasoning' | 'small' | 'default' = 'default',  // Task type
  opts: Partial<LLMCallOptions> = {},                   // Custom options
  config: Partial<LLMConfig> = {}                       // Custom config
): Promise<{ text: string; raw: unknown }> {
  // Merge configuration
  const mergedConfig = { ...defaultLLMConfig, ...config };
  
  // Select the appropriate model based on task type
  const modelName = taskType === 'default' 
    ? mergedConfig.models.generate 
    : mergedConfig.models[taskType];
    
  // Extract options with defaults
  const { 
    timeoutMs = mergedConfig.default.timeoutMs, 
    retries = mergedConfig.default.retries, 
    backoffMs = mergedConfig.default.backoffMs 
  } = opts;
  
  // Get the client from our provider
  const client = llmProviderFactory(modelName) as unknown;

  // Validate inputs
  if (!modelName || !messages.length) {
    throw new Error("Model name and messages are required");
  }

  // Retry logic
  let attempt = 0;
  while (attempt <= retries) {
    try {
      // This is the actual call to the LLM
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

      // Execute with timeout
      const resp = await timeoutPromise(run(), timeoutMs);
      return { text: extractText(resp), raw: resp };
    } catch (error) {
      attempt++;

      // If we've exhausted retries, throw the error
      if (attempt > retries) {
        throw error;
      }

      // Wait before retrying with exponential backoff
      await new Promise((resolve) => setTimeout(resolve, backoffMs * attempt));
    }
  }

  // This should never be reached due to the throw above, but TypeScript needs it
  throw new Error("Unexpected error in callModel");
}
```

### 6.3 What This Does

This adapter file:
1. **Provides enhanced error handling** with timeouts and retries
2. **Handles different response formats** from various models
3. **Supports task-specific model selection**
4. **Uses our centralized configuration**
5. **Can be used by any agent** without duplication

## Step 7: Create Centralized Exports

### 7.1 What We're Building

We're creating a single point to export all LLM functionality:

```
src/mastra/llm/
├── config.ts     ← Already created
├── provider.ts   ← Already created
├── adapter.ts    ← Already created
└── index.ts      ← This file
```

### 7.2 Create the Index File

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

## Step 8: Create Your First Agent

### 8.1 What We're Building

We're creating an agent that uses our centralized LLM configuration:

```
src/mastra/agents/
├── my-first-agent.ts    ← This file
└── index.ts            ← Next file
```

### 8.2 Create the Agent File

Create `src/mastra/agents/my-first-agent.ts`:

```typescript
/**
 * This is your first Mastra agent using centralized LLM configuration.
 */

// Import Mastra components
import { Agent } from '@mastra/core/agent';
import { LibSQLStore } from '@mastra/libsql';
import { Memory } from '@mastra/memory';

// Import our centralized LLM provider and configuration
import { llmProviderFactory } from '../llm/provider';
import { defaultLLMConfig } from '../llm/config';

/**
 * Define the agent's personality and instructions
 * This tells the agent how to behave and what it can do
 */
const AGENT_PERSONALITY = `
# Helpful Assistant - Personality Profile

## Overall Character
You are a helpful, friendly, and knowledgeable assistant. You're like a tech-savvy friend who's always excited to help users with their questions.

## Communication Style
- **Tone**: Warm, approachable, and knowledgeable
- **Language**: Clear, conversational English
- **Pacing**: Not too rushed, allows users to absorb information

## Key Personality Traits
1. **Helpful & Supportive**: You actively listen to user concerns and show understanding
2. **Knowledgeable but Humble**: You're well-informed but don't boast
3. **Patient & Persistent**: You're patient with users who need more explanation time
4. **Professional & Tactful**: You maintain professional boundaries at all times

## Interaction Patterns
- Always respond to every user message
- Never ignore or skip any input
- Provide contextual responses that show you're listening
- Redirect inappropriate questions back to your main purpose politely
`;

/**
 * Create the agent instance
 * This uses our centralized LLM provider and configuration
 */
export const myFirstAgent = new Agent({
  name: 'Helpful Assistant',
  instructions: AGENT_PERSONALITY,
  model: llmProviderFactory(defaultLLMConfig.models.generate),
  tools: {
    // Add tools here later
  },
  memory: new Memory({
    storage: new LibSQLStore({
      url: process.env.DATABASE_URL || 'file:./mastra.db',
    }),
  }),
});
```

## Step 9: Create Agents Export

### 9.1 What We're Building

We're creating a single point to export all agents:

```
src/mastra/agents/
├── my-first-agent.ts    ← Already created
└── index.ts            ← This file
```

### 9.2 Create the Index File

Create `src/mastra/agents/index.ts`:

```typescript
/**
 * This file exports all agents for easy importing.
 */

// Export all agents
export { myFirstAgent } from './my-first-agent';
// Add more agents here as you create them
```

## Step 10: Create Test Scripts

### 10.1 What We're Building

We're creating test scripts to verify everything works:

```
├── test-llm-connection.ts    ← This file
└── test-agent.ts            ← This file
```

### 10.2 Create LLM Test Script

Create `test-llm-connection.ts` in your project root:

```typescript
/**
 * Test script to verify LLM connection works
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
    
    console.log('📝 Testing default model...');
    const response = await callModel(messages);
    console.log('✅ Default model response:', response.text);
    
    console.log('🧠 Testing reasoning model...');
    const reasoningResponse = await callModel(messages, 'reasoning');
    console.log('✅ Reasoning model response:', reasoningResponse.text);
    
    console.log('⚡ Testing small model...');
    const smallResponse = await callModel(messages, 'small', { timeoutMs: 5000 });
    console.log('✅ Small model response:', smallResponse.text);
    
    console.log('🎉 All LLM connection tests passed!');
    
  } catch (error) {
    console.error('❌ LLM connection test failed:', error);
    process.exit(1);
  }
}

// Run the test
testLLMConnection();
```

### 10.3 Create Agent Test Script

Create `test-agent.ts` in your project root:

```typescript
/**
 * Test script to verify agent works
 */

// Import dotenv to load environment variables
import dotenv from 'dotenv';
dotenv.config();

// Import our agent
import { myFirstAgent } from './src/mastra/agents';

async function testAgent() {
  try {
    console.log('🤖 Testing agent...');
    
    // Test the agent
    const response = await myFirstAgent.generate("Hello, what can you help me with?");
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

## Step 11: Update Package.json Scripts

Update your `package.json` to add test scripts:

```json
{
  "name": "your-mastra-project",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test-llm": "npx ts-node test-llm-connection.ts",
    "test-agent": "npx ts-node test-agent.ts",
    "build": "tsc",
    "start": "node dist/index.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "@ai-sdk/openai": "^0.0.0",
    "@mastra/core": "^0.0.0",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "ts-node": "^10.0.0",
    "typescript": "^4.0.0"
  }
}
```

## Step 12: Test Your Setup

### 12.1 Run LLM Connection Test

```bash
# Load environment variables and test LLM connection
npm run test-llm
```

You should see output like:
```
🧪 Testing centralized LLM configuration...
📝 Testing default model...
✅ Default model response: Hello! I'm doing well, thank you for asking!
🧠 Testing reasoning model...
✅ Reasoning model response: As an AI, I don't have feelings...
⚡ Testing small model...
✅ Small model response: Hello! I'm doing well, thanks for asking!
🎉 All LLM connection tests passed!
```

### 12.2 Run Agent Test

```bash
# Test your agent
npm run test-agent
```

You should see output like:
```
🤖 Testing agent...
✅ Agent response: Hello! I'm an AI assistant designed to help you...
🎉 Agent test passed!
```

## Step 13: Create Your Second Agent

### 13.1 Create Second Agent File

Create `src/mastra/agents/my-second-agent.ts`:

```typescript
/**
 * This is your second Mastra agent with a different personality
 */

// Import Mastra components
import { Agent } from '@mastra/core/agent';
import { LibSQLStore } from '@mastra/libsql';
import { Memory } from '@mastra/memory';

// Import our centralized LLM provider and configuration
import { llmProviderFactory } from '../llm/provider';
import { defaultLLMConfig } from '../llm/config';

/**
 * Define a different personality for this agent
 */
const TECH_EXPERT_PERSONALITY = `
# Tech Expert - Personality Profile

## Overall Character
You are a tech expert who specializes in helping users with technical questions.
You're knowledgeable, precise, and good at explaining complex topics simply.

## Communication Style
- **Tone**: Professional but approachable
- **Language**: Technical English with simple explanations
- **Focus**: Accuracy and clarity

## Key Skills
1. **Technical Knowledge**: Deep understanding of technology
2. **Problem Solving**: Good at breaking down complex problems
3. **Clear Explanations**: Can explain technical concepts simply
4. **Patient Guidance**: Helps users learn, not just gives answers

## Interaction Patterns
- Ask clarifying questions when needed
- Provide step-by-step instructions
- Include examples when helpful
- Verify understanding before moving on
`;

/**
 * Create the second agent instance
 */
export const mySecondAgent = new Agent({
  name: 'Tech Expert',
  instructions: TECH_EXPERT_PERSONALITY,
  model: llmProviderFactory(defaultLLMConfig.models.reasoning), // Use reasoning model
  tools: {
    // Add tools here later
  },
  memory: new Memory({
    storage: new LibSQLStore({
      url: process.env.DATABASE_URL || 'file:./mastra.db',
    }),
  }),
});
```

### 13.2 Update Agents Export

Update `src/mastra/agents/index.ts`:

```typescript
/**
 * This file exports all agents for easy importing.
 */

// Export all agents
export { myFirstAgent } from './my-first-agent';
export { mySecondAgent } from './my-second-agent'; // Add this line
```

### 13.3 Test Both Agents

Update `test-agent.ts`:

```typescript
/**
 * Test script to verify both agents work
 */

// Import dotenv to load environment variables
import dotenv from 'dotenv';
dotenv.config();

// Import our agents
import { myFirstAgent, mySecondAgent } from './src/mastra/agents';

async function testAgents() {
  try {
    console.log('🤖 Testing first agent...');
    const response1 = await myFirstAgent.generate("Hello, what can you help me with?");
    console.log('✅ First agent response:', response1.text);
    
    console.log('\n🤖 Testing second agent...');
    const response2 = await mySecondAgent.generate("Explain what vLLM is");
    console.log('✅ Second agent response:', response2.text);
    
    console.log('\n🎉 Both agents test passed!');
    
  } catch (error) {
    console.error('❌ Agents test failed:', error);
    process.exit(1);
  }
}

// Run the test
testAgents();
```

Run the test:
```bash
npm run test-agent
```

## Step 14: Advanced Usage Examples

### 14.1 Task-Specific Model Selection

```typescript
// In your application code
import { callModel } from './src/mastra/llm';
import type { CoreMessage } from '@mastra/core';

async function handleUserRequest() {
  const messages: CoreMessage[] = [
    { role: 'user', content: 'Explain quantum computing simply' }
  ];
  
  // For complex explanations, use reasoning model
  const complexResponse = await callModel(messages, 'reasoning');
  console.log('Complex response:', complexResponse.text);
  
  // For quick responses, use small model
  const quickMessages: CoreMessage[] = [
    { role: 'user', content: 'What time is it?' }
  ];
  const quickResponse = await callModel(quickMessages, 'small');
  console.log('Quick response:', quickResponse.text);
}
```

### 14.2 Custom Configuration Override

```typescript
// Override default configuration for special cases
import { callModel, defaultLLMConfig } from './src/mastra/llm';

async function handleHighPriorityRequest() {
  const customConfig = {
    ...defaultLLMConfig,
    default: {
      timeoutMs: 60000,  // Longer timeout
      retries: 3,        // More retries
      backoffMs: 2000    // Longer backoff
    }
  };
  
  const messages = [
    { role: 'user', content: 'Very complex technical question...' }
  ];
  
  const response = await callModel(messages, 'default', {}, customConfig);
  return response;
}
```

### 14.3 Error Handling with Fallbacks

```typescript
// Robust error handling with fallback models
import { callModel } from './src/mastra/llm';

async function robustLLMCall(messages: CoreMessage[]) {
  try {
    // Try with preferred model first
    console.log('Trying primary model...');
    return await callModel(messages, 'generate', { timeoutMs: 30000 });
  } catch (error) {
    console.warn('Primary model failed, trying fallback...', error.message);
    
    try {
      // Fallback to smaller model
      console.log('Trying fallback model...');
      return await callModel(messages, 'small', { timeoutMs: 15000 });
    } catch (fallbackError) {
      console.error('All models failed:', fallbackError.message);
      throw new Error('Unable to get LLM response after all retries');
    }
  }
}
```

## Step 15: Production Considerations

### 15.1 Environment Validation

Create `validate-env.ts`:

```typescript
/**
 * Validate environment configuration
 */

import dotenv from 'dotenv';
dotenv.config();

function validateEnvironment() {
  console.log('🔍 Validating environment configuration...');
  
  const required = ['VLLM_BASE_URL'];
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    console.error('❌ Missing required environment variables:', missing);
    console.log('Please check your .env file');
    process.exit(1);
  }
  
  console.log('✅ Environment validation passed:');
  console.log('   VLLM_BASE_URL:', process.env.VLLM_BASE_URL);
  console.log('   GENERATE_MODEL:', process.env.GENERATE_MODEL || 'gpt-oss-20b');
  console.log('   API Key:', process.env.VLLM_API_KEY ? 'Set' : 'Not set (may be optional)');
  
  return true;
}

// Run validation
validateEnvironment();
```

Add to package.json:
```json
{
  "scripts": {
    "validate-env": "npx ts-node validate-env.ts",
    "test-llm": "npm run validate-env && npx ts-node test-llm-connection.ts",
    "test-agent": "npm run validate-env && npx ts-node test-agent.ts"
  }
}
```

### 15.2 Monitoring and Logging

Update `src/mastra/llm/adapter.ts` to add logging:

```typescript
// Add logging to the callModel function
export async function callModel(/* ... */) {
  const startTime = Date.now();
  console.log(`🚀 Calling model: ${modelName} (attempt ${attempt + 1})`);
  
  try {
    const result = await /* ... existing logic ... */;
    const duration = Date.now() - startTime;
    console.log(`✅ Model call completed in ${duration}ms`);
    return result;
  } catch (error) {
    const duration = Date.now() - startTime;
    console.error(`❌ Model call failed after ${duration}ms:`, error.message);
    throw error;
  }
}
```

## Benefits of This Architecture

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
curl http://100.122.140.63:8000/v1/models

# Check network connectivity
ping 100.122.140.63
```

### 2. "Model not found" Error
```bash
# List available models
curl http://100.122.140.63:8000/v1/models

# Update your .env with correct model names
```

### 3. "Timeout" Errors
```bash
# Increase timeout values in .env
LLM_TIMEOUT_MS=60000
LLM_RETRIES=3
```

## Next Steps

1. **✅ Completed**: Set up centralized LLM configuration
2. **✅ Completed**: Created multiple agents using shared configuration
3. **✅ Completed**: Tested everything works
4. **Now**: Add custom tools to your agents
5. **Next**: Implement database storage for conversation history
6. **Later**: Add more agents with different personalities
7. **Finally**: Deploy to production environment

This architecture provides a solid foundation for building scalable Mastra applications with consistent, maintainable LLM configuration that any developer can understand and extend.
