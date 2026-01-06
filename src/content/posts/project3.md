---
title: Real-Time Chat Application
published: 2025-07-22
tags: [Markdown, Blogging, Demo]
category: Projects
draft: true
---

# Building a Production Real-Time Chat Application: Mastering Supabase, WebSockets, and Optimistic Updates

## Introduction

Real-time applications are notoriously difficult to build. Managing WebSocket connections, handling concurrent updates, implementing presence detection, and ensuring data consistency—it's a maze of complexity that can easily consume months of development time.

Then I discovered Supabase Realtime.

In this post, I'll walk you through building **SuperChat**: a full-featured, production-ready real-time chat application with public/private rooms, user invitations, optimistic updates, infinite scrolling, and presence tracking. But more importantly, I'll share the lessons learned, the problems solved, and why certain architectural decisions matter.

This isn't just another chat tutorial—it's a deep dive into modern real-time application development.

## Project Overview

**What I Built:**

- Real-time messaging with instant delivery
- Public and private chat rooms
- User invitation system
- GitHub OAuth authentication
- Optimistic UI updates
- Infinite scroll with message history
- Real-time presence tracking
- Row-level security for data protection

**Why This Project Matters:**

Real-time features are everywhere: chat apps, collaborative tools, live dashboards, multiplayer games. Understanding how to build them properly is a critical skill that sets you apart as a developer.

## The Next.js Limitation: Why Supabase?

### The WebSocket Problem

Next.js is fantastic for server-side rendering and API routes, but it has one major limitation: **it doesn't support WebSockets natively**.

Why? Next.js is designed to run on serverless platforms like Vercel. Serverless functions are stateless—they spin up, handle a request, and die. WebSockets require persistent, long-lived connections. These two paradigms are fundamentally incompatible.

**Options for Real-Time in Next.js:**

1. **Run a separate WebSocket server** (adds complexity)
2. **Use polling** (inefficient, delayed updates)
3. **Use a managed service** (Supabase, Pusher, Ably)

I chose Supabase because it provides:

- Built-in PostgreSQL with real-time subscriptions
- Row-level security
- Authentication out of the box
- Free tier that's actually usable

### Why Not Build Your Own?

I've built WebSocket servers from scratch. Here's what you're signing up for:

- Managing connection states
- Handling reconnections
- Broadcasting to specific channels
- Scaling horizontally
- Load balancing
- Security and authentication
- Message queuing
- Presence detection

That's easily 200+ hours of work before you write a single line of business logic.

Supabase handles all of this. You focus on your app.

## Authentication: The Foundation

### GitHub OAuth Integration

I started with authentication because everything depends on knowing who the user is. Supabase's authentication is ridiculously easy to set up.

**Step 1: Configure GitHub OAuth**

In GitHub Settings → Developer Settings:

```
Application name: SuperChat
Homepage URL: http://localhost:3000
Callback URL: https://your-project.supabase.co/auth/v1/callback
```

**Step 2: Add to Supabase**

Paste Client ID and Secret into Supabase Dashboard → Authentication → Providers → GitHub.

**Step 3: Use Supabase Auth UI Block**

Instead of building login forms from scratch, I used Supabase's Shadcn/ui blocks:

```bash
npx shadcn@latest add https://shadcn.supabase.com/auth/login
```

This gave me a complete login page with:

- GitHub OAuth button
- Error handling
- Loading states
- Redirects

**Key Learning:**

Don't reinvent authentication. It's a solved problem. Use battle-tested solutions.

## Database Architecture: Planning for Scale

### The Schema Design

I designed the database with four core tables:

```sql
-- Users (automatically synced from auth)
CREATE TABLE user_profile (
  id UUID PRIMARY KEY REFERENCES auth.users,
  name VARCHAR NOT NULL,
  image_url VARCHAR
);

-- Chat rooms
CREATE TABLE chat_room (
  id UUID PRIMARY KEY,
  name VARCHAR NOT NULL,
  is_public BOOLEAN NOT NULL
);

-- Room membership (join table)
CREATE TABLE chat_room_member (
  member_id UUID REFERENCES user_profile,
  chat_room_id UUID REFERENCES chat_room,
  PRIMARY KEY (member_id, chat_room_id)
);

-- Messages
CREATE TABLE message (
  id UUID PRIMARY KEY,
  text TEXT NOT NULL,
  chat_room_id UUID REFERENCES chat_room,
  author_id UUID REFERENCES user_profile,
  created_at TIMESTAMP DEFAULT NOW()
);
```

**Why This Structure?**

1. **Composite Primary Key**: `chat_room_member` uses both IDs as the primary key. This ensures a user can only be in a room once—no duplicates.
    
