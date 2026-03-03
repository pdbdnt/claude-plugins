---
name: flow-gc-visualizer
description: |
  Group-chat style architecture/flow visualization where system components talk to each other
  like friends in a messaging app. **Load when:**
  - Building chat-style visualizations of system architecture or technical flows
  - Creating "how it works" pages or onboarding flows
  - Adding new flow tabs to a "how it works" page
  - Explaining multi-service interactions as conversations
  - Working on architecture chat visualization components
---

# Flow Group-Chat Visualizer

Visualize system architecture as group chat conversations. Each service/component is an "actor" with a colored avatar, and they exchange messages explaining what they do in plain language.

**Recommended directory:** `components/architecture-chat/`

## Data Model

```typescript
// types.ts (place in your chat visualization directory)
type ChatActor = {
  id: string;
  name: string;
  shortName?: string;       // For tight spaces (legend chips, mobile)
  color: string;            // FULL Tailwind class: "bg-amber-500"
  textColor: string;        // FULL Tailwind class: "text-amber-950"
  borderColor: string;      // FULL Tailwind class: "border-amber-500/40"
  icon: LucideIcon;
  description?: string;     // Tooltip text
};

type ChatMessage = {
  actorId: string | "user" | "narrator";  // "user" and "narrator" are reserved
  content: string;
  delay?: number;           // Override stagger (default: index * 150ms)
};

type ChatFlow = {
  id: string;
  title: string;
  subtitle?: string;
  emoji?: string;
  actors: ChatActor[];
  messages: ChatMessage[];
};
```

## Critical Rules

### Tailwind Class Safety
**NEVER** construct Tailwind classes dynamically. Store full class strings:
```typescript
// CORRECT - JIT compiler sees the full string
color: "bg-violet-500"

// WRONG - JIT cannot resolve this
color: `bg-${colorName}-500`
```

### Three Message Types
- **`"user"`** - Right-aligned orange bubble. The end user's action/request.
- **`"narrator"`** - Centered italic text with divider lines. Scene transitions, time skips.
- **Actor ID** - Left-aligned glass bubble with colored border + avatar. System speaking.

### Animation Pattern
CSS stagger via `tw-animate-css` + inline `animationDelay`:
```tsx
className={cn(
  "flex ...",
  animated && "animate-in fade-in slide-in-from-bottom-3 duration-500"
)}
style={animated ? {
  animationDelay: `${index * 150}ms`,
  animationFillMode: "both"  // KEY: holds invisible state until delay fires
} : undefined}
```

### No ScrollArea
Let content determine height naturally. The page scrolls, not the chat container. Remove fixed heights on the card wrapper.

### Tab Re-Animation
Use `data-[state=inactive]:hidden` on TabsContent. When a tab becomes active again, CSS animations replay automatically (visibility change re-triggers them).

## Component Hierarchy

```
ArchitectureChatPage        - Stateful: manages active tab
  Tabs (shadcn)
    TabsTrigger per flow    - Emoji + title, underline active
    TabsContent per flow
      ChatFlow              - Assembles header + messages
        FlowHeader          - Title, subtitle, actor legend chips
        ChatMessageList     - Maps messages to components
          UserMessage       - Orange gradient, right-aligned
          NarratorMessage   - Centered italic with dividers
          ActorMessage      - Glass bubble, left-aligned
            ActorAvatar     - Colored circle with Lucide icon
```

## Adding a New Flow

1. Create a flow file (e.g., `flows/my-flow.ts`) in your visualization directory
2. Define actors array (reuse existing actor definitions when possible)
3. Write messages array with `actorId` references
4. Export from your flows index and add to the `allFlows` array
5. Use phase comments for flows with 10+ messages: `// --- PHASE 1: NAME ---`

## Example Actor Color Palette

| Actor | Color | Border |
|-------|-------|--------|
| Your App | `bg-amber-500` | `border-amber-500/40` |
| AI Model | `bg-blue-500` | `border-blue-500/40` |
| Database | `bg-emerald-500` | `border-emerald-500/40` |
| API Gateway | `bg-violet-500` | `border-violet-500/40` |
| Auth Service | `bg-teal-500` | `border-teal-500/40` |
| Payments | `bg-sky-500` | `border-sky-500/40` |
| Queue | `bg-rose-500` | `border-rose-500/40` |
| CDN | `bg-cyan-500` | `border-cyan-500/40` |
| Cache | `bg-yellow-500` | `border-yellow-500/40` |
| Extension | `bg-purple-500` | `border-purple-500/40` |

Pick distinct hue families for new actors. Avoid adjacent hues (e.g., don't add another yellow next to amber).

## Reference Files

- **Component code patterns:** See [references/component-patterns.md](references/component-patterns.md) for implementation details of each component
- **Flow authoring guide:** See [references/flow-authoring.md](references/flow-authoring.md) for writing compelling actor dialogue
