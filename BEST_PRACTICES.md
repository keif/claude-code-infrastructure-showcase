# Best Practices for Claude Code Infrastructure

Lessons learned from 6 months of production use and deploying to multiple projects.

## Table of Contents
- [Integration Strategy](#integration-strategy)
- [Skill Development](#skill-development)
- [Hook Configuration](#hook-configuration)
- [Agent Usage](#agent-usage)
- [Git Hooks](#git-hooks)
- [Maintenance](#maintenance)
- [Common Pitfalls](#common-pitfalls)

---

## Integration Strategy

### Start Small, Scale Up

**❌ Bad:** Copy everything from showcase on day one
**✅ Good:** Incremental integration

```
Week 1: Essential hooks + 1 skill
Week 2: Add relevant agents
Week 3: Add git hooks
Week 4: Customize and expand
```

**Why:** Allows team to adapt gradually, reduces confusion.

### Choose the Right Integration Level

**Light Integration (15 min):**
- Standard tech stacks (Next.js, standard Express, React)
- Small teams
- Quick wins needed
- Generic patterns sufficient

**Heavy Integration (90 min):**
- Specialized stacks (real-time, GraphQL, unique architectures)
- Large teams
- Long-term investment
- Domain-specific patterns needed

**Example:**
- orbit: Next.js → Light integration → 15 minutes
- cards-against-humanity: Socket.IO → Heavy integration → 90 minutes

### Test Before Committing

Always test in a feature branch first:

```bash
git checkout -b test/claude-infrastructure
# Integrate infrastructure
# Test with real development work
# Verify hooks work correctly
git checkout main
# Apply learnings to main branch
```

---

## Skill Development

### Follow the 500-Line Rule

**Why it exists:**
- Claude's context window is finite
- Large skills get truncated
- Details at the end get lost

**The Pattern:**
```
skill-name/
  SKILL.md                    # < 500 lines (overview + navigation)
  resources/
    specific-topic-1.md       # < 500 lines
    specific-topic-2.md       # < 500 lines
```

**SKILL.md should contain:**
- High-level overview (50-100 lines)
- Quick reference examples (100-200 lines)
- Key principles (50-100 lines)
- Links to resource files (remaining)

**Resource files should contain:**
- Deep dive into one specific topic
- Comprehensive examples
- Edge cases and troubleshooting

### Write for Your Actual Codebase

**❌ Bad:** Generic examples that don't match your code

```typescript
// Generic example
class UserService {
  getUser(id: number) { }
}
```

**✅ Good:** Examples from your actual patterns

```typescript
// From your codebase
export class UserService {
  constructor(private prisma: PrismaClient) {}

  async getById(userId: string): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: { id: userId },
      include: { profile: true }
    });
  }
}
```

### Include Anti-Patterns

Showing what NOT to do is as valuable as showing best practices:

```typescript
// ❌ BAD: Missing error handling
socket.on('message', async (data) => {
  await processMessage(data);
});

// ✅ GOOD: Proper error handling
socket.on('message', async (data) => {
  try {
    await processMessage(data);
  } catch (error) {
    logger.error('Message processing failed', { error, data });
    socket.emit('error', { message: 'Processing failed' });
  }
});
```

### Version Control Your Skills

Skills evolve. Track changes:

```markdown
<!-- At top of SKILL.md -->
## Changelog
- 2025-11-01: Added testing patterns
- 2025-10-15: Updated for Prisma 6
- 2025-10-01: Initial version
```

---

## Hook Configuration

### Essential vs. Optional

**Always include:**
- `skill-activation-prompt` - Auto-activation magic
- `post-tool-use-tracker` - File modification tracking

**Conditional:**
- `tsc-check` - Only for TypeScript monorepos
- `error-handling-reminder` - Only if your team needs reminders
- Custom build hooks - Only for complex build systems

### skill-rules.json Precision

**Be specific with file patterns:**

```json
{
  "fileTriggers": {
    "pathPatterns": [
      "src/app/api/**/*.ts",           // ✅ Specific
      "NOT src/**/*.ts"                 // ❌ Too broad
    ]
  }
}
```

**Use intent patterns wisely:**

```json
{
  "intentPatterns": [
    "create.*component",                // ✅ Clear intent
    "NOT build|create|make|add"        // ❌ Too generic
  ]
}
```

### Test Hook Activation

After setup, verify:

```bash
# 1. Edit a file that should trigger a skill
# 2. Check if skill is suggested
# 3. Try a prompt with trigger keywords
# 4. Verify skill activates
```

---

## Agent Usage

### When to Use Agents vs. Direct Prompts

**Use agents for:**
- Multi-step tasks (refactoring, architecture review)
- Tasks requiring context gathering
- Specialized analysis (security, performance)
- Tasks you do repeatedly

**Use direct prompts for:**
- Simple questions
- One-off tasks
- Quick fixes
- Exploratory work

### Proactive Agent Use

Don't wait for problems:

```
After writing feature:
→ Use code-architecture-reviewer

Before big refactor:
→ Use refactor-planner

When stuck on error:
→ Use web-research-specialist

Before merging PR:
→ Use plan-reviewer
```

### Agent Customization

Agents work out-of-the-box, but you can enhance them:

1. **Add project context:**
   ```markdown
   <!-- In agent file -->
   ## Project-Specific Patterns
   - We use Prisma for database access
   - All services follow dependency injection
   ```

2. **Customize output format:**
   ```markdown
   ## Output Format
   Use our standard review template:
   - Security issues
   - Performance concerns
   - Best practice violations
   ```

---

## Git Hooks

### Balance Speed vs. Coverage

**pre-commit (Fast):**
- Type checking (1-3 seconds)
- Linting (1-3 seconds)
- Quick syntax checks

**pre-push (Thorough):**
- Full test suite (10-30 seconds)
- Builds (10-30 seconds)
- Integration tests (optional)

**Don't put slow checks in pre-commit** - developers will use `--no-verify`.

### Commit Message Validation

**Recommended format:**
```
type(scope): description

feat(api): add user authentication
fix(ui): resolve button alignment issue
docs(readme): update installation steps
```

**But don't be too strict:**
- Allow reasonable messages that don't follow format
- Focus on quality, not format compliance
- Warn instead of block for minor issues

### Monorepo Considerations

For monorepos, scope hooks to changed packages:

```bash
# Only test changed packages
CHANGED_DIRS=$(git diff --name-only --cached | xargs -n1 dirname | sort -u)

for dir in $CHANGED_DIRS; do
  if [ -d "$dir" ] && [ -f "$dir/package.json" ]; then
    cd "$dir" && npm test
  fi
done
```

---

## Maintenance

### Sync Updates from Showcase

**Use SYNC_MANIFEST.json:**
- Track which files are synced vs. custom
- Use sync strategies (overwrite-safe, manual-review, never-overwrite)
- Update regularly but carefully

**Safe to update:**
- General-purpose agents
- Claude hooks (if not customized)
- Generic skills

**Review before updating:**
- Project-specific configurations
- Custom skills
- Git hooks adapted to your workflow

### Keep Documentation Current

**Update after:**
- Major refactoring
- Tech stack changes
- New patterns emerge
- Team feedback

**Schedule:**
- Review skills quarterly
- Update examples when they become outdated
- Prune deprecated patterns

### Monitor Effectiveness

**Track:**
- How often skills activate
- Which agents get used
- Git hook pass/fail rates
- Team feedback

**Adjust based on data:**
- Low activation → Update trigger patterns
- Agent unused → Remove or improve
- Hooks failing often → Review criteria

---

## Common Pitfalls

### 1. Over-Engineering Skills

**Pitfall:** 1000+ line skills trying to cover everything

**Solution:** Start minimal, add incrementally

```
v1: Core patterns (200 lines)
v2: Add edge cases (300 lines)
v3: Add resources as needed (500 lines + resources)
```

### 2. Copy-Paste Without Adaptation

**Pitfall:** Using showcase examples verbatim

**Solution:** Always adapt to your codebase

```markdown
<!-- In skill -->
## Your Project Patterns

Based on showcase patterns but adapted for our use case:
- We use Prisma not Sequelize
- We use Next.js API routes not Express
- Our auth is JWT not sessions
```

### 3. Ignoring skill-rules.json

**Pitfall:** Skills copied but not configured for auto-activation

**Solution:** Always create/update skill-rules.json

```json
{
  "your-custom-skill": {
    "promptTriggers": { "keywords": ["your", "domain", "terms"] },
    "fileTriggers": { "pathPatterns": ["your/code/paths/**/*.ts"] }
  }
}
```

### 4. Git Hooks Too Strict

**Pitfall:** Hooks block commits too often

**Solution:** Warn, don't block

```bash
# ❌ BAD: Block on minor issues
if [ $WARNINGS -gt 0 ]; then
  exit 1
fi

# ✅ GOOD: Warn but allow
if [ $WARNINGS -gt 0 ]; then
  echo "⚠️  Found $WARNINGS warnings"
  echo "Fix when convenient, or use --no-verify"
fi
```

### 5. Not Using SYNC_MANIFEST

**Pitfall:** Losing track of what's custom vs. synced

**Solution:** Maintain SYNC_MANIFEST.json from day one

```json
{
  "syncedFiles": {
    ".claude/agents/auto-error-resolver.md": {
      "status": "synced",
      "syncStrategy": "overwrite-safe"
    },
    ".claude/skills/my-custom-skill/": {
      "status": "custom",
      "syncStrategy": "never-overwrite"
    }
  }
}
```

### 6. Forgetting About Team Onboarding

**Pitfall:** Infrastructure works for you, confusing for teammates

**Solution:** Document locally

```
.claude/
  README.md          # Project-specific guide
  ONBOARDING.md      # How to use Claude Code here
  CONVENTIONS.md     # Our patterns and standards
```

### 7. Skills That Don't Reflect Reality

**Pitfall:** Skills describe ideal code, not actual code

**Solution:** Sync skills with refactoring

```
When refactoring:
1. Update the code
2. Update the skill
3. Keep them in sync
```

---

## Quick Wins

### Day 1
- Install essential hooks
- Add one relevant skill
- Configure skill-rules.json
- Verify auto-activation

### Week 1
- Add git hooks
- Add 2-3 relevant agents
- Create first custom command (if needed)

### Month 1
- Customize skills with project patterns
- Create custom resources
- Gather team feedback
- Iterate on trigger patterns

### Quarter 1
- Full heavy integration (if needed)
- Custom skills for unique patterns
- Complete git hook coverage
- Team training complete

---

## Metrics for Success

**Good indicators:**
- Skills activate without manual triggering
- Git hooks catch issues before push
- Team references skills in discussions
- Agents save time on complex tasks
- Code quality improves

**Warning signs:**
- Skills never auto-activate (fix triggers)
- Git hooks bypassed frequently (too strict)
- Skills ignored (not relevant or too generic)
- Agents not used (not discoverable)

---

## Resources

- [INTEGRATION_GUIDE.md](INTEGRATION_GUIDE.md) - Step-by-step integration
- [INTEGRATION_REGISTRY.json](INTEGRATION_REGISTRY.json) - Real integration examples
- [Showcase README](README.md) - Component catalog

---

Remember: This infrastructure should **enhance** your workflow, not constrain it. If something doesn't work for your team, adapt or remove it. The goal is productivity, not perfection.
