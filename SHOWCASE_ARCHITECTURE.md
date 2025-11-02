# Claude Code Infrastructure Architecture

How the components work together to create an intelligent development environment.

## Table of Contents
- [System Overview](#system-overview)
- [Component Interaction](#component-interaction)
- [Auto-Activation Flow](#auto-activation-flow)
- [Skill Architecture](#skill-architecture)
- [Hook System](#hook-system)
- [Agent System](#agent-system)
- [Git Integration](#git-integration)
- [Sync Management](#sync-management)

---

## System Overview

The infrastructure consists of four main subsystems that work together:

```
┌─────────────────────────────────────────────────────────┐
│                   User Interaction                       │
│              (Typing, Editing Files, Committing)        │
└──────────────────┬──────────────────────────────────────┘
                   │
         ┌─────────┴─────────┐
         │                   │
    ┌────▼──────┐      ┌────▼────────┐
    │  Hooks    │      │ Git Hooks   │
    │  System   │      │  (CI/CD)    │
    └────┬──────┘      └─────────────┘
         │
    ┌────▼────────────┐
    │ skill-rules.json│
    │   (Triggers)    │
    └────┬────────────┘
         │
    ┌────▼─────┐
    │  Skills  ├──────┐
    └──────────┘      │
                 ┌────▼──────┐
                 │  Agents   │
                 └───────────┘
```

---

## Component Interaction

### 1. User Types → Hooks Analyze → Skills Activate

**Flow:**
```
User: "Create a new API endpoint for user authentication"
                    ↓
    skill-activation-prompt hook runs
                    ↓
    Checks skill-rules.json:
    - Keywords: "API", "endpoint"
    - Intent: "create.*endpoint"
                    ↓
    Matches: backend-dev-guidelines skill
                    ↓
    Suggests skill to Claude
                    ↓
    Claude loads skill and responds with patterns
```

### 2. User Edits File → Hook Tracks → Context Updated

**Flow:**
```
User edits: src/app/api/auth/route.ts
                    ↓
    post-tool-use-tracker hook runs
                    ↓
    Checks file path against skill-rules.json
                    ↓
    Path matches: src/app/api/**/*.ts
                    ↓
    Suggests: backend-dev-guidelines
                    ↓
    Skill activates if not already active
```

### 3. User Commits → Git Hooks → Quality Checks

**Flow:**
```
User: git commit -m "feat(api): add auth endpoint"
                    ↓
    pre-commit hook runs
                    ↓
    TypeScript check: tsc --noEmit
    ESLint check: eslint src/**/*.ts
                    ↓
    commit-msg hook runs
                    ↓
    Validates message format
                    ↓
    Commit succeeds/fails based on checks
```

### 4. User Requests Complex Task → Agent Handles

**Flow:**
```
User: "Review this code for architectural issues"
                    ↓
    Claude recognizes complex task
                    ↓
    Launches code-architecture-reviewer agent
                    ↓
    Agent:
    - Reads code
    - Analyzes patterns
    - Compares to best practices
    - Generates report
                    ↓
    Returns comprehensive review to user
```

---

## Auto-Activation Flow

The core feature that makes skills "just work."

### Phase 1: UserPromptSubmit Hook

```typescript
// Simplified flow
UserPromptSubmit Hook:
1. User types prompt
2. Hook receives prompt text
3. Hook runs skill-activation-prompt.ts
4. Script analyzes:
   - Keywords in prompt
   - Intent patterns
   - Current file context
5. Checks skill-rules.json for matches
6. Returns matching skills
7. Claude receives skill suggestions
8. Claude loads relevant skills
```

**Example:**

```
User prompt: "How do I implement rate limiting in my API?"

Hook analysis:
- Keywords found: "API", "rate limiting"
- Current file: src/app/api/users/route.ts
- Path pattern match: src/app/api/**/*.ts

skill-rules.json check:
{
  "backend-dev-guidelines": {
    "keywords": ["API", "rate limiting", ...],
    "fileTriggers": {
      "pathPatterns": ["src/app/api/**/*.ts"]
    }
  }
}

Result: ✅ Activate backend-dev-guidelines
```

### Phase 2: PostToolUse Hook

```typescript
// Simplified flow
PostToolUse Hook:
1. User edits file via Edit/Write tool
2. Hook receives file path
3. Hook runs post-tool-use-tracker.sh
4. Script checks:
   - File path against skill patterns
   - File content against content patterns
5. Matches against skill-rules.json
6. Returns relevant skills
7. Skills activate for subsequent prompts
```

**Example:**

```
User edits: client/src/components/UserCard.tsx

Hook analysis:
- File path: client/src/components/UserCard.tsx
- Pattern match: src/components/**/*.tsx

skill-rules.json check:
{
  "frontend-dev-guidelines": {
    "fileTriggers": {
      "pathPatterns": ["src/components/**/*.tsx"],
      "contentPatterns": ["useState", "useEffect"]
    }
  }
}

Result: ✅ Activate frontend-dev-guidelines
```

---

## Skill Architecture

Skills follow a modular pattern for scalability.

### Structure

```
skill-name/
├── SKILL.md                    # Main file (<500 lines)
├── resources/
│   ├── topic-1.md              # Deep dive 1
│   ├── topic-2.md              # Deep dive 2
│   └── topic-3.md              # Deep dive 3
└── SKILL_SUMMARY.md (optional)
```

### Loading Strategy

**Progressive Disclosure:**

```
1. Claude loads SKILL.md first
   ↓
2. SKILL.md contains:
   - Overview
   - Quick reference
   - Links to resources
   ↓
3. User asks specific question
   ↓
4. Claude reads relevant resource file
   ↓
5. Returns detailed answer
```

**Why this works:**
- SKILL.md stays under 500 lines → doesn't hit context limits
- Resources loaded only when needed → efficient context use
- User gets quick answers first → better UX

### Skill Lifecycle

```
Created → Activated → Used → Updated → Maintained

Created:
- Written by developer or AI
- Follows 500-line rule
- Added to skill-rules.json

Activated:
- Hooks match trigger patterns
- Claude loads skill
- Skill provides context

Used:
- Developer asks questions
- Claude references skill patterns
- Code generated follows patterns

Updated:
- Patterns evolve
- Examples updated
- Resources added

Maintained:
- Synced from showcase (if generic)
- Kept current with codebase
- Outdated patterns removed
```

---

## Hook System

Hooks provide automation at key moments.

### Hook Types

**1. UserPromptSubmit**
- **Trigger:** Every user prompt
- **Purpose:** Analyze and suggest skills
- **Performance:** Must be fast (<100ms)
- **Essential:** YES

**2. PostToolUse**
- **Trigger:** After file edits
- **Purpose:** Track file context, suggest skills
- **Performance:** Fast (<50ms)
- **Essential:** YES

**3. Stop**
- **Trigger:** Before builds/compilations
- **Purpose:** Prevent errors, enforce quality
- **Performance:** Can be slower (1-5s)
- **Essential:** NO (optional)

### Hook Execution Flow

```
User Action
    ↓
Hook Trigger
    ↓
Run Command (bash script or TypeScript)
    ↓
Process Results
    ↓
Return to Claude
    ↓
Claude Acts on Results
```

### Hook Configuration

**settings.json structure:**

```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/skill-activation-prompt.sh"
          }
        ]
      }
    ]
  }
}
```

**Execution:**
1. Claude detects user prompt
2. Reads hooks configuration
3. Runs each command in sequence
4. Collects output
5. Uses output to inform response

---

## Agent System

Agents handle complex, multi-step tasks autonomously.

### Agent vs. Direct Response

**Direct Response (Normal Mode):**
```
User: "Fix this function"
  → Claude analyzes
  → Claude responds immediately
  → One-shot fix
```

**Agent (Complex Task Mode):**
```
User: "Review entire codebase for architectural issues"
  → Claude launches code-architecture-reviewer agent
  → Agent:
    1. Reads multiple files
    2. Analyzes patterns
    3. Compares to best practices
    4. Generates comprehensive report
  → Agent returns final report
  → Claude presents findings
```

### Agent Lifecycle

```
Defined → Invoked → Executes → Returns

Defined:
- Markdown file in .claude/agents/
- Describes task and approach
- Lists available tools

Invoked:
- User requests complex task
- Claude recognizes agent needed
- Launches agent

Executes:
- Agent reads context
- Performs analysis
- Uses tools (Read, Grep, etc.)
- Generates findings

Returns:
- Agent completes task
- Returns results to Claude
- Claude formats for user
```

### Agent Architecture

```markdown
---
name: agent-name
description: What the agent does
---

# Agent Purpose
Brief description

## When to Use
Specific scenarios

## How It Works
1. Step 1
2. Step 2
3. Step 3

## Examples
Concrete examples

## Output Format
What user receives
```

---

## Git Integration

Git hooks provide CI/CD quality gates locally.

### Hook Chain

```
git commit
    ↓
pre-commit hook
    ├─ TypeScript check
    ├─ ESLint check
    └─ Format check
    ↓
commit-msg hook
    └─ Message validation
    ↓
Commit created
    ↓
git push
    ↓
pre-push hook
    ├─ Run tests
    ├─ Run builds
    └─ Integration checks
    ↓
Push to remote
```

### Hook Installation

```
.claude/git-hooks/
├── pre-commit           # Symlinked to .git/hooks/pre-commit
├── pre-push             # Symlinked to .git/hooks/pre-push
├── commit-msg           # Symlinked to .git/hooks/commit-msg
└── install.sh           # Creates symlinks
```

**Why symlinks:**
- Git hooks must be in `.git/hooks/`
- `.git/` is not version controlled
- Symlinks allow versioning source files in `.claude/git-hooks/`
- Team shares same hooks

---

## Sync Management

Tracking what's synced vs. custom across projects.

### Two-Way Tracking

**Showcase → Projects (INTEGRATION_REGISTRY.json):**
```json
{
  "projects": {
    "project-name": {
      "components": {
        "agents": {...},
        "skills": {...}
      }
    }
  }
}
```

**Projects → Showcase (SYNC_MANIFEST.json):**
```json
{
  "syncedFiles": {
    ".claude/agents/auto-error-resolver.md": {
      "syncStrategy": "overwrite-safe"
    }
  }
}
```

### Sync Strategies

**overwrite-safe:**
- File can be updated from showcase
- No custom modifications
- Safe to overwrite

**manual-review:**
- File may have customizations
- Review changes before syncing
- Merge conflicts possible

**never-overwrite:**
- Fully custom file
- Never sync from showcase
- Project-specific

### Sync Flow

```
Showcase Updated
    ↓
Developer runs /sync-infrastructure
    ↓
Command reads SYNC_MANIFEST.json
    ↓
Compares files with showcase
    ↓
Shows available updates
    ↓
Developer selects files to sync
    ↓
Updates applied based on strategy
    ↓
SYNC_MANIFEST.json updated
```

---

## Data Flow Diagram

Complete system interaction:

```
┌─────────────┐
│    User     │
└──────┬──────┘
       │
       ├─ Types Prompt ──────┐
       │                     │
       ├─ Edits File ────────┤
       │                     │
       └─ Commits Code ──────┤
                             │
                    ┌────────▼────────┐
                    │  Hook System    │
                    │                 │
                    │ UserPromptSubmit│
                    │ PostToolUse     │
                    │ Git Hooks       │
                    └────────┬────────┘
                             │
                    ┌────────▼─────────┐
                    │ skill-rules.json │
                    │ (Pattern Matching│
                    └────────┬─────────┘
                             │
                   ┌─────────┴──────────┐
                   │                    │
          ┌────────▼─────┐     ┌───────▼────────┐
          │    Skills    │     │    Agents      │
          │              │     │                │
          │ Load Context │     │ Complex Tasks  │
          │ Provide Help │     │ Multi-Step     │
          └──────┬───────┘     └───────┬────────┘
                 │                     │
                 └──────────┬──────────┘
                            │
                   ┌────────▼────────┐
                   │  Claude Response│
                   └────────┬────────┘
                            │
                   ┌────────▼────────┐
                   │   User Result   │
                   └─────────────────┘
```

---

## Performance Considerations

### Hook Performance

**Critical (must be fast):**
- skill-activation-prompt: <100ms
- post-tool-use-tracker: <50ms

**Why:** Run on every interaction, slow hooks = bad UX

**Optimization:**
- Use bash scripts (faster startup)
- Cache skill-rules.json parsing
- Minimal file I/O

**Acceptable (can be slower):**
- Git hooks: 1-30s (user expects wait)
- Agents: 10-60s (complex tasks)

### Skill Loading

**Lazy loading:**
- Don't load all skills upfront
- Load on-demand via hooks
- Progressive disclosure reduces context usage

**Context management:**
- Main SKILL.md: ~300-400 lines (fits in context)
- Resources: Loaded only when needed
- Total available: Unlimited (via resources)

---

## Security Considerations

### Hook Execution

**Hooks run arbitrary code:**
- Review hooks before installation
- Only use trusted sources
- Verify scripts don't expose secrets

**Sandboxing:**
- Hooks run in project context
- Have access to project files
- Can execute system commands
- **Review carefully before enabling**

### Sensitive Data

**Never commit:**
- API keys in skills/agents
- Environment-specific paths
- Company-proprietary patterns (if applicable)

**Use placeholders:**
```typescript
// ✅ Good
const apiKey = process.env.API_KEY;

// ❌ Bad
const apiKey = "sk-123456789";
```

---

## Debugging the System

### Skill Not Activating

**Check:**
1. Is skill-rules.json valid JSON?
2. Do trigger patterns match your prompt/files?
3. Are hooks installed and executable?
4. Check Claude Code logs for hook errors

**Debug:**
```bash
# Test skill-activation hook manually
./.claude/hooks/skill-activation-prompt.sh "create API endpoint"

# Check if patterns match
cat .claude/skills/skill-rules.json | jq '.skills["backend-dev-guidelines"]'
```

### Hook Not Running

**Check:**
1. Is hook executable? (`chmod +x`)
2. Is settings.json configured correctly?
3. Are hook dependencies installed? (`npm install` in .claude/hooks)

**Debug:**
```bash
# Test hook directly
./.claude/hooks/post-tool-use-tracker.sh "src/app/api/route.ts"
```

### Git Hook Failing

**Check:**
1. Are hooks symlinked correctly?
2. Do checks pass when run manually?
3. Are paths correct for your project structure?

**Debug:**
```bash
# Check symlinks
ls -la .git/hooks/{pre-commit,pre-push,commit-msg}

# Run checks manually
cd server && pnpm exec tsc --noEmit
```

---

## Extension Points

The architecture is designed for extension:

**Add New Skills:**
1. Create skill directory
2. Write SKILL.md
3. Add to skill-rules.json
4. Test activation

**Add New Agents:**
1. Create agent.md
2. Define purpose and approach
3. Copy to .claude/agents/
4. Use when needed

**Add New Hooks:**
1. Write hook script
2. Add to settings.json
3. Test execution
4. Document behavior

**Add Git Hooks:**
1. Create hook script
2. Make executable
3. Test with actual commits
4. Document for team

---

## Summary

The infrastructure works through:

1. **Hooks** monitor user actions
2. **skill-rules.json** defines activation patterns
3. **Skills** provide context and patterns
4. **Agents** handle complex tasks
5. **Git hooks** enforce quality
6. **Sync system** maintains consistency

All components work together to create an intelligent development environment that **anticipates needs** and **automates quality**.

For implementation details, see:
- [INTEGRATION_GUIDE.md](INTEGRATION_GUIDE.md) - How to integrate
- [BEST_PRACTICES.md](BEST_PRACTICES.md) - How to use effectively
- Component-specific READMEs in `.claude/` directories
