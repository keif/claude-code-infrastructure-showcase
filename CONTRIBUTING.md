# Contributing to Claude Code Infrastructure Showcase

Thank you for your interest in improving this showcase! Contributions from the community help make this resource better for everyone.

## Table of Contents
- [Ways to Contribute](#ways-to-contribute)
- [Contribution Guidelines](#contribution-guidelines)
- [Skill Contributions](#skill-contributions)
- [Agent Contributions](#agent-contributions)
- [Hook Contributions](#hook-contributions)
- [Documentation](#documentation)
- [Integration Examples](#integration-examples)
- [Code of Conduct](#code-of-conduct)

---

## Ways to Contribute

### 1. Share Your Skills
Have a skill that works well for your domain? Share it!

**Good candidates:**
- Domain-specific patterns (e-commerce, SaaS, gaming, etc.)
- Framework-specific skills (Next.js, NestJS, FastAPI, etc.)
- Tech stack combinations (Prisma + tRPC, GraphQL + Apollo, etc.)

### 2. Share Your Agents
Created a useful agent? Contribute it!

**Good candidates:**
- Specialized debugging agents
- Code analysis agents
- Documentation generators
- Testing assistants

### 3. Share Integration Stories
Successfully integrated into your project? Share your experience!

**What to include:**
- Tech stack
- Integration approach (light/heavy)
- Challenges faced
- Solutions found
- Time investment

### 4. Improve Documentation
- Fix typos or unclear instructions
- Add examples
- Improve diagrams
- Translate to other languages

### 5. Report Issues
- Skills not activating properly
- Hook errors
- Documentation gaps
- Integration problems

---

## Contribution Guidelines

### Before Contributing

1. **Check existing issues/PRs** - Someone may be working on it
2. **Open an issue first** - Discuss your idea before implementing
3. **Follow existing patterns** - Match the style of the showcase
4. **Test thoroughly** - Ensure it works in a real project

### Quality Standards

**All contributions should:**
- ✅ Be production-tested (not theoretical)
- ✅ Include clear documentation
- ✅ Follow the modular pattern (skills)
- ✅ Include examples
- ✅ Be domain-agnostic or clearly labeled

**Avoid:**
- ❌ Company-specific patterns
- ❌ Proprietary information
- ❌ Untested code
- ❌ Skills over 500 lines without resources

---

## Skill Contributions

### Skill Structure

```
.claude/skills/your-skill-name/
├── SKILL.md                    # Main file (<500 lines)
├── resources/                  # Optional but recommended
│   ├── topic-1.md
│   └── topic-2.md
└── skill-rules.json (example)  # Optional: example triggers
```

### Skill Template

```markdown
---
name: your-skill-name
description: Brief description (one sentence)
---

# Your Skill Name

Brief introduction (2-3 sentences)

---

## Purpose

What this skill helps with

---

## When to Use This Skill

Scenarios when this skill activates

---

## Quick Reference

Common patterns with code examples

---

## Resource Files

Links to detailed resources

---

## Key Principles

Best practices

---

## Anti-Patterns

What to avoid
```

### Skill Checklist

Before submitting a skill:

- [ ] Main SKILL.md is under 500 lines
- [ ] Includes code examples from real projects
- [ ] Shows both good and bad patterns
- [ ] Links to resource files if >500 lines
- [ ] Includes suggested trigger patterns
- [ ] Tested in actual development
- [ ] Documentation is clear
- [ ] No company-specific details
- [ ] No sensitive information

### Submitting a Skill

1. Fork the repository
2. Create skill directory in `.claude/skills/`
3. Write SKILL.md and resources
4. Add example `skill-rules.json` entry
5. Test in a real project
6. Create pull request with:
   - Skill description
   - Tech stack it supports
   - Testing performed
   - Screenshots (optional)

**PR Template:**
```markdown
## Skill: [name]

**Tech Stack:** [frameworks, languages, tools]

**Purpose:** [what it does]

**Testing:**
- Tested in [project type]
- Activation works for [scenarios]
- Examples verified

**Additional Notes:** [anything else]
```

---

## Agent Contributions

### Agent Structure

```
.claude/agents/your-agent-name.md
```

Single file, focused on one task.

### Agent Template

```markdown
---
name: your-agent-name
description: What this agent does
---

# Your Agent Name

## Purpose
What problem this solves

## When to Use
Specific scenarios

## How It Works
1. Step 1
2. Step 2
3. Step 3

## Examples
Concrete usage examples

## Output Format
What the user receives
```

### Agent Checklist

- [ ] Focuses on one specific task
- [ ] Includes clear examples
- [ ] Describes expected output
- [ ] Tested with complex scenarios
- [ ] Self-contained (doesn't require other agents)
- [ ] Documentation explains when to use

### Submitting an Agent

1. Fork the repository
2. Create agent file in `.claude/agents/`
3. Test in various scenarios
4. Create pull request with:
   - Agent description
   - Use cases
   - Testing performed

---

## Hook Contributions

Hooks are more sensitive as they execute code. Higher bar for contributions.

### Requirements for Hook Contributions

**Must have:**
- Detailed documentation
- Security considerations explained
- Clear customization instructions
- Tested across different project types
- Minimal dependencies

**Security review:**
- No external network calls (unless justified)
- No file modifications outside project
- Clear permission requirements
- Documented risk profile

### Hook Checklist

- [ ] Well-documented
- [ ] Security reviewed
- [ ] Minimal dependencies
- [ ] Fast execution (<100ms for UserPromptSubmit)
- [ ] Works cross-platform (macOS/Linux)
- [ ] Error handling included
- [ ] Customization guide provided

### Submitting a Hook

1. Open issue FIRST - discuss the hook idea
2. If approved, fork repository
3. Create hook in `.claude/hooks/`
4. Add comprehensive README
5. Test thoroughly
6. Security review by maintainer
7. Create pull request

---

## Documentation

### Documentation Improvements

**Welcome contributions:**
- Typo fixes
- Clarifications
- Additional examples
- Better explanations
- New diagrams

**Process:**
1. Fork repository
2. Make changes
3. Ensure markdown renders correctly
4. Create pull request

### Documentation Standards

- Use clear, concise language
- Include code examples
- Explain the "why" not just the "what"
- Link to related docs
- Keep formatting consistent

---

## Integration Examples

Sharing your integration story helps others!

### What to Include

```json
{
  "project-name": {
    "techStack": "Next.js 15 + Prisma + Redis",
    "integrationLevel": "light | heavy",
    "timeInvestment": "15 minutes",
    "components": {
      "skills": ["list of skills used"],
      "agents": ["list of agents used"],
      "customizations": "what you customized"
    },
    "challenges": ["challenge 1", "challenge 2"],
    "solutions": ["solution 1", "solution 2"],
    "results": "what you achieved"
  }
}
```

### Submitting Integration Example

1. Add entry to `INTEGRATION_REGISTRY.json`
2. Optionally create case study in `case-studies/`
3. Create pull request

**Note:** Only include if you're comfortable sharing publicly. Can anonymize project name.

---

## Code of Conduct

### Our Standards

**Positive environment:**
- Be respectful and inclusive
- Welcome newcomers
- Provide constructive feedback
- Assume good intentions

**Unacceptable:**
- Harassment or discrimination
- Trolling or insulting comments
- Publishing private information
- Spamming or self-promotion

### Enforcement

Violations reported to maintainers will be reviewed. Consequences range from warnings to permanent bans.

---

## Pull Request Process

### Before Submitting

1. **Test your changes** in a real project
2. **Update documentation** if needed
3. **Follow existing patterns**
4. **Write clear commit messages**

### PR Description Template

```markdown
## Type of Change
- [ ] New skill
- [ ] New agent
- [ ] New hook
- [ ] Documentation
- [ ] Bug fix
- [ ] Integration example

## Description
[Clear description of changes]

## Testing
[How you tested this]

## Screenshots (if applicable)
[Add screenshots]

## Checklist
- [ ] Tested in real project
- [ ] Documentation updated
- [ ] Follows existing patterns
- [ ] No sensitive information
```

### Review Process

1. **Automated checks** run on PR
2. **Maintainer reviews** for:
   - Quality
   - Security (for hooks)
   - Fit with showcase
3. **Feedback provided** if changes needed
4. **Merge** when approved

---

## Development Setup

### Local Testing

1. Fork and clone:
   ```bash
   git clone https://github.com/your-username/claude-code-infrastructure-showcase
   cd claude-code-infrastructure-showcase
   ```

2. Create test project:
   ```bash
   mkdir test-integration
   cd test-integration
   npm init -y
   ```

3. Copy component to test:
   ```bash
   mkdir -p .claude/skills
   cp -r ../claude-code-infrastructure-showcase/.claude/skills/your-skill .claude/skills/
   ```

4. Test in Claude Code

5. Iterate until working

### Testing Checklist

**For Skills:**
- [ ] Auto-activation works
- [ ] Examples are accurate
- [ ] Resources load correctly
- [ ] Fits under context limits

**For Agents:**
- [ ] Handles complex scenarios
- [ ] Output is useful
- [ ] Error handling works
- [ ] Documentation is clear

**For Hooks:**
- [ ] Executes without errors
- [ ] Performance is acceptable
- [ ] Works cross-platform
- [ ] Security review passed

---

## Recognition

Contributors will be:
- Listed in CONTRIBUTORS.md
- Credited in relevant documentation
- Mentioned in release notes (for significant contributions)

---

## Questions?

- **General questions:** Open a discussion
- **Bug reports:** Open an issue
- **Feature ideas:** Open an issue for discussion first
- **Security concerns:** Email maintainer directly

---

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

---

## Thank You!

Every contribution, no matter how small, helps make this resource better for the community. Thank you for taking the time to contribute! 🙏
