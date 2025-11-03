# Claude Code Infrastructure - Integration Guide

Quick-start guide for integrating this infrastructure into your projects.

## Overview

This showcase provides production-tested Claude Code infrastructure that can be integrated into any TypeScript/JavaScript project. Choose your integration level based on project needs.

## Integration Levels

### Light Integration (Recommended for most projects)
**Time:** ~15 minutes
**Customization:** Minimal
**Best for:** Standard web apps, Next.js, standard backends

**Components:**
- ✓ All agents (10)
- ✓ Generic skills (backend-dev-guidelines, frontend-dev-guidelines)
- ✓ Claude hooks (skill activation, tracking)
- ✓ Git hooks (pre-commit, pre-push, commit-msg)
- ✓ Basic configuration

**Example projects:** orbit (Next.js + Prisma)

### Heavy Integration (For specialized stacks)
**Time:** ~90 minutes
**Customization:** Extensive
**Best for:** Real-time apps, unique architectures, specialized patterns

**Components:**
- ✓ All agents (8-10)
- ✓ Custom skills tailored to tech stack
- ✓ Custom slash commands
- ✓ Claude hooks
- ✓ Git hooks
- ✓ Comprehensive testing/debugging resources

**Example projects:** cards-against-humanity (Express + Socket.IO + Redis)

## Prerequisites

- Git repository
- Node.js 18+ (for hooks)
- pnpm, npm, or yarn
- TypeScript project (recommended)

## Light Integration Steps

### 1. Create Directory Structure

```bash
cd /path/to/your-project
mkdir -p .claude/{agents,hooks,git-hooks,skills,commands}
```

### 2. Copy Agents

```bash
cp /path/to/showcase/.claude/agents/*.md .claude/agents/
```

**Agents included:**
- auto-error-resolver
- frontend-error-fixer
- code-architecture-reviewer
- documentation-architect
- refactor-planner
- web-research-specialist
- plan-reviewer
- code-refactor-master
- auth-route-debugger (if using Keycloak/JWT)
- auth-route-tester (if using Keycloak/JWT)

### 3. Copy Skills

```bash
cp -r /path/to/showcase/.claude/skills/backend-dev-guidelines .claude/skills/
cp -r /path/to/showcase/.claude/skills/frontend-dev-guidelines .claude/skills/
```

### 4. Copy Hooks

```bash
cp -r /path/to/showcase/.claude/hooks/* .claude/hooks/
cd .claude/hooks && npm install
chmod +x *.sh
```

### 5. Copy Git Hooks

```bash
cp /path/to/cards-against-humanity/.claude/git-hooks/* .claude/git-hooks/
chmod +x .claude/git-hooks/*.sh .claude/git-hooks/{pre-commit,pre-push,commit-msg}
```

