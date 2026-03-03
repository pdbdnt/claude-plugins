# Component Patterns Reference

## Table of Contents
- ActorAvatar
- UserMessage
- NarratorMessage
- ActorMessage
- ChatMessageList
- FlowHeader
- ChatFlow
- ArchitectureChatPage

## ActorAvatar

Colored circle with Lucide icon. Used in messages and legend.

```tsx
// Props: actor: ChatActor, size?: "sm" | "md"
<div className={cn(
  "rounded-full flex items-center justify-center flex-shrink-0",
  "shadow-lg ring-1 ring-black/20",
  size === "sm" ? "w-6 h-6" : "w-9 h-9",
  actor.color, actor.textColor
)}>
  <Icon size={size === "sm" ? 12 : 16} strokeWidth={2.5} />
</div>
```

## UserMessage

Right-aligned orange gradient bubble with "YOU" avatar.

Key styling:
- Bubble: `bg-gradient-to-br from-orange-500 via-orange-500 to-amber-600`
- Shadow: `shadow-xl shadow-orange-600/30`
- Corner: `rounded-2xl rounded-br-md` (sharp bottom-right like iMessage)
- Avatar: Dark circle with "YOU" text, `ring-2 ring-orange-400/40`
- Label: `text-[10px] font-bold tracking-[0.15em] uppercase text-orange-300/50`

## NarratorMessage

Centered italic text with gradient divider lines on each side.

```tsx
<div className="flex items-center gap-4 max-w-[90%]">
  <div className="flex-1 h-px bg-gradient-to-r from-transparent via-orange-400/25 to-transparent" />
  <p className="text-[12px] italic text-orange-200/45 text-center font-medium">
    {content}
  </p>
  <div className="flex-1 h-px bg-gradient-to-r from-transparent via-orange-400/25 to-transparent" />
</div>
```

## ActorMessage

Left-aligned glass bubble with colored border.

Key styling:
- Avatar positioned with `mt-6` (aligns with bubble top, below name label)
- Name label: `text-[10px] font-bold tracking-[0.15em] uppercase text-orange-200/50`
- Bubble background: `rgba(255,255,255,0.06)` to `rgba(255,255,255,0.02)` (inline gradient)
- Bubble border: `actor.borderColor` (e.g., `border-violet-500/40`)
- Corner: `rounded-2xl rounded-tl-md` (sharp top-left, opposite of user)
- Text: `text-[14px] text-orange-100/85`

## ChatMessageList

Maps flow.messages to the correct component. No ScrollArea (content flows naturally).

```tsx
const STAGGER_MS = 150;
const actorMap = useMemo(() => new Map(flow.actors.map(a => [a.id, a])), [flow.actors]);

// Dispatch: "user" -> UserMessage, "narrator" -> NarratorMessage, else -> ActorMessage
// Each gets: animated, animationDelay = message.delay ?? index * STAGGER_MS
```

## FlowHeader

Title + subtitle + horizontal actor legend.

```tsx
<div className="px-5 py-4 md:px-8 md:py-5 border-b border-white/[0.06]">
  <h2>emoji + title</h2>
  <p>subtitle (text-[13px] text-orange-200/45)</p>
  <div className="flex gap-1.5 overflow-x-auto scrollbar-hide">
    {actors.map(actor => (
      <chip: ActorAvatar size="sm" + name label>
    ))}
  </div>
</div>
```

## ChatFlow

Simple wrapper: `<div><FlowHeader /><ChatMessageList /></div>`

## ArchitectureChatPage

Stateful root. Manages active tab via `useState`.

Tab styling:
- TabsTrigger: `text-[13px] font-semibold px-5 py-3.5 rounded-none border-b-2`
- Active: `border-orange-400 text-orange-100 bg-white/[0.03]`
- Inactive: `text-orange-300/40 border-transparent`
- Mobile: emoji only shown (title hidden with `hidden sm:inline`)

Page integration (how-it-works):
```tsx
<div className="rounded-2xl overflow-hidden shadow-2xl shadow-black/40"
  style={{ background: "linear-gradient(180deg, rgba(8,6,4,0.85) 0%, rgba(12,10,7,0.92) 100%)",
           border: "1px solid rgba(255,255,255,0.06)" }}>
  <ArchitectureChatPage flows={allFlows} animated={true} />
</div>
```
