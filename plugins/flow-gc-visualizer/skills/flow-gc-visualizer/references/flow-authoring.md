# Flow Authoring Guide

## Table of Contents
- Flow Structure
- Actor Voice Guidelines
- Message Length Rules
- Narrator Usage
- Phase Structure
- Emoji Guidelines
- Example: Minimal Flow
- Example: Multi-Phase Flow
- Checklist

## Flow Structure

Every flow needs:
1. **id** - kebab-case, unique across all flows
2. **title** - Short, action-oriented (e.g., "Analyzing Your Image")
3. **subtitle** - One-line explanation of what the flow demonstrates
4. **emoji** - Single emoji for tab label
5. **actors** - Array of services involved (define BEFORE messages)
6. **messages** - Array of conversation messages

## Actor Voice Guidelines

Each actor should have a consistent personality:

- **App/Orchestrator (Your App)**: Enthusiastic, coordinating. Delegates to specialists. Uses the brand voice.
- **AI Models** (GPT, Gemini, Claude, etc.): Technical but accessible. Shows what they "see" or "think". Shares specific details.
- **Infrastructure** (Queue, CDN, Storage): Matter-of-fact, reliable. Reports status clearly.
- **Business Services** (Payments, Credits): Professional, reassuring. Handles money/permissions.
- **User Tools** (Extension, Tag System): Helpful, quick. Acknowledges user action immediately.

Rules:
- Actors address each other by name (creates the group chat feel)
- Actors describe WHAT they're doing, not HOW (no code, no API details)
- Keep personality consistent across flows (same actor = same voice)

## Message Length Rules

- **User messages**: 1-2 short sentences. Conversational, may include emoji.
- **Actor messages**: 2-4 sentences max. Be specific but scannable.
- **Narrator messages**: 1 sentence. Brief scene transition.
- **Never** write paragraph-length messages. If an actor needs to say a lot, split into two messages.

## Narrator Usage

Use narrator messages for:
- **Scene transitions**: "You head over to /create..."
- **Time skips**: "A few seconds pass..."
- **User actions**: "You enter your card details on the secure payment page..."
- **Phase breaks**: Between major sections of a long flow

Do NOT use narrator for:
- Information that an actor could say
- Technical explanations
- Multiple narrator messages in a row

## Phase Structure (for 10+ message flows)

Organize with comment markers:
```typescript
messages: [
  // --- PHASE 1: SAVE ---
  { actorId: "user", content: "..." },
  { actorId: "extension", content: "..." },

  // --- PHASE 2: ANALYZE ---
  { actorId: "narrator", content: "..." },  // transition
  { actorId: "user", content: "..." },
  ...
]
```

Each phase should:
- Start with a user action or narrator transition
- Include 3-6 messages
- End with the orchestrator summarizing the result

## Emoji Guidelines

- Flow emoji (tab label): One emoji that captures the flow's essence
- In messages: Use sparingly. 1-2 per actor message max.
- User messages: Can be more emoji-heavy (feels natural)
- Narrator: Never use emoji
- Don't repeat the same emoji in consecutive messages

## Example: Minimal Flow

```typescript
export const myFlow: ChatFlow = {
  id: "my-flow",
  title: "My Flow Title",
  subtitle: "What this flow demonstrates",
  emoji: "icon",
  actors: [
    {
      id: "app",
      name: "My App",
      shortName: "App",
      color: "bg-amber-500",
      textColor: "text-amber-950",
      borderColor: "border-amber-500/40",
      icon: Sparkles,
    },
    {
      id: "service",
      name: "My Service",
      shortName: "Service",
      color: "bg-blue-500",
      textColor: "text-blue-950",
      borderColor: "border-blue-500/40",
      icon: Server,
    },
  ],
  messages: [
    { actorId: "user", content: "User's request" },
    { actorId: "app", content: "App delegates to the service" },
    { actorId: "service", content: "Service does its thing and reports back" },
    { actorId: "app", content: "App summarizes the result for the user" },
  ],
};
```

## Example: Multi-Phase Flow

For a complete multi-phase example, see your flows directory. A typical pattern:

Save -> Analyze -> Generate -> Create

## Checklist

Before submitting a new flow:

- [ ] All actor IDs in messages match an actor in the actors array
- [ ] No two consecutive messages from the same actor (break them up)
- [ ] User speaks first (or narrator sets the scene, then user speaks)
- [ ] The orchestrator summarizes at the end
- [ ] Narrator used only for transitions, not information
- [ ] Messages are scannable (2-4 sentences each, not walls of text)
- [ ] Actor colors are distinct from each other (check the palette table in SKILL.md)
- [ ] Exported from your flows index and added to the allFlows array
