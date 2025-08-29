# 🔄 Part 2: Decoupled Multi-Channel Architecture - Core Implementation

## Overview

This guide will walk you through creating a decoupled multi-channel architecture that keeps business logic centralized while making channel support clearly visible. This is Part 1 - the core implementation that applies to all channels.

## Current Structure Analysis

Let's first understand what we have:

```
src/
├── mastra/
│   ├── core/              ← Core business logic (KEEP THIS)
│   │   ├── models/        ← Message models
│   │   ├── processor/     ← Central message processor
│   │   └── index.ts
│   ├── llm/               ← LLM adapters (KEEP THIS)
│   │   ├── adapter.ts
│   │   ├── config.ts
│   │   ├── provider.ts
│   │   └── index.ts
│   ├── channels/          ← Base adapters (MOVE THESE)
│   │   ├── base-adapter.ts
│   │   └── index.ts
│   └── index.ts
└── (other files)
```

## Step 1: Restructure Channel Architecture

### 1.1 Create Decoupled Channel Structure

First, let's create the new structure for channels:

```bash
# Create the channels directory at project root
mkdir channels

# Create individual channel directories (examples)
mkdir -p channels/telegram/{src,config}
mkdir -p channels/whatsapp/{src,config}
mkdir -p channels/web/{src,config}
mkdir -p channels/line/{src,config}
```

This creates a clear structure:
```
channels/
├── telegram/      ← Clearly shows Telegram support
├── whatsapp/      ← Clearly shows WhatsApp support
├── web/           ← Clearly shows Web support
└── line/          ← Clearly shows LINE support
```

### 1.2 Move Base Adapter

Move the base adapter to a shared location:

```bash
# Move base adapter to a shared location
mkdir -p src/mastra/shared
mv src/mastra/channels/base-adapter.ts src/mastra/shared/
```

Create `src/mastra/shared/base-channel-adapter.ts`:

```typescript
/**
 * Base channel adapter that all channel adapters extend
 * This provides common functionality for all channels
 */

import { NormalizedMessage, ProcessedResponse, ChannelUser, ChannelContext } from '../core/models/message';
import { ChannelType, CHANNEL_CONFIGS } from '../core/models/channel';

export abstract class BaseChannelAdapter {
  protected channelId: string;
  protected channelType: ChannelType;
  protected config: typeof CHANNEL_CONFIGS[ChannelType];

  constructor(channelId: string, channelType: ChannelType) {
    this.channelId = channelId;
    this.channelType = channelType;
    this.config = CHANNEL_CONFIGS[channelType];
  }

  /**
   * Normalize channel-specific message to standardized format
   * Each channel must implement this method
   */
  protected abstract normalizeMessage(rawMessage: any): NormalizedMessage;

  /**
   * Send response back through the channel
   * Each channel must implement this method
   */
  protected abstract sendResponse(response: ProcessedResponse, originalMessage: NormalizedMessage): Promise<void>;

  /**
   * Handle incoming message from channel
   * This is the main entry point for all channel messages
   */
  async handleMessage(rawMessage: any): Promise<void> {
    try {
      // Normalize the message to our standard format
      const normalizedMessage = this.normalizeMessage(rawMessage);
      
      console.log(`📥 Received message from ${this.channelId}`, {
        messageId: normalizedMessage.id,
        sender: normalizedMessage.sender.id,
        channel: this.channelId
      });

      // At this point, the message is normalized and ready for central processing
      // The actual processing will be done by the injected processor
      // This method should be overridden by channels that need custom handling
      
    } catch (error) {
      console.error(`❌ Error in base handleMessage for ${this.channelId}:`, error);
      await this.sendErrorResponse(error, rawMessage);
    }
  }

  /**
   * Send error response - can be overridden by specific adapters
   */
  protected async sendErrorResponse(error: any, originalMessage: any): Promise<void> {
    console.warn(`⚠️  Error response not implemented for ${this.channelId}`);
  }

  /**
   * Helper method to create channel context
   */
  protected createChannelContext(
    channelMessageId?: string,
    threadId?: string,
    metadata: Record<string, any> = {}
  ): ChannelContext {
    return {
      channelId: this.channelId,
      channelMessageId,
      threadId,
      metadata
    };
  }

  /**
   * Helper method to create channel user
   */
  protected createChannelUser(
    id: string,
    username?: string,
    displayName?: string,
    phoneNumber?: string,
    email?: string
  ): ChannelUser {
    return {
      id,
      username,
      displayName,
      phoneNumber,
      email
    };
  }

  /**
   * Validate message before processing
   */
  protected validateMessage(message: NormalizedMessage): boolean {
    // Check if message content is not empty
    if (!message.content || message.content.trim().length === 0) {
      return false;
    }

    // Check if sender information is present
    if (!message.sender || !message.sender.id) {
      return false;
    }

    // Check message length against channel limits
    if (message.content.length > this.config.maxMessageLength) {
      return false;
    }

    return true;
  }
}
```

