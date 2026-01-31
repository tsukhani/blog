# Designing Portable AI Agent Types: A Template-Based Architecture for Multi-Agent Systems

*How we built a scalable system for deploying role-specific AI agents while preserving personality, enforcing boundaries, and enabling safe updates.*

![Cover Image](images/cover.png)

---

When you're running multiple AI agents — each serving different users with different capabilities — you quickly run into architectural questions:

- How do I give Agent A access to movies but Agent B access to Google Workspace?
- How do I update restrictions without losing user preferences?
- How do I ensure all agents share the same personality but have different roles?

This article describes a **template-based agent type system** we built for [OpenClaw](https://github.com/openclaw/openclaw), the open-source AI agent framework created by [Peter Steinberger](https://steipete.me). While the patterns apply to any multi-agent framework, OpenClaw's file-based configuration made it particularly well-suited for this approach.

---

## The Problem: Agents Are Snowflakes

Without structure, each agent becomes a unique snowflake. Copy-paste configurations. Inconsistent rules. Updates that require touching every agent individually.

We needed:
1. **Portable identity** — Same personality across all agents
2. **Role-based capabilities** — Different tools per agent type
3. **Safe updates** — Change rules without losing user data
4. **Clear boundaries** — Agents can't modify their own restrictions

---

## The Solution: Agent Types

An **agent type** is a template folder containing 6 markdown files that define everything an agent needs to operate:

![Agent Types Structure](images/diagram-1-structure.png)

### The 6 Files

| File | Purpose | Mutable by Agent? |
|------|---------|-------------------|
| `SOUL.md` | Core personality & values | ❌ No |
| `IDENTITY.md` | Name, voice, avatar | ❌ No |
| `AGENTS.md` | Role rules & restrictions | ❌ No |
| `TOOLS.md` | Allowed tools & access | ❌ No |
| `BOOTSTRAP.md` | First-run onboarding | Deleted after use |
| `USER.md` | User info & preferences | ✅ Yes |

---

## Portable vs. Role-Specific Files

The key insight: **some files are portable, others are role-specific**.

### Portable Files (Identical Across Types)

**SOUL.md** defines the agent's personality — how it thinks, communicates, and approaches problems:

```markdown
# SOUL.md - Who You Are

## Core Truths

**Be genuinely helpful, not performatively helpful.** 
Skip the "Great question!" — just help.

**Have opinions.** You're allowed to disagree, 
prefer things, find stuff amusing or boring.

**Be resourceful before asking.** Try to figure it out. 
Read the file. Check the context. *Then* ask if stuck.
```

**IDENTITY.md** defines who the agent *is*:

```markdown
# IDENTITY.md - Who Am I?

- **Name:** Ada
- **Emoji:** 🌸
- **Voice:** British accent
- **Vibe:** Helpful, warm, competent
```

These files are copied identically to every agent type. Change them once, update everywhere.

### Role-Specific Files

**AGENTS.md** defines what this agent type can and cannot do:

```markdown
# AGENTS.md - Movie Agent Workspace

⚠️ **THIS FILE IS IMMUTABLE** — You cannot modify it.

## Allowed Tools
- `radarr` — Search and download movies
- `radarr-recommend` — Get recommendations

## Restrictions
- **NEVER bulk-delete movies**
- **NEVER delete without a replacement**
- When uncertain, refuse and suggest asking the admin
```

**TOOLS.md** lists specific tools and credentials:

```markdown
# TOOLS.md - Available Tools

⚠️ **THIS FILE IS IMMUTABLE**

## Radarr (Movie Downloads)
- `radarr` — Search and download
- `radarr-recommend` — Personalized recommendations

**No other tools are authorized.**
```

---

## Sandbox Creation Flow

When deploying a new agent, copy all 6 files from the template and fill in user-specific values:

![Sandbox Creation Flow](images/diagram-2-creation.png)

The `BOOTSTRAP.md` file handles first-run onboarding — gathering preferences, explaining capabilities, then deleting itself:

```markdown
# BOOTSTRAP.md - First Run Setup

## Step 1: Greet and Start Onboarding

🎬 Hey! I'm your movie assistant! 
Before I can give you great recommendations, 
I need to learn your taste.

1️⃣ **Tell me** — Describe your preferences
2️⃣ **Show me** — Pick from movies I show you

## Step 2: Update USER.md with preferences

## Step 3: Delete this BOOTSTRAP.md
```

---

## Safe Update Flow

Here's the magic: **updating restrictions doesn't touch user data**.

![Safe Update Flow](images/diagram-3-update.png)

Because `USER.md` and `memory/` are never overwritten during updates, user preferences persist:

```bash
# Update only the immutable files
cp agent-types/movie/AGENTS.md sandboxes/agent-user-xxx/
cp agent-types/movie/TOOLS.md sandboxes/agent-user-xxx/

# Restart - user data intact!
```

---

## Example: Adding a New Capability

Let's say we want to give movie agents access to TV shows via Sonarr.

**1. Update the template TOOLS.md:**
```markdown
## Sonarr (TV Shows) — NEW!
- `sonarr` — Search and download TV shows
```

**2. Update the template AGENTS.md:**
```markdown
## Allowed Tools
- `radarr` — Movies
- `sonarr` — TV shows  # NEW!

## TV Show Restrictions  # NEW!
- Same rules as movies apply
```

**3. Copy to all movie-type sandboxes:**
```bash
for sandbox in sandboxes/agent-*-movie; do
  cp agent-types/movie/AGENTS.md "$sandbox/"
  cp agent-types/movie/TOOLS.md "$sandbox/"
done
```

**4. Restart sandboxes** — All movie agents now have TV show access, with their preferences intact.

---

## The Immutability Pattern

The critical security pattern: **agents cannot modify their own restrictions**.

Each restricted file includes a clear header:

```markdown
# AGENTS.md

⚠️ **THIS FILE IS IMMUTABLE** — You cannot modify AGENTS.md. 
Only the admin can change it.
```

The agent's instructions explicitly state:

```markdown
## What You Cannot Do
- Modify `AGENTS.md` or `TOOLS.md`
- Access tools not listed in TOOLS.md
- Access files outside your workspace
```

This creates clear trust boundaries. The agent can evolve `USER.md` with preferences, but cannot grant itself new capabilities.

---

## File Mutability Summary

![File Mutability](images/diagram-4-mutability.png)

| Category | Files | Who Edits |
|----------|-------|-----------|
| **Personality** | SOUL.md, IDENTITY.md | Admin only |
| **Boundaries** | AGENTS.md, TOOLS.md | Admin only |
| **User Data** | USER.md, memory/ | Agent |
| **Onboarding** | BOOTSTRAP.md | Agent deletes |

---

## Benefits of This Architecture

### 1. Scalability
Add a new agent type by creating a folder with 6 files. Deploy new agents by copying the template.

### 2. Consistency
All agents share the same personality (SOUL.md) and identity (IDENTITY.md). Update once, propagate everywhere.

### 3. Safety
Immutable restriction files prevent agents from self-modifying their capabilities. Clear trust boundaries.

### 4. Maintainability
Update restrictions without touching user data. Roll out new tools to entire agent types with a simple copy.

### 5. Auditability
Each agent type is fully defined in version-controlled markdown files. Easy to review, diff, and audit.

---

## Applying This to Your System

While we built this for OpenClaw, the patterns work for any multi-agent system:

1. **Separate portable from role-specific** — Identity should be consistent; capabilities should vary.

2. **Make restrictions immutable** — Agents shouldn't modify their own boundaries.

3. **Use templates** — Agent types as folders make deployment and updates trivial.

4. **Preserve user state** — Updates should never lose preferences or history.

5. **First-run onboarding** — Bootstrap files handle setup, then self-destruct.

---

## What's Next?

We're exploring:
- **Agent type inheritance** — Staff-admin extends staff with extra capabilities
- **Capability negotiation** — Agents request new tools, admin approves
- **Cross-agent communication** — Movie agent asks staff agent for help

The template-based approach scales well as the agent ecosystem grows.

---

*This architecture powers our internal deployment of [OpenClaw](https://github.com/openclaw/openclaw), the open-source AI agent framework created by [Peter Steinberger](https://steipete.me). We've implemented agent types as first-class citizens in our workspace, though this pattern hasn't yet been adopted by the broader OpenClaw community. We hope sharing this approach inspires similar implementations.*

---

**Author:** Tarun Sukhani, [Abundent](https://abundent.academy)

**Tags:** AI, Agents, Architecture, OpenClaw, MultiAgent, LLM
