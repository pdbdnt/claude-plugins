# flow-gc-visualizer

A Claude Code skill for building **group-chat style architecture visualizations** — where your system components talk to each other like friends in a messaging app.
## What It Does

Instead of static architecture diagrams, this skill helps you build animated chat conversations between your services. Each service is an "actor" with a colored avatar, and they exchange messages explaining what they do in plain language.

Perfect for:
- **"How It Works" pages** — show visitors how your tech stack collaborates
- **Onboarding flows** — explain system architecture to new developers
- **Documentation** — make complex service interactions easy to understand

## Example

Imagine your app processes an image upload. Instead of a flowchart, you get:

> **User:** "I just uploaded a photo!"
>
> **API Gateway:** "Got it! Validating the file and routing to the processor."
>
> **AI Model:** "Analyzing the image — I see warm tones, minimalist composition, and natural lighting."
>
> **Database:** "Stored the analysis results. Ready for retrieval."
>
> **Your App:** "All done! Here's your style breakdown."

Each message animates in with a staggered delay, creating a live group chat feel.

## Stack

Built for **Next.js + Tailwind CSS + shadcn/ui** projects using:
- Lucide icons for actor avatars
- `tw-animate-css` for staggered entry animations
- shadcn Tabs for multi-flow navigation

## What's Included

| File | Purpose |
|------|---------|
| `SKILL.md` | Data model, critical rules, component hierarchy, color palette |
| `references/component-patterns.md` | Implementation patterns for each React component |
| `references/flow-authoring.md` | Guide for writing compelling actor dialogue and flow structure |

## Quick Start

1. Add the marketplace: `/plugin marketplace add pdbdnt/claude-plugins`
2. Install: `/plugin install flow-gc-visualizer`
3. Ask Claude: *"Create an architecture chat visualization showing how user authentication works in my app"*

The skill guides Claude to generate properly structured flow data with animated chat components.
