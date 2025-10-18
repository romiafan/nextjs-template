# Next.js Template with Convex & Clerk

A modern Next.js template featuring authentication via Clerk and real-time backend with Convex.

## Tech Stack

- **Framework**: [Next.js 15](https://nextjs.org) with App Router
- **Language**: TypeScript
- **Styling**: Tailwind CSS 4
- **Authentication**: [Clerk](https://clerk.com)
- **Backend**: [Convex](https://convex.dev) - Real-time backend with type-safe APIs
- **UI Components**: Custom components with class-variance-authority
- **Package Manager**: pnpm

## Features

- 🔐 Authentication with Clerk (email, social logins)
- ⚡ Real-time database with Convex
- 🎨 Tailwind CSS with custom component system
- 📱 Responsive design
- 🔒 Type-safe API calls
- 🚀 Turbopack for fast development

## Prerequisites

- Node.js 20+ installed
- pnpm installed (`npm install -g pnpm`)
- Clerk account ([clerk.com](https://clerk.com))
- Convex account ([convex.dev](https://convex.dev))

## Getting Started

### 1. Clone and Install

```bash
# Clone the repository
git clone <your-repo-url>
cd nextjs-template

# Install dependencies
pnpm install
```

### 2. Environment Variables

Create a `.env.local` file in the root directory:

```env
# Clerk Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...
NEXT_PUBLIC_CLERK_SIGN_IN_URL=/sign-in
NEXT_PUBLIC_CLERK_SIGN_UP_URL=/sign-up

# Convex
NEXT_PUBLIC_CONVEX_URL=https://your-convex-url.convex.cloud
CONVEX_DEPLOYMENT=your-deployment-name
```

**Get your keys:**
- **Clerk**: Sign up at [clerk.com](https://clerk.com) and create an application
- **Convex**: Sign up at [convex.dev](https://convex.dev) and create a project

### 3. Set Up Convex

```bash
# Initialize Convex (if not already done)
npx convex dev
```

This will:
- Create a Convex project (if needed)
- Generate type-safe API code
- Start the Convex development server
- Watch for changes to your Convex functions

### 4. Run the Development Server

```bash
pnpm dev
```

Open [http://localhost:3000](http://localhost:3000) to see your app.

## Project Structure

```
├── convex/                 # Convex backend
│   ├── auth.config.ts     # Authentication configuration
│   ├── messages.ts        # Message queries/mutations
│   ├── schema.ts          # Database schema
│   └── _generated/        # Auto-generated types
├── src/
│   ├── app/               # Next.js app router pages
│   │   ├── layout.tsx     # Root layout with providers
│   │   ├── page.tsx       # Home page
│   │   └── pricing/       # Pricing page example
│   ├── components/        # React components
│   │   └── ConvexClientProvider.tsx
│   ├── lib/               # Utilities
│   │   └── utils.ts
│   └── middleware.ts      # Clerk middleware
├── public/                # Static assets
└── package.json
```

## Available Scripts

```bash
pnpm dev          # Start development server with Turbopack
pnpm build        # Build for production
pnpm start        # Start production server
pnpm lint         # Run ESLint
```

## Convex Backend

### Schema

The database schema is defined in `convex/schema.ts`. Example:

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  messages: defineTable({
    author: v.string(),
    text: v.string(),
  }),
});
```

### Queries and Mutations

Define backend functions in `convex/`:

```typescript
// convex/messages.ts
import { query } from "./_generated/server";

export const getForCurrentUser = query({
  args: {},
  handler: async (ctx) => {
    const identity = await ctx.auth.getUserIdentity();
    // ... your logic
  },
});
```

Use them in your React components:

```typescript
import { useQuery } from "convex/react";
import { api } from "../../convex/_generated/api";

const messages = useQuery(api.messages.getForCurrentUser);
```

## Authentication

This template uses Clerk for authentication. Key files:

- `src/middleware.ts` - Protects routes
- `src/app/layout.tsx` - ClerkProvider setup
- `src/components/ConvexClientProvider.tsx` - Integrates Clerk with Convex

### Protected Routes

By default, all routes are protected. To make a route public, update `src/middleware.ts`:

```typescript
export default clerkMiddleware((auth, req) => {
  // Public routes configuration
});
```

## Styling

This project uses:
- **Tailwind CSS 4** for utility-first styling
- **class-variance-authority** for component variants
- **tailwind-merge** for merging Tailwind classes
- **lucide-react** for icons

Components are in `src/components/` with utilities in `src/lib/utils.ts`.

## Deployment

### Deploy to Vercel

1. Push your code to GitHub
2. Import your repository on [Vercel](https://vercel.com)
3. Add environment variables in Vercel dashboard
4. Deploy!

### Deploy Convex

```bash
npx convex deploy
```

Update your production environment variables with the production Convex URL.

## Learn More

- [Next.js Documentation](https://nextjs.org/docs)
- [Convex Documentation](https://docs.convex.dev)
- [Clerk Documentation](https://clerk.com/docs)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)

## License

MIT
