# SKILLS

Agent skills for Claude Code — drop these into `~/.claude/skills/` to use them.

## Skills

### [`code-quality`](./code-quality/SKILL.md)
Clean, maintainable code standards. Applies naming conventions, TypeScript strictness, error handling rules, and Biome linting guidance to any code written or reviewed.

### [`ship`](./ship/SKILL.md)
Full feature development lifecycle — PRD drafting, critical review, task breakdown, parallel implementation with subagents, and automated quality gates (type check → tests → code review → lint). Takes a feature description all the way to reviewed, tested code.

## Usage

```bash
# Install all skills
cp -r SKILLS/* ~/.claude/skills/

# Or install a single skill
cp -r SKILLS/code-quality ~/.claude/skills/
```

Then invoke in Claude Code:
```
/code-quality
/ship <feature description>
```