## Step 2: Update Core Message Processor

### 2.1 Enhance Central Message Processor

Update `src/mastra/core/processor/message-processor.ts`:

```typescript
/**
 * Central message processor that handles messages from all channels
 * This is where your business logic lives
 */

import { NormalizedMessage, ProcessedResponse } from '../models/message';
import { callModel } from '../../llm/adapter';
import { CHANNEL_CONFIGS } from '../models/channel';

export class CentralMessageProcessor {
  private processingQueue: Array<{
    message: NormalizedMessage;
    resolve: (response: ProcessedResponse) => void;
    reject: (error: Error) => void;
  }> = [];
  
  private isProcessing: boolean = false;

  /**
   * Process a normalized message and return a response
   * This is the main business logic entry point
   */
  async processMessage(message: NormalizedMessage): Promise<ProcessedResponse> {
    try {
      console.log(`🔄 Processing message from ${message.channel.channelId}`, {
        messageId: message.id,
        sender: message.sender.id,
        contentType: message.contentType
      });

      // Handle different content types
      switch (message.contentType) {
        case 'text':
          return await this.processTextMessage(message);
        case 'image':
          return await this.processImageMessage(message);
        case 'document':
          return await this.processDocumentMessage(message);
        default:
          return await this.processTextMessage(message); // Fallback to text processing
      }
    } catch (error) {
      console.error('❌ Error processing message:', error);
      return {
        content: 'Sorry, I encountered an error processing your message. Please try again.',
        contentType: 'text',
        metadata: {
          error: true,
          originalError: error instanceof Error ? error.message : String(error)
        }
      };
    }
  }

  /**
   * Process text messages using LLM
   */
  private async processTextMessage(message: NormalizedMessage): Promise<ProcessedResponse> {
    try {
      // Prepare context for the LLM
      const context = [
        {
          role: 'system',
          content: `You are a helpful assistant. The user is communicating via ${message.channel.channelId}.`
        },
        {
          role: 'user',
          content: message.content
        }
      ];

      // Get appropriate model based on message length and complexity
      const modelName = this.selectModelForMessage(message);
      
      // Call the LLM using your existing adapter
      const result = await callModel(modelName, context, {
        timeoutMs: 30000,
        retries: 2
      });

      return {
        content: result.text,
        contentType: 'text',
        metadata: {
          modelUsed: modelName,
          processingTime: Date.now() - (message.timestamp.getTime() || Date.now()),
          confidence: 'high'
        }
      };
    } catch (error) {
      throw new Error(`Failed to process text message: ${error instanceof Error ? error.message : String(error)}`);
    }
  }

  /**
   * Process image messages (placeholder - implement based on your needs)
   */
  private async processImageMessage(message: NormalizedMessage): Promise<ProcessedResponse> {
    // For image messages, you might want to:
    // 1. Download the image
    // 2. Process it with a vision model
    // 3. Extract text or analyze content
    // 4. Respond appropriately
    
    if (message.attachments && message.attachments.length > 0) {
      const imageUrl = message.attachments[0].url;
      
      // Example: Simple response for image messages
      return {
        content: `I received your image. I can see it's an image file. In the future, I'll be able to analyze images and extract information from them.`,
        contentType: 'text',
        attachments: [{
          url: imageUrl,
          type: 'image',
          filename: message.attachments[0].filename
        }],
        metadata: {
          imageProcessed: false,
          imageUrl: imageUrl
        }
      };
    }

    return {
      content: 'I received your image. Thank you for sharing!',
      contentType: 'text'
    };
  }

  /**
   * Process document messages (placeholder - implement based on your needs)
   */
  private async processDocumentMessage(message: NormalizedMessage): Promise<ProcessedResponse> {
    if (message.attachments && message.attachments.length > 0) {
      const docUrl = message.attachments[0].url;
      
      return {
        content: `I received your document: ${message.attachments[0].filename || 'unnamed file'}. I can see it's a document file. In the future, I'll be able to process documents and extract information from them.`,
        contentType: 'text',
        attachments: [{
          url: docUrl,
          type: 'document',
          filename: message.attachments[0].filename
        }],
        metadata: {
          documentProcessed: false,
          documentUrl: docUrl
        }
      };
    }

    return {
      content: 'I received your document. Thank you for sharing!',
      contentType: 'text'
    };
  }

  /**
   * Select appropriate model based on message characteristics
   */
  private selectModelForMessage(message: NormalizedMessage): string {
    const channelConfig = CHANNEL_CONFIGS[message.channel.channelId as keyof typeof CHANNEL_CONFIGS] 
      || CHANNEL_CONFIGS.web;

    // For complex queries or long messages, use reasoning model
    if (message.content.length > 500 || message.content.includes('?')) {
      return process.env.REASONING_MODEL || process.env.GENERATE_MODEL || 'gpt-oss-20b';
    }

    // For quick responses, use small model
    if (message.content.length < 50) {
      return process.env.SMALL_GENERATE_MODEL || process.env.GENERATE_MODEL || 'gpt-oss-20b';
    }

    // Default to generate model
    return process.env.GENERATE_MODEL || 'gpt-oss-20b';
  }

  /**
   * Queue message for processing (for high-volume scenarios)
   */
  async queueMessage(message: NormalizedMessage): Promise<ProcessedResponse> {
    return new Promise((resolve, reject) => {
      this.processingQueue.push({ message, resolve, reject });
      this.processQueue();
    });
  }

  /**
   * Process queued messages
   */
  private async processQueue(): Promise<void> {
    if (this.isProcessing || this.processingQueue.length === 0) {
      return;
    }

    this.isProcessing = true;

    while (this.processingQueue.length > 0) {
      const { message, resolve, reject } = this.processingQueue.shift()!;
      
      try {
        const response = await this.processMessage(message);
        resolve(response);
      } catch (error) {
        reject(error instanceof Error ? error : new Error(String(error)));
      }

      // Add small delay to prevent overwhelming the system
      await new Promise(resolve => setTimeout(resolve, 10));
    }

    this.isProcessing = false;
  }
}

// Export singleton instance
export const messageProcessor = new CentralMessageProcessor();
```