### 6. Create settings.json

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
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|MultiEdit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/post-tool-use-tracker.sh"
          }
        ]
      }
    ]
  }
}
```

### 7. Create skill-rules.json

Adapt for your project structure:

**Next.js Example:**
```json
{
  "version": "1.0.0",
  "skills": {
    "backend-dev-guidelines": {
      "type": "domain",
      "enforcement": "suggest",
      "priority": "high",
      "promptTriggers": {
        "keywords": ["api", "route", "endpoint", "server", "prisma", "database"],
        "intentPatterns": ["create.*api", "add.*endpoint", "implement.*route"]
      },
      "fileTriggers": {
        "pathPatterns": ["src/app/api/**/*.ts", "src/lib/**/*.ts"],
        "contentPatterns": ["NextRequest", "NextResponse", "prisma"]
      }
    },
    "frontend-dev-guidelines": {
      "type": "domain",
      "enforcement": "suggest",
      "priority": "high",
      "promptTriggers": {
        "keywords": ["component", "react", "ui", "page"],
        "intentPatterns": ["create.*component", "add.*page"]
      },
      "fileTriggers": {
        "pathPatterns": ["src/app/**/*.tsx", "src/components/**/*.tsx"],
        "contentPatterns": ["'use client'", "useState", "useEffect"]
      }
    }
  }
}
```

**Express + React Monorepo Example:**
```json
{
  "version": "1.0.0",
  "skills": {
    "backend-dev-guidelines": {
      "type": "domain",
      "enforcement": "suggest",
      "priority": "high",
      "fileTriggers": {
        "pathPatterns": ["server/src/**/*.ts", "packages/backend/**/*.ts"],
        "contentPatterns": ["import.*express", "Router()", "app.use"]
      }
    },
    "frontend-dev-guidelines": {
      "type": "domain",
      "enforcement": "suggest",
      "priority": "high",
      "fileTriggers": {
        "pathPatterns": ["client/src/**/*.tsx", "packages/frontend/**/*.tsx"],
        "contentPatterns": ["import.*react", "useState", "useEffect"]
      }
    }
  }
}
```

### 8. Install Git Hooks

```bash
./.claude/git-hooks/install.sh
```

### 9. Create SYNC_MANIFEST.json

Track your integration:

```json
{
  "version": "1.0.0",
  "showcaseSource": {
    "path": "/path/to/claude-code-infrastructure-showcase",
    "version": "1.0.0",
    "lastSync": "2025-11-01"
  },
  "project": {
    "name": "your-project",
    "description": "Brief description of your project",
    "integrationDate": "2025-11-01",
    "customizationLevel": "light"
  },
  "syncedFiles": {
    "agents": {
      ".claude/agents/auto-error-resolver.md": {
        "status": "synced",
        "sourceFile": ".claude/agents/auto-error-resolver.md",
        "syncStrategy": "overwrite-safe",
        "lastSync": "2025-11-01"
      }
      // ... list all copied files
    }
  }
}
```

### 10. Register in Showcase (Optional)

Update `INTEGRATION_REGISTRY.json` in the showcase:

```json
{
  "your-project": {
    "path": "/path/to/your-project",
    "integrationDate": "2025-11-01",
    "customizationLevel": "light",
    "description": "Your project description",
    "components": {
      // ... component details
    }
  }
}
```

## Heavy Integration Steps

For specialized stacks requiring custom skills:

### 1. Complete Light Integration
Follow steps 1-10 above as foundation.

### 2. Analyze Your Stack
Document unique patterns:
- Framework-specific patterns (Socket.IO, GraphQL, tRPC, etc.)
- Database patterns (Redis, MongoDB, Prisma, etc.)
- Authentication approach
- Real-time features
- Testing setup

### 3. Create Custom Skills
Use cards-against-humanity as template:

```bash
mkdir -p .claude/skills/your-custom-skill/resources
```

Create `SKILL.md` with:
- Tech stack overview
- Architecture patterns
- Quick reference code snippets
- Key principles
- Anti-patterns to avoid
- Links to resource files

Create resource files for:
- testing.md - Testing patterns
- debugging.md - Debugging guides
- [technology].md - Technology-specific patterns

### 4. Create Custom Commands
Create `.claude/commands/` for project-specific workflows:
- test-[feature].md
- debug-[feature].md
- review-[pattern].md

### 5. Update skill-rules.json
Add triggers for custom skills with specific:
- Keywords from your domain
- File path patterns
- Content patterns

## Verification

After integration, verify everything works:

```bash
# 1. Check hooks are executable
ls -l .claude/hooks/*.sh .claude/git-hooks/*

# 2. Test git hooks
git commit -m "test: verify commit-msg hook"  # Should validate message
git status  # Should see pre-commit would run

# 3. Test skill activation
# Start Claude Code and mention keywords from your skill-rules.json
# Skills should auto-activate

# 4. Test agents
# Try commands like "review this code" or "help me debug"
# Agents should be available
```

## Customization Tips

### Adjusting Git Hooks

Edit `.claude/git-hooks/pre-commit` to customize checks:

```bash
# Add custom check
run_check "Custom Validation" "your-custom-command"

# Skip check for your project
# Comment out unwanted checks
```

### Adapting Skills

Generic skills work for most projects, but you can:
1. Keep generic skills as-is (recommended for most)
2. Fork and customize for your stack
3. Create entirely custom skills (like cards-against-humanity)

### Selecting Agents

Not all agents may be relevant:
- Remove auth-specific agents if not using JWT/Keycloak
- Keep general-purpose agents (error-fixer, architecture-reviewer)
- Add custom agents for specialized needs

## Troubleshooting

### Hooks Not Running

```bash
# Reinstall
./.claude/git-hooks/install.sh

# Check symlinks
ls -la .git/hooks/{pre-commit,pre-push,commit-msg}
```

### Skills Not Activating

1. Check skill-rules.json syntax
2. Verify file path patterns match your structure
3. Test with explicit keywords from promptTriggers

### Git Hooks Failing

```bash
# Run checks manually
cd server && pnpm exec tsc --noEmit  # TypeScript
cd client && pnpm exec eslint 'src/**/*.ts'  # ESLint
```

## Maintenance

### Syncing Updates from Showcase

```bash
# Check for updates
diff -r /path/to/showcase/.claude/agents .claude/agents

# Copy updated files
cp /path/to/showcase/.claude/agents/[updated-file].md .claude/agents/

# Update SYNC_MANIFEST.json with new lastSync dates
```

### Using /sync-infrastructure Command

If you copied sync commands from cards-against-humanity:

```bash
# In Claude Code
/sync-infrastructure
# Follow prompts to sync selectively
```

## Examples

### Orbit (Light Integration)
- Next.js 15 + Prisma + Socket.IO
- Generic skills
- ~15 minutes integration time

### Cards Against Humanity (Heavy Integration)
- Express + Socket.IO + Redis + React
- Custom real-time skills
- Custom slash commands
- ~90 minutes integration time

## Support

- **Showcase README:** See main README.md for component details
- **Integration Registry:** Check INTEGRATION_REGISTRY.json for examples
- **Git Hooks Docs:** `.claude/git-hooks/README.md` in integrated projects

## Next Steps

After integration:
1. Test with actual development workflow
2. Customize skill-rules.json for your patterns
3. Add project-specific commands if needed
4. Share feedback to improve showcase

Happy coding with Claude! 🚀