2. **Separate User Profile**: Auth data lives in `auth.users` (Supabase internal). User profiles live in `user_profile` (our schema). This separation keeps authentication concerns isolated.
    
3. **UUID Primary Keys**: Better for distributed systems than auto-incrementing integers.
    

### Database Triggers: The Secret Sauce

Two critical triggers power the app:

**Trigger 1: Auto-Create User Profiles**

When someone signs in with GitHub, we need to create their user profile automatically:

```sql
CREATE FUNCTION handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.user_profile (id, name, image_url)
  VALUES (
    NEW.id,
    NEW.raw_user_meta_data->>'preferred_username',
    NEW.raw_user_meta_data->>'avatar_url'
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW
  EXECUTE FUNCTION handle_new_user();
```

**Key Insight:**

The `SECURITY DEFINER` clause is critical. By default, triggers run with the permissions of the user who caused them. But new users don't have permission to insert into `user_profile` yet!

`SECURITY DEFINER` makes the trigger run with the permissions of the function's owner (you), solving this chicken-and-egg problem.

**Trigger 2: Broadcast New Messages**

When a message is inserted, broadcast it to real-time subscribers:

```sql
CREATE FUNCTION broadcast_new_message()
RETURNS TRIGGER AS $$
BEGIN
  PERFORM realtime.send(
    jsonb_build_object(
      'id', NEW.id,
      'text', NEW.text,
      'created_at', NEW.created_at,
      'author_id', NEW.author_id,
      'author', jsonb_build_object(
        'name', (SELECT name FROM user_profile WHERE id = NEW.author_id),
        'image_url', (SELECT image_url FROM user_profile WHERE id = NEW.author_id)
      )
    ),
    'insert',
    'room:' || NEW.chat_room_id || ':messages',
    true  -- private channel
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

This trigger does the heavy lifting: when you insert a message, it automatically broadcasts to anyone subscribed to that room's channel.

## Row-Level Security: Permission Control

### The Challenge

How do you ensure users can only:

- Read messages from rooms they're in
- Join public rooms (but not private ones)
- Invite users to private rooms

Answer: Row-Level Security (RLS).

### Example RLS Policies

**Users can read messages from their rooms:**

```sql
CREATE POLICY "Users can read room messages"
ON message FOR SELECT
USING (
  EXISTS (
    SELECT 1 FROM chat_room_member
    WHERE member_id = auth.uid()
    AND chat_room_id = message.chat_room_id
  )
);
```

**Users can join public rooms:**

```sql
CREATE POLICY "Users can join public rooms"
ON chat_room_member FOR INSERT
WITH CHECK (
  EXISTS (
    SELECT 1 FROM chat_room
    WHERE id = chat_room_id
    AND is_public = true
  )
);
```

**Key Learning:**

RLS moves security to the database layer. Even if your application has bugs, the database enforces permissions. This is defense in depth.

## Real-Time Implementation: The Core Feature

### Setting Up Realtime Subscriptions

```typescript
const useRealtimeChat = (roomId: string, userId: string) => {
  const [messages, setMessages] = useState<Message[]>([]);
  const [connectedUsers, setConnectedUsers] = useState(1);

  useEffect(() => {
    const supabase = createClient();
    
    // Create channel with presence tracking
    const channel = supabase.channel(`room:${roomId}:messages`, {
      config: { 
        private: true,
        presence: { key: userId }
      }
    });

    // Listen for presence changes
    channel.on('presence', { event: 'sync' }, () => {
      const state = channel.presenceState();
      setConnectedUsers(Object.keys(state).length);
    });

    // Listen for new messages
    channel.on('broadcast', { event: 'insert' }, (payload) => {
      setMessages(prev => [...prev, payload.record]);
    });

    // Subscribe and track presence
    channel.subscribe((status) => {
      if (status === 'SUBSCRIBED') {
        channel.track({ user_id: userId });
      }
    });

    return () => {
      channel.untrack();
      channel.unsubscribe();
    };
  }, [roomId, userId]);

  return { messages, connectedUsers };
};
```

**What's Happening:**

1. **Private Channels**: `private: true` requires RLS policies to access
2. **Presence Key**: `userId` prevents duplicate counts if user has multiple tabs open
3. **Broadcast Events**: Listen for `insert` events from database trigger
4. **Cleanup**: Untrack and unsubscribe when component unmounts

## Optimistic Updates: Instant Feedback

### The Problem

Without optimistic updates, here's what happens:

1. User types message, clicks send
2. Message sends to server (300ms)
3. Server saves to database (50ms)
4. Trigger broadcasts message (50ms)
5. Client receives broadcast (100ms)
6. Total: ~500ms delay

Users perceive delays over 100ms. This feels sluggish.

### The Solution: Optimistic UI

```typescript
const sendMessage = async (text: string) => {
  // 1. Generate optimistic message
  const optimisticMessage = {
    id: crypto.randomUUID(),
    text,
    author_id: userId,
    author: { name, image_url },
    created_at: new Date().toISOString(),
    status: 'pending' as const
  };

  // 2. Add to UI immediately
  onSend(optimisticMessage);

  // 3. Send to server
  const result = await sendMessageAction({ id: optimisticMessage.id, text, roomId });

  // 4. Update status
  if (result.error) {
    onErrorSend(optimisticMessage.id);
  } else {
    onSuccessSend(result.message);
  }
};
```

**State Management:**

```typescript
const [sentMessages, setSentMessages] = useState<SentMessage[]>([]);