## Step 3: Create Channel Registry

### 3.1 Create Channel Management

Create `src/mastra/channels/registry.ts`:

```typescript
/**
 * Channel registry for managing active channels
 * This keeps track of which channels are currently active
 */

import { BaseChannelAdapter } from '../shared/base-channel-adapter';

export class ChannelRegistry {
  private adapters = new Map<string, BaseChannelAdapter>();
  
  /**
   * Register a channel adapter
   */
  register(channelId: string, adapter: BaseChannelAdapter): void {
    this.adapters.set(channelId, adapter);
    console.log(`✅ Registered channel: ${channelId}`);
  }
  
  /**
   * Get a channel adapter by ID
   */
  get(channelId: string): BaseChannelAdapter | undefined {
    return this.adapters.get(channelId);
  }
  
  /**
   * Remove a channel adapter
   */
  unregister(channelId: string): boolean {
    const removed = this.adapters.delete(channelId);
    if (removed) {
      console.log(`📤 Unregistered channel: ${channelId}`);
    }
    return removed;
  }
  
  /**
   * List all registered channels
   */
  listChannels(): string[] {
    return Array.from(this.adapters.keys());
  }
  
  /**
   * Check if a channel is registered
   */
  has(channelId: string): boolean {
    return this.adapters.has(channelId);
  }
  
  /**
   * Shutdown all channels gracefully
   */
  async shutdownAll(): Promise<void> {
    console.log('🛑 Shutting down all channels...');
    const shutdownPromises: Promise<void>[] = [];
    
    for (const [channelId, adapter] of this.adapters) {
      if (typeof (adapter as any).shutdown === 'function') {
        shutdownPromises.push((adapter as any).shutdown());
      }
    }
    
    await Promise.all(shutdownPromises);
    this.adapters.clear();
    console.log('✅ All channels shut down');
  }
}

// Export singleton instance
export const channelRegistry = new ChannelRegistry();
```

## Step 4: Create Main Application

### 4.1 Create Bootstrap Application

Create `src/main.ts`:

```typescript
/**
 * Main application entry point
 * This bootstraps all active channels with the central processor
 */

import dotenv from 'dotenv';
dotenv.config();

import { CentralMessageProcessor } from './mastra/core/processor/message-processor';
import { channelRegistry } from './mastra/channels/registry';

// Import channel adapters dynamically based on environment
async function loadChannels(processor: CentralMessageProcessor) {
  console.log('🚀 Loading channels...');

  // Load channels dynamically - this will be implemented by each channel
  // See specific channel guides for implementation details
  
  console.log(`✅ Loaded channels: ${channelRegistry.listChannels().join(', ') || 'None'}`);
}

// Graceful shutdown handler
async function shutdown() {
  console.log('🛑 Shutting down application...');
  try {
    await channelRegistry.shutdownAll();
    console.log('✅ Application shutdown complete');
    process.exit(0);
  } catch (error) {
    console.error('❌ Error during shutdown:', error);
    process.exit(1);
  }
}

// Main bootstrap function
async function bootstrap() {
  console.log('🚀 Starting multi-channel Mastra application...');

  // Create central message processor
  const processor = new CentralMessageProcessor();
  console.log('✅ Central message processor initialized');

  // Load configured channels
  await loadChannels(processor);

  // Set up signal handlers for graceful shutdown
  process.on('SIGINT', shutdown);
  process.on('SIGTERM', shutdown);

  console.log('🎉 Multi-channel application ready!');
  console.log(`📋 Active channels: ${channelRegistry.listChannels().join(', ') || 'None'}`);
}

// Start the application
bootstrap().catch(error => {
  console.error('❌ Failed to start application:', error);
  process.exit(1);
});

export { bootstrap, shutdown };
```

## Step 5: Update Environment Configuration

### 5.1 Update .env File

Update your `.env` file:

```bash
# .env
# Your vLLM endpoint
VLLM_BASE_URL=http://yourip:yourport/v1
VLLM_API_KEY=your-api-key-here

# Model names
GENERATE_MODEL=gpt-oss-20b
REASONING_MODEL=gpt-oss-20b
SMALL_GENERATE_MODEL=gpt-oss-20b

# Default timeout and retry settings
LLM_TIMEOUT_MS=30000
LLM_RETRIES=2
LLM_BACKOFF_MS=1000

# Database configuration
DATABASE_URL=file:./mastra.db

# Channel configurations
# Add channel-specific configurations as needed
# TELEGRAM_BOT_TOKEN=your-telegram-bot-token
# WHATSAPP_ACCESS_TOKEN=your-whatsapp-access-token
```

## Step 6: Update Package.json

### 6.1 Add Scripts and Dependencies

Update your `package.json`:

```json
{
  "name": "multi-channel-mastra",
  "version": "1.0.0",
  "description": "Multi-channel Mastra agent with decoupled architecture",
  "main": "dist/main.js",
  "scripts": {
    "dev": "ts-node src/main.ts",
    "build": "tsc",
    "start": "node dist/main.js",
    "test": "jest",
    "test:watch": "jest --watch"
  },
  "dependencies": {
    "@mastra/core": "^0.14.0",
    "@ai-sdk/openai": "^0.0.0",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "@types/node": "^18.0.0",
    "ts-node": "^10.0.0",
    "typescript": "^4.0.0"
  }
}
```

## Benefits of This Decoupled Architecture

### 🎯 **Clear Channel Visibility**
```
channels/
├── telegram/     ← Clearly shows Telegram support
├── whatsapp/     ← Clearly shows WhatsApp support
├── web/          ← Clearly shows Web support
└── line/         ← Clearly shows LINE support
```

### 🔧 **Easy Maintenance**
- **Business logic** in `src/mastra/core/processor/` (one place)
- **Channel adapters** in `channels/` (separate modules)
- **No redundant packages** - uses existing mastra structure

### 🚀 **Scalability**
- Add new channel: `mkdir channels/newchannel`
- Remove channel: `rm -rf channels/oldchannel`
- No impact on other channels

### 🛡️ **Reliability**
- Channel failure doesn't affect others
- Central processor handles all business logic
- Easy error isolation and debugging

### 🎨 **Developer Experience**
- Clear structure shows what's supported
- Each developer can focus on one channel
- Consistent patterns across channels
- Easy to onboard new team members

## Next Steps

1. **✅ Completed**: Restructured to decoupled channel architecture
2. **✅ Completed**: Kept business logic in central processor
3. **✅ Completed**: Made channel support clearly visible
4. **Now**: Implement specific channel adapters (see channel-specific guides)
5. **Next**: Test with actual channels
6. **Later**: Add more channels as needed
7. **Finally**: Deploy with monitoring and logging

This architecture provides a clean, maintainable, and scalable solution that makes it immediately clear which channels are supported while keeping all business logic centralized.