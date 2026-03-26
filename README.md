# >private_chat

A real-time, ephemeral chat application where rooms self-destruct after 10 minutes. Built for private, temporary conversations between two people — no accounts, no history, no trace.

![Next.js](https://img.shields.io/badge/Next.js-16-black?logo=next.js)
![React](https://img.shields.io/badge/React-19-blue?logo=react)
![Upstash](https://img.shields.io/badge/Upstash-Redis-green?logo=redis)
![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?logo=typescript)
![Tailwind CSS](https://img.shields.io/badge/Tailwind-4-38bdf8?logo=tailwindcss)

## How It Works

1. A user creates a room and gets a shareable link
2. They send the link to someone — only 2 people can join
3. Messages appear in real-time via Server-Sent Events
4. After 10 minutes, the room and all messages are permanently deleted
5. Either user can hit "DESTROY NOW" to delete everything instantly

There are no accounts. Each user gets a random anonymous identity (e.g. `anon-wolf-a1b`) stored in localStorage.

## Architecture

This project runs entirely on **serverless infrastructure** — no persistent server, no WebSockets, no traditional database.

```
┌──────────┐       ┌────────────────────┐       ┌──────────────┐
│  Browser  │──────▶│  Vercel Serverless  │──────▶│  Upstash Redis│
│  (React)  │◀──SSE─│  (Next.js + Elysia) │◀──────│  (Data + Pub/Sub)│
└──────────┘       └────────────────────┘       └──────────────┘
```

- **Upstash Redis** serves as the database, auth store, and message broker — all in one
- **Upstash Realtime** provides real-time delivery via SSE (Server-Sent Events), which works on serverless platforms unlike WebSockets
- **Elysia** handles API routing with end-to-end type safety via Eden
- **Next.js 16 Proxy** (`proxy.ts`) controls room access, sets auth cookies, and blocks bots from consuming room slots

### Redis Data Model

```
meta:{roomId}       → Hash  { connected: [token1, token2], createdAt: timestamp }
messages:{roomId}   → List  [ {id, sender, text, timestamp, roomId}, ... ]

All keys expire automatically after 600 seconds (10 minutes).
```

### Auth Flow

Room access is cookie-based with no login required:

1. User visits `/room/{id}` → `proxy.ts` runs
2. Proxy generates a token, stores it in Redis `connected` array, and sets it as an `httpOnly` cookie
3. Every API call passes through `auth.ts` middleware which validates the cookie token against Redis
4. Max 2 tokens per room — third visitor gets redirected with `?error=room-full`

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Framework | [Next.js 16](https://nextjs.org) (App Router) |
| UI | [React 19](https://react.dev), [Tailwind CSS 4](https://tailwindcss.com), [shadcn/ui](https://ui.shadcn.com) |
| API | [Elysia](https://elysiajs.com) + [Eden](https://elysiajs.com/eden/overview) (type-safe RPC) |
| Database | [Upstash Redis](https://upstash.com/redis) (serverless, REST-based) |
| Realtime | [Upstash Realtime](https://upstash.com/docs/redis/sdks/ts/modules/realtime) (SSE) |
| Validation | [Zod](https://zod.dev) |
| Data Fetching | [TanStack Query](https://tanstack.com/query) |
| Icons | [Phosphor Icons](https://phosphoricons.com) |
| Deployment | [Vercel](https://vercel.com) |

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org) 18+ or [Bun](https://bun.sh)
- An [Upstash](https://upstash.com) account (free tier works)

### Setup

1. **Clone the repository**

   ```bash
   git clone https://github.com/YOUR_USERNAME/real-time-chat.git
   cd real-time-chat
   ```

2. **Install dependencies**

   ```bash
   npm install
   # or
   bun install
   ```

3. **Create a `.env` file** in the project root with your Upstash credentials:

   ```env
   UPSTASH_REDIS_REST_URL="your-upstash-redis-rest-url"
   UPSTASH_REDIS_REST_TOKEN="your-upstash-redis-rest-token"
   ```

   You can get these from the [Upstash Console](https://console.upstash.com) after creating a Redis database.

4. **Run the dev server**

   ```bash
   npm run dev
   # or
   bun dev
   ```

5. Open [http://localhost:3000](http://localhost:3000)

## Project Structure

```
src/
├── app/
│   ├── api/
│   │   ├── [[...slugs]]/
│   │   │   ├── route.ts        # Elysia API (rooms + messages)
│   │   │   └── auth.ts         # Cookie-based auth middleware
│   │   └── realtime/
│   │       └── route.ts        # SSE endpoint for real-time events
│   ├── room/[roomId]/
│   │   └── page.tsx            # Chat room UI
│   ├── page.tsx                # Lobby / room creation
│   └── layout.tsx              # Root layout with providers
├── components/
│   ├── providers.tsx           # React Query + Realtime providers
│   └── ui/                     # shadcn/ui components
├── hooks/
│   └── use-username.ts         # Random anonymous identity
├── lib/
│   ├── redis.ts                # Upstash Redis client
│   ├── realtime.ts             # Realtime channel + event schemas
│   ├── realtime-client.ts      # Client-side useRealtime hook
│   ├── client.ts               # Elysia Eden API client
│   └── utils.ts                # Tailwind class merge utility
└── proxy.ts                    # Room access control (Next.js 16 proxy)
```

## API Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/room/create` | No | Create a new room |
| `GET` | `/api/room/ttl` | Yes | Get remaining time for a room |
| `DELETE` | `/api/room` | Yes | Destroy a room and all its messages |
| `POST` | `/api/message` | Yes | Send a message (max 1000 chars) |
| `GET` | `/api/message` | Yes | Fetch all messages in a room |
| `GET` | `/api/realtime` | No | SSE stream for real-time events |

## Deployment

Deploy to Vercel with one click:

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/YOUR_USERNAME/real-time-chat&env=UPSTASH_REDIS_REST_URL,UPSTASH_REDIS_REST_TOKEN)

Or deploy manually:

```bash
npm install -g vercel
vercel
```

Set `UPSTASH_REDIS_REST_URL` and `UPSTASH_REDIS_REST_TOKEN` in your Vercel project's environment variables.

## License

MIT