// Combine server messages + realtime + optimistic
const visibleMessages = [
  ...serverMessages,
  ...realtimeMessages.reverse(),
  ...sentMessages.filter(msg => 
    msg.status !== 'success' || 
    !realtimeMessages.find(m => m.id === msg.id)
  )
];
```

**Visual Feedback:**

```typescript
// Pending messages are semi-transparent
<div className={cn(
  "message",
  status === 'pending' && "opacity-70",
  status === 'error' && "bg-destructive/10 text-destructive"
)}>
```

**Result:** Messages appear instantly. Users feel the app is fast even when network is slow.

## Infinite Scroll: Loading Message History

### The Naive Approach

Load all messages at once:

```typescript
const messages = await supabase
  .from('message')
  .select('*')
  .eq('chat_room_id', roomId);
```

**Problem:** If a room has 10,000 messages, you're loading 10,000 messages on page load. Slow, wasteful, terrible UX.

### The Better Approach: Pagination

```typescript
const useInfiniteScrollChat = (roomId: string, startingMessages: Message[]) => {
  const [messages, setMessages] = useState(startingMessages);
  const [status, setStatus] = useState<'idle' | 'loading' | 'done'>('idle');
  const LIMIT = 25;

  const loadMore = async () => {
    if (status !== 'idle') return;

    setStatus('loading');

    const { data } = await supabase
      .from('message')
      .select('*')
      .eq('chat_room_id', roomId)
      .lt('created_at', messages[0].created_at)  // Older than oldest message
      .order('created_at', { ascending: false })
      .limit(LIMIT);

    if (data.length < LIMIT) setStatus('done');
    else setStatus('idle');

    setMessages(prev => [...data.reverse(), ...prev]);
  };

  const triggerRef = useCallback((node: HTMLDivElement | null) => {
    if (!node || status !== 'idle') return;

    const observer = new IntersectionObserver(
      (entries) => {
        if (entries[0].isIntersecting) {
          loadMore();
          observer.disconnect();
        }
      },
      { rootMargin: '50px' }
    );

    observer.observe(node);
  }, [status]);

  return { messages, triggerRef, status };
};
```

**Key Concepts:**

1. **Cursor-Based Pagination**: Use `created_at` as cursor, load messages older than oldest visible message
2. **Intersection Observer**: Trigger load when user scrolls near top
3. **Root Margin**: Start loading 50px before user reaches top (preload)
4. **Status Tracking**: Prevent duplicate loads

### The Scroll Bar Trick

Normally, when you add content to the top of a scrollable div, the scroll position stays at the top. But we want it to stay at the bottom (showing newest messages).

**CSS Solution:**

```css
.messages-container {
  display: flex;
  flex-direction: column-reverse;
}
```

By reversing the flex direction, the scroll "top" is actually the bottom. When you add messages, they go to the "top" (actually bottom), and the scroll stays put.

Then wrap messages in a div so they don't render in reverse:

```tsx
<div className="flex flex-col-reverse">
  <div> {/* Messages render normally */}
    {messages.map(msg => <ChatMessage />)}
  </div>
</div>
```

Brilliant? Yes. Hacky? Also yes. But it works perfectly.

## AI-Assisted Development: The MCP Server

### What is MCP?

Model Context Protocol—a way for AI to directly interact with external services.

The Supabase MCP Server lets AI:

- Read your database schema
- Execute SQL queries
- Create migrations
- Generate RLS policies

### Example: Generating RLS Policies

Instead of writing complex PostgreSQL policies by hand, I used AI:

```
Create RLS policies for chat_room_member that allow:
- Users to join public rooms
- Users to leave any room they're in
- Users to read their own memberships
```

AI + MCP generated:

```sql
CREATE POLICY "users_join_public_rooms"
ON chat_room_member FOR INSERT
WITH CHECK (
  EXISTS (
    SELECT 1 FROM chat_room
    WHERE id = chat_room_id AND is_public = true
  )
);

