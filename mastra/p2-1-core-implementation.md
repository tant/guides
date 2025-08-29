# 🔄 Part 2: Decoupled Multi-Channel Architecture - Core Implementation

## Overview

This guide will walk you through creating a decoupled multi-channel architecture that keeps business logic centralized while making channel support clearly visible. This is Part 1 - the core implementation for a new project.

## Project Structure Overview

Before diving into implementation, let's understand the clean architecture we're building:

```
src/
├── mastra/
│   ├── core/              ← Central business logic and message processing
│   │   ├── models/        ← Standardized message formats
│   │   ├── processor/      ← Central message processor (business logic)
│   │   └── index.ts
│   ├── llm/               ← LLM integration and adapters
│   │   ├── adapter.ts
│   │   ├── config.ts
│   │   ├── provider.ts
│   │   └── index.ts
│   └── index.ts
├── channels/               ← Each channel as independent module
│   ├── telegram/
│   │   ├── adapter.ts     ← Telegram-specific integration
│   │   ├── config.ts      ← Telegram configuration
│   │   └── index.ts
│   ├── whatsapp/
│   │   ├── adapter.ts     ← WhatsApp-specific integration
│   │   ├── config.ts      ← WhatsApp configuration
│   │   └── index.ts
│   └── web/
│       ├── adapter.ts     ← Web-specific integration
│       ├── config.ts      ← Web configuration
│       └── index.ts
└── main.ts                ← Application entry point
```

### Key Design Principles

1. **Centralized Business Logic**: All message processing logic lives in `src/mastra/core/processor/`
2. **Decoupled Channels**: Each channel is an independent module in `src/channels/`
3. **Standardized Interface**: All channels use the same message format
4. **Clear Visibility**: Immediately obvious which channels are supported

## Why This Architecture?

### The Problem with Monolithic Approach
```
❌ Bad: Everything mixed together
src/
├── telegram-handler.ts
├── whatsapp-handler.ts
├── web-handler.ts
├── business-logic.ts
└── message-processor.ts
```

### The Solution: Clear Separation
```
✅ Good: Clean separation of concerns
src/
├── mastra/core/processor/     ← Business logic (ONE PLACE)
└── channels/                 ← Channel integrations (MANY PLACES)
    ├── telegram/            ← Only Telegram integration code
    ├── whatsapp/            ← Only WhatsApp integration code
    └── web/                 ← Only Web integration code
```

## Step 1: Create Project Structure

### 1.1 Initialize Clean Project Structure

Let's create the foundational project structure:

```bash
# Create the main project structure
mkdir -p src/mastra/{core,llm}
mkdir -p src/mastra/core/{models,processor}
mkdir -p src/channels/{telegram,whatsapp,web,line}
mkdir -p src/channels/telegram/{config,tests}
mkdir -p src/channels/whatsapp/{config,tests}
mkdir -p src/channels/web/{config,tests}
mkdir -p src/channels/line/{config,tests}
```

This creates a clear, immediately understandable structure:
```
src/
├── mastra/
│   ├── core/              ← ONE PLACE for all business logic
│   │   ├── models/        ← Standardized message formats
│   │   └── processor/    ← Central message processing
│   └── llm/               ← LLM integration
└── channels/              ← EACH CHANNEL as independent module
    ├── telegram/         ← Only Telegram-related code
    ├── whatsapp/        ← Only WhatsApp-related code
    ├── web/              ← Only Web-related code
    └── line/             ← Only LINE-related code
```

## Step 2: Create Message Models

### 2.1 Define Standardized Message Formats

Create `src/mastra/core/models/message.ts`:

```typescript
/**
 * Standardized message models for multi-channel processing
 * This is the common language ALL channels speak
 */

export interface ChannelUser {
  id: string;           // Channel-specific user ID
  username?: string;    // Username if available
  displayName?: string; // Display name
  phoneNumber?: string; // Phone number if available
  email?: string;       // Email if available
}

export interface ChannelContext {
  channelId: string;        // Channel identifier (telegram, whatsapp, web, etc.)
  channelMessageId?: string; // Channel-specific message ID
  threadId?: string;        // Conversation thread ID if supported
  metadata: Record<string, any>; // Channel-specific metadata
}

export interface NormalizedMessage {
  id: string;                    // Unique message ID
  content: string;              // Message content
  contentType: 'text' | 'image' | 'document' | 'audio' | 'video' | 'location' | 'contact';
  sender: ChannelUser;          // Sender information
  timestamp: Date;              // When message was sent
  channel: ChannelContext;     // Channel context
  replyToMessage?: {            // Information about message being replied to
    id: string;
    content: string;
  };
  attachments?: Array<{
    url: string;
    type: string;
    filename?: string;
  }>;
  metadata: Record<string, any>; // Additional message metadata
}

export interface ProcessedResponse {
  content: string;              // Response content
  contentType: 'text' | 'image' | 'document' | 'quick_reply' | 'carousel';
  attachments?: Array<{
    url: string;
    type: string;
    filename?: string;
  }>;
  quickReplies?: Array<{
    title: string;
    payload: string;
  }>;
  metadata: Record<string, any>; // Response metadata
}
```

### 2.2 Define Channel Configurations

Create `src/mastra/core/models/channel.ts`:

```typescript
/**
 * Channel definitions and utilities
 */

export type ChannelType = 'telegram' | 'whatsapp' | 'web' | 'line' | 'facebook' | 'email' | 'zalo';

export interface ChannelConfig {
  type: ChannelType;
  name: string;
  supports: {
    text: boolean;
    images: boolean;
    documents: boolean;
    audio: boolean;
    video: boolean;
    quickReplies: boolean;
    carousel: boolean;
  };
  maxMessageLength: number;
  rateLimits?: {
    messagesPerSecond: number;
    messagesPerMinute: number;
  };
}

export const CHANNEL_CONFIGS: Record<ChannelType, ChannelConfig> = {
  telegram: {
    type: 'telegram',
    name: 'Telegram',
    supports: {
      text: true,
      images: true,
      documents: true,
      audio: true,
      video: true,
      quickReplies: true,
      carousel: false
    },
    maxMessageLength: 4096
  },
  whatsapp: {
    type: 'whatsapp',
    name: 'WhatsApp',
    supports: {
      text: true,
      images: true,
      documents: true,
      audio: true,
      video: true,
      quickReplies: true,
      carousel: false
    },
    maxMessageLength: 4096
  },
  web: {
    type: 'web',
    name: 'Web Chat',
    supports: {
      text: true,
      images: true,
      documents: true,
      audio: true,
      video: true,
      quickReplies: true,
      carousel: true
    },
    maxMessageLength: 10000
  },
  line: {
    type: 'line',
    name: 'LINE',
    supports: {
      text: true,
      images: true,
      documents: false,
      audio: true,
      video: true,
      quickReplies: true,
      carousel: true
    },
    maxMessageLength: 2000
  },
  facebook: {
    type: 'facebook',
    name: 'Facebook Messenger',
    supports: {
      text: true,
      images: true,
      documents: false,
      audio: true,
      video: true,
      quickReplies: true,
      carousel: true
    },
    maxMessageLength: 2000
  },
  email: {
    type: 'email',
    name: 'Email',
    supports: {
      text: true,
      images: true,
      documents: true,
      audio: false,
      video: false,
      quickReplies: false,
      carousel: false
    },
    maxMessageLength: 100000
  },
  zalo: {
    type: 'zalo',
    name: 'Zalo',
    supports: {
      text: true,
      images: true,
      documents: true,
      audio: true,
      video: true,
      quickReplies: true,
      carousel: false
    },
    maxMessageLength: 4096
  }
};
```

## Step 3: Create Central Message Processor

### 3.1 Main Message Processor

Create `src/mastra/core/processor/message-processor.ts`:

```typescript
/**
 * Central message processor that handles messages from all channels
 * This is WHERE YOUR BUSINESS LOGIC LIVES
 */

import { NormalizedMessage, ProcessedResponse } from '../models/message';
import { callModel } from '../../../llm/adapter';
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
   * THIS IS WHERE ALL YOUR BUSINESS LOGIC GOES
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
      
      // Call the LLM
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
      await new Promise((resolve) => setTimeout(resolve, 10));
    }

    this.isProcessing = false;
  }
}

// Export singleton instance
export const messageProcessor = new CentralMessageProcessor();
```

## Step 4: Create Channel Management

### 4.1 Channel Registry

Create `src/mastra/channels/registry.ts`:

```typescript
/**
 * Channel registry for managing active channels
 * This keeps track of which channels are currently active
 */

// Simple interface for channel adapters
export interface ChannelAdapter {
  channelId: string;
  handleMessage: (rawMessage: any) => Promise<void>;
  shutdown?: () => Promise<void>;
}

export class ChannelRegistry {
  private adapters = new Map<string, ChannelAdapter>();
  
  /**
   * Register a channel adapter
   */
  register(channelId: string, adapter: ChannelAdapter): void {
    this.adapters.set(channelId, adapter);
    console.log(`✅ Registered channel: ${channelId}`);
  }
  
  /**
   * Get a channel adapter by ID
   */
  get(channelId: string): ChannelAdapter | undefined {
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
      if (adapter.shutdown) {
        shutdownPromises.push(adapter.shutdown());
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

## Step 5: Create Main Application

### 5.1 Application Entry Point

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

  // Load Telegram channel if configured
  if (process.env.TELEGRAM_BOT_TOKEN) {
    try {
      const { TelegramChannelAdapter } = await import('./channels/telegram');
      const adapter = new TelegramChannelAdapter(
        { token: process.env.TELEGRAM_BOT_TOKEN },
        processor
      );
      channelRegistry.register('telegram', adapter);
    } catch (error) {
      console.error('❌ Failed to load Telegram channel:', error);
    }
  }

  // Load WhatsApp channel if configured
  if (process.env.WHATSAPP_ACCESS_TOKEN && process.env.WHATSAPP_PHONE_NUMBER_ID) {
    try {
      const { WhatsAppChannelAdapter } = await import('./channels/whatsapp');
      const adapter = new WhatsAppChannelAdapter(
        {
          accessToken: process.env.WHATSAPP_ACCESS_TOKEN,
          phoneNumberId: process.env.WHATSAPP_PHONE_NUMBER_ID,
          webhookVerifyToken: process.env.WHATSAPP_WEBHOOK_VERIFY_TOKEN || 'default_verify_token'
        },
        processor
      );
      channelRegistry.register('whatsapp', adapter);
    } catch (error) {
      console.error('❌ Failed to load WhatsApp channel:', error);
    }
  }

  // Add more channels here as needed
  console.log(`✅ Loaded channels: ${channelRegistry.listChannels().join(', ')}`);
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

## Benefits of This Architecture

### 🎯 **Clear Channel Visibility**
```
src/channels/
├── telegram/     ← Immediately clear Telegram support
├── whatsapp/     ← Immediately clear WhatsApp support
├── web/          ← Immediately clear Web support
└── line/         ← Immediately clear LINE support
```

### 🔧 **Easy Maintenance**
- **Business logic** in `src/mastra/core/processor/` (ONE PLACE)
- **Channel adapters** in `src/channels/` (SEPARATE MODULES)
- **No redundant packages** - clean, simple structure

### 🚀 **Scalability**
- Add new channel: `mkdir src/channels/newchannel`
- Remove channel: `rm -rf src/channels/oldchannel`
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

1. **✅ Completed**: Created clean project structure
2. **✅ Completed**: Defined standardized message models
3. **✅ Completed**: Built central message processor
4. **✅ Completed**: Set up channel management
5. **Now**: Implement specific channel adapters (see Part 2)
6. **Next**: Test with actual channels
7. **Later**: Add more channels as needed
8. **Finally**: Deploy with monitoring and logging

This architecture provides a clean, maintainable, and scalable solution that makes it immediately clear which channels are supported while keeping all business logic centralized.