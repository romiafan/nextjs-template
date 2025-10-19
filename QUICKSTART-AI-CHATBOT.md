# Quick Start: AI Chatbot Project

This guide walks you through building an AI chatbot (GPT/Gemini wrapper) using this template.

## 🎯 Project Overview

You'll build:

- Real-time chat interface
- Conversation history stored in Convex
- OpenAI/Google AI integration
- User authentication with Clerk
- Streaming responses

**Estimated time**: 30-60 minutes

---

## Step 1: Set Up New Project

### Fork/Clone Template

```bash
# Option A: Use GitHub's "Use this template" button (recommended)
# Go to: https://github.com/romiafan/nextjs-template
# Click "Use this template" → "Create a new repository"
# Name: "ai-chatbot" or your project name

# Option B: Clone and create new repo
git clone https://github.com/romiafan/nextjs-template.git ai-chatbot
cd ai-chatbot
rm -rf .git
git init
git add .
git commit -m "feat: initialize AI chatbot from template"
```

### Install Dependencies

```bash
pnpm install
```

---

## Step 2: Set Up Services

### 2.1 Clerk (Authentication)

1. Go to [clerk.com](https://clerk.com)
2. Create new application: "AI Chatbot"
3. Copy API keys

### 2.2 Convex (Backend)

1. Go to [convex.dev](https://convex.dev)
2. Create new project: "ai-chatbot"
3. Run: `npx convex dev`
4. Copy the URL it generates

### 2.3 OpenAI or Google AI

Choose one:

**OpenAI**:

1. Go to [platform.openai.com](https://platform.openai.com)
2. Create API key
3. Copy key

**Google AI (Gemini)**:

1. Go to [ai.google.dev](https://ai.google.dev)
2. Get API key
3. Copy key

### 2.4 Environment Variables

Create `.env.local`:

```bash
# Clerk
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

# Convex
NEXT_PUBLIC_CONVEX_URL=https://...convex.cloud

# AI Provider (choose one)
OPENAI_API_KEY=sk-...
# OR
GOOGLE_AI_API_KEY=...

# Site config
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

---

## Step 3: Update Branding

### 3.1 Site Metadata

Edit `src/lib/metadata.ts`:

```typescript
export const siteConfig = {
  name: "AI Chat Assistant", // ← Change this
  description: "Your intelligent AI-powered chat assistant", // ← Change this
  url: process.env.NEXT_PUBLIC_SITE_URL || "http://localhost:3000",
  ogImage: "/og-image.png",
  links: {
    twitter: "https://twitter.com/yourhandle",
    github: "https://github.com/yourusername/ai-chatbot",
  },
};
```

### 3.2 Navbar

Edit `src/components/Navbar.tsx`:

```typescript
// Line ~27
<span className="text-xl font-bold text-foreground">
  AI Chat Assistant {/* ← Change this */}
</span>
```

### 3.3 Home Page

Edit `src/app/page.tsx` - update hero text:

```typescript
<h1 className="text-4xl md:text-6xl font-bold text-foreground">
  Your AI Assistant
  <span className="block text-primary">Available 24/7</span>
</h1>
```

---

## Step 4: Database Schema

Edit `convex/schema.ts`:

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  // Conversations
  conversations: defineTable({
    userId: v.string(),
    title: v.string(),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_user", ["userId"])
    .index("by_updated", ["updatedAt"]),

  // Messages
  messages: defineTable({
    conversationId: v.id("conversations"),
    userId: v.string(),
    role: v.union(
      v.literal("user"),
      v.literal("assistant"),
      v.literal("system")
    ),
    content: v.string(),
    createdAt: v.number(),
  })
    .index("by_conversation", ["conversationId"])
    .index("by_user", ["userId"]),
});
```

**Save and watch Convex auto-generate types!**

---

## Step 5: Convex Backend Functions

### 5.1 Conversation Functions

Create `convex/conversations.ts`:

```typescript
import { query, mutation } from "./_generated/server";
import { v } from "convex/values";

// List user's conversations
export const list = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    return await ctx.db
      .query("conversations")
      .withIndex("by_user", (q) => q.eq("userId", identity.subject))
      .order("desc")
      .collect();
  },
});

// Create new conversation
export const create = mutation({
  args: {
    title: v.string(),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    const now = Date.now();
    return await ctx.db.insert("conversations", {
      userId: identity.subject,
      title: args.title,
      createdAt: now,
      updatedAt: now,
    });
  },
});

// Delete conversation
export const remove = mutation({
  args: {
    id: v.id("conversations"),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    // Verify ownership
    const conversation = await ctx.db.get(args.id);
    if (!conversation || conversation.userId !== identity.subject) {
      throw new Error("Not authorized");
    }

    // Delete all messages first
    const messages = await ctx.db
      .query("messages")
      .withIndex("by_conversation", (q) => q.eq("conversationId", args.id))
      .collect();

    for (const message of messages) {
      await ctx.db.delete(message._id);
    }

    // Delete conversation
    await ctx.db.delete(args.id);
  },
});
```

### 5.2 Message Functions

Create `convex/messages.ts`:

```typescript
import { query, mutation, action } from "./_generated/server";
import { v } from "convex/values";
import { api } from "./_generated/api";

// Get messages for a conversation
export const list = query({
  args: {
    conversationId: v.id("conversations"),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    return await ctx.db
      .query("messages")
      .withIndex("by_conversation", (q) =>
        q.eq("conversationId", args.conversationId)
      )
      .order("asc")
      .collect();
  },
});

// Add user message
export const add = mutation({
  args: {
    conversationId: v.id("conversations"),
    content: v.string(),
  },
  handler: async (ctx, args) => {
    const identity = await ctx.auth.getUserIdentity();
    if (!identity) throw new Error("Not authenticated");

    // Verify conversation ownership
    const conversation = await ctx.db.get(args.conversationId);
    if (!conversation || conversation.userId !== identity.subject) {
      throw new Error("Not authorized");
    }

    // Add message
    const messageId = await ctx.db.insert("messages", {
      conversationId: args.conversationId,
      userId: identity.subject,
      role: "user",
      content: args.content,
      createdAt: Date.now(),
    });

    // Update conversation timestamp
    await ctx.db.patch(args.conversationId, {
      updatedAt: Date.now(),
    });

    return messageId;
  },
});

// Add AI response (called from action)
export const addAIResponse = mutation({
  args: {
    conversationId: v.id("conversations"),
    content: v.string(),
  },
  handler: async (ctx, args) => {
    return await ctx.db.insert("messages", {
      conversationId: args.conversationId,
      userId: "ai",
      role: "assistant",
      content: args.content,
      createdAt: Date.now(),
    });
  },
});

// Send message and get AI response
export const sendMessage = action({
  args: {
    conversationId: v.id("conversations"),
    content: v.string(),
  },
  handler: async (ctx, args) => {
    // Add user message
    await ctx.runMutation(api.messages.add, {
      conversationId: args.conversationId,
      content: args.content,
    });

    // Get conversation history
    const messages = await ctx.runQuery(api.messages.list, {
      conversationId: args.conversationId,
    });

    // Call OpenAI API
    const response = await fetch("https://api.openai.com/v1/chat/completions", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${process.env.OPENAI_API_KEY}`,
      },
      body: JSON.stringify({
        model: "gpt-4o-mini",
        messages: messages.map((m) => ({
          role: m.role,
          content: m.content,
        })),
        temperature: 0.7,
        max_tokens: 1000,
      }),
    });

    if (!response.ok) {
      throw new Error(`OpenAI API error: ${response.statusText}`);
    }

    const data = await response.json();
    const aiMessage = data.choices[0].message.content;

    // Save AI response
    await ctx.runMutation(api.messages.addAIResponse, {
      conversationId: args.conversationId,
      content: aiMessage,
    });

    return aiMessage;
  },
});
```

---

## Step 6: Chat UI Components

### 6.1 Chat Page

Create `src/app/chat/page.tsx`:

```tsx
"use client";

import { MainLayout } from "@/components/layouts/MainLayout";
import { useQuery, useMutation, useAction } from "convex/react";
import { api } from "../../../convex/_generated/api";
import { useState } from "react";
import { Send, Plus, Trash2 } from "lucide-react";
import { Id } from "../../../convex/_generated/dataModel";

export default function ChatPage() {
  const conversations = useQuery(api.conversations.list);
  const createConversation = useMutation(api.conversations.create);
  const deleteConversation = useMutation(api.conversations.remove);
  const sendMessage = useAction(api.messages.sendMessage);

  const [selectedConversationId, setSelectedConversationId] =
    useState<Id<"conversations"> | null>(null);
  const [input, setInput] = useState("");
  const [isSending, setIsSending] = useState(false);

  const messages = useQuery(
    api.messages.list,
    selectedConversationId ? { conversationId: selectedConversationId } : "skip"
  );

  const handleNewChat = async () => {
    const id = await createConversation({ title: "New Chat" });
    setSelectedConversationId(id);
  };

  const handleSend = async () => {
    if (!input.trim() || !selectedConversationId || isSending) return;

    const message = input;
    setInput("");
    setIsSending(true);

    try {
      await sendMessage({
        conversationId: selectedConversationId,
        content: message,
      });
    } catch (error) {
      console.error("Error sending message:", error);
    } finally {
      setIsSending(false);
    }
  };

  return (
    <MainLayout>
      <div className="container mx-auto h-[calc(100vh-4rem)] flex gap-4 p-4">
        {/* Sidebar */}
        <div className="w-64 border-r flex flex-col gap-2">
          <button
            onClick={handleNewChat}
            className="flex items-center gap-2 px-4 py-2 bg-primary text-primary-foreground rounded-md hover:bg-primary/90"
          >
            <Plus className="w-4 h-4" />
            New Chat
          </button>

          <div className="flex-1 overflow-y-auto space-y-1">
            {conversations?.map((conv) => (
              <div
                key={conv._id}
                className={`flex items-center gap-2 px-3 py-2 rounded-md cursor-pointer hover:bg-muted ${
                  selectedConversationId === conv._id ? "bg-muted" : ""
                }`}
                onClick={() => setSelectedConversationId(conv._id)}
              >
                <span className="flex-1 truncate text-sm">{conv.title}</span>
                <button
                  onClick={(e) => {
                    e.stopPropagation();
                    deleteConversation({ id: conv._id });
                    if (selectedConversationId === conv._id) {
                      setSelectedConversationId(null);
                    }
                  }}
                  className="text-muted-foreground hover:text-destructive"
                >
                  <Trash2 className="w-4 h-4" />
                </button>
              </div>
            ))}
          </div>
        </div>

        {/* Chat Area */}
        <div className="flex-1 flex flex-col">
          {selectedConversationId ? (
            <>
              {/* Messages */}
              <div className="flex-1 overflow-y-auto space-y-4 p-4">
                {messages?.map((msg) => (
                  <div
                    key={msg._id}
                    className={`flex ${
                      msg.role === "user" ? "justify-end" : "justify-start"
                    }`}
                  >
                    <div
                      className={`max-w-[70%] rounded-lg px-4 py-2 ${
                        msg.role === "user"
                          ? "bg-primary text-primary-foreground"
                          : "bg-muted"
                      }`}
                    >
                      <p className="text-sm whitespace-pre-wrap">
                        {msg.content}
                      </p>
                    </div>
                  </div>
                ))}
                {isSending && (
                  <div className="flex justify-start">
                    <div className="bg-muted rounded-lg px-4 py-2">
                      <p className="text-sm">Thinking...</p>
                    </div>
                  </div>
                )}
              </div>

              {/* Input */}
              <div className="border-t p-4">
                <div className="flex gap-2">
                  <input
                    type="text"
                    value={input}
                    onChange={(e) => setInput(e.target.value)}
                    onKeyDown={(e) => e.key === "Enter" && handleSend()}
                    placeholder="Type a message..."
                    className="flex-1 px-4 py-2 border rounded-md focus:outline-none focus:ring-2 focus:ring-primary"
                    disabled={isSending}
                  />
                  <button
                    onClick={handleSend}
                    disabled={isSending || !input.trim()}
                    className="px-4 py-2 bg-primary text-primary-foreground rounded-md hover:bg-primary/90 disabled:opacity-50"
                  >
                    <Send className="w-5 h-5" />
                  </button>
                </div>
              </div>
            </>
          ) : (
            <div className="flex-1 flex items-center justify-center text-muted-foreground">
              <p>Select a conversation or create a new one</p>
            </div>
          )}
        </div>
      </div>
    </MainLayout>
  );
}
```

### 6.2 Add Chat Link to Navbar

Edit `src/components/Navbar.tsx`, add after Pricing link:

```typescript
<Link
  href="/chat"
  className="text-sm font-medium text-muted-foreground hover:text-foreground transition-colors"
>
  Chat
</Link>
```

---

## Step 7: Update Home Page

Edit `src/app/page.tsx` - update CTA button:

```typescript
<Link
  href="/chat" // ← Change from /pricing
  className="inline-flex items-center justify-center gap-2 px-6 py-3 bg-primary text-primary-foreground rounded-md hover:bg-primary/90 transition-colors font-medium"
>
  Start Chatting {/* ← Change text */}
  <ArrowRight className="w-4 h-4" />
</Link>
```

---

## Step 8: Test Locally

```bash
# Terminal 1: Start Convex
npx convex dev

# Terminal 2: Start Next.js
pnpm dev
```

Visit `http://localhost:3000`:

1. Sign up/Sign in with Clerk
2. Click "Chat" in navbar
3. Create new chat
4. Send a message
5. Watch AI respond!

---

## Step 9: Deploy

### Deploy Convex

```bash
npx convex deploy --prod
```

Copy the production URL.

### Deploy to Vercel

1. Push to GitHub
2. Go to [vercel.com](https://vercel.com)
3. Import repository
4. Add environment variables:
   ```
   NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
   CLERK_SECRET_KEY
   NEXT_PUBLIC_CONVEX_URL  (production URL)
   OPENAI_API_KEY
   NEXT_PUBLIC_SITE_URL
   ```
5. Deploy!

### Update Clerk

1. Go to Clerk dashboard
2. Add your Vercel domain to allowed domains
3. Update redirect URLs

---

## 🎨 Customization Ideas

### Add Features

- [ ] Conversation titles auto-generated from first message
- [ ] Message streaming (use OpenAI streaming API)
- [ ] Code syntax highlighting
- [ ] File uploads
- [ ] Voice input
- [ ] Model selection (GPT-4, GPT-3.5, etc.)
- [ ] System prompts/personalities

### UI Improvements

- [ ] Markdown rendering for AI responses
- [ ] Copy message button
- [ ] Regenerate response
- [ ] Edit messages
- [ ] Dark/light mode toggle
- [ ] Mobile-responsive sidebar

### Advanced

- [ ] Rate limiting (Convex actions)
- [ ] Token counting and usage tracking
- [ ] Conversation sharing
- [ ] Export conversations
- [ ] Multi-user conversations

---

## 📚 Resources

- [OpenAI API Docs](https://platform.openai.com/docs)
- [Convex Actions Guide](https://docs.convex.dev/functions/actions)
- [Clerk Authentication](https://clerk.com/docs)
- [Next.js App Router](https://nextjs.org/docs/app)

---

## 🐛 Troubleshooting

**"Not authenticated" error**:

- Make sure you're signed in with Clerk
- Check Convex dev is running

**OpenAI API errors**:

- Verify `OPENAI_API_KEY` in `.env.local`
- Check API key has credits
- Try simpler model: `gpt-3.5-turbo`

**Messages not showing**:

- Check Convex dashboard for data
- Verify conversation ID is correct
- Check browser console for errors

---

## ✅ Done!

You now have a working AI chatbot! 🎉

Next steps:

- Customize the UI
- Add more features
- Deploy to production
- Share with users