CREATE POLICY "users_leave_rooms"
ON chat_room_member FOR DELETE
USING (member_id = auth.uid());

CREATE POLICY "users_read_own_memberships"
ON chat_room_member FOR SELECT
USING (member_id = auth.uid());
```

Perfect. Saved me 30 minutes of documentation-reading and debugging.

## Problems Solved & Lessons Learned

### Problem 1: Race Conditions in Optimistic Updates

**Issue:** User sends message A, then message B quickly. If B completes before A, messages appear out of order.

**Solution:** Generate UUIDs on client. Server uses client-provided ID. Order preserved.

### Problem 2: Duplicate Messages

**Issue:** Message appears twice—once optimistically, once from real-time broadcast.

**Solution:** Filter optimistic messages once they appear in real-time:

```typescript
sentMessages.filter(msg => 
  !realtimeMessages.find(m => m.id === msg.id)
)
```

### Problem 3: Permission Denied Errors

**Issue:** Users couldn't send messages despite being room members.

**Root Cause:** Database trigger used `SECURITY INVOKER` (default). New users don't have permissions yet.

**Solution:** Change to `SECURITY DEFINER`. Trigger runs with elevated permissions.

### Problem 4: Stale Presence Data

**Issue:** User closes tab, but presence shows them as "online" for 30 seconds.

**Partial Solution:** Supabase automatically removes presence after timeout. Can't improve further without WebSocket ping/pong.

## Performance Optimization

### Type Safety with Generated Types

```bash
npx supabase gen types typescript --project-id YOUR_ID > types/database.ts
```

This generates TypeScript types from your database schema. Autocomplete for tables, columns, and relationships.

### Caching with React Cache

```typescript
import { cache } from 'react';

export const getCurrentUser = cache(async () => {
  const supabase = createClient();
  const { data: { user } } = await supabase.auth.getUser();
  return user;
});
```

Multiple components can call `getCurrentUser()` in the same render. It only executes once.

### Debouncing Real-Time Updates

If messages arrive faster than 100ms, batch them:

```typescript
const [pendingMessages, setPendingMessages] = useState<Message[]>([]);

useEffect(() => {
  const timer = setTimeout(() => {
    setMessages(prev => [...prev, ...pendingMessages]);
    setPendingMessages([]);
  }, 100);

  return () => clearTimeout(timer);
}, [pendingMessages]);
```

Reduces re-renders, improves performance.

## What I'd Do Differently

### 1. Add Message Editing/Deletion

Currently messages are immutable. Adding edit history and soft deletes would improve UX.

### 2. Implement Direct Messages

Right now it's only rooms. 1-on-1 DMs are a natural extension.

### 3. File Uploads

Text-only is limiting. Would add Supabase Storage for images/files.

### 4. Better Mobile Experience

Works on mobile, but could benefit from native apps using React Native + Supabase.

### 5. Message Search

With infinite scroll, finding old messages is hard. Full-text search with PostgreSQL `tsvector` would help.

## Key Takeaways

1. **Don't Build What You Can Buy**: WebSockets, presence, real-time are solved problems. Use Supabase.
    
2. **Database Triggers Are Powerful**: Move logic to the database. It's closer to the data and more reliable.
    
3. **Optimistic Updates Are Non-Negotiable**: For real-time apps, perceived performance > actual performance.
    
4. **RLS Is Security**: Don't rely solely on application code. Enforce permissions at the database level.
    
5. **AI + MCP = Productivity**: Writing database functions and RLS policies by hand is tedious. Let AI handle it.
    
6. **Plan Your Schema**: Good schema design prevents headaches. Composite keys, foreign keys, and constraints save debugging time.
    

## Conclusion

Building real-time applications has traditionally been hard. But with modern tools like Supabase, Next.js 15, and AI-assisted development, it's become accessible.

This project taught me that great real-time experiences require:

- Smart optimistic updates
- Proper state management
- Database-level security
- Efficient pagination
- Thoughtful UX design

The result? A chat application that feels instant, scales effortlessly, and costs nothing to host (on free tiers).

Real-time is the future. Now you know how to build it.

---

## Resources

- **Live Demo**: [Your Deployed URL]
- **Source Code**: [Your GitHub Repo]
- **Supabase Docs**: https://supabase.com/docs
- **Next.js 15**: https://nextjs.org
- **Shadcn/ui**: https://ui.shadcn.com

---

_Building real-time apps? What challenges have you faced? Share in the comments!_
