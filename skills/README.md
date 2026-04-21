# OpenHands Cloud Skills

This directory contains reusable skills for OpenHands agents.

## Available Skills

### laminar-quickstart-trace

A skill for setting up Laminar observability tracing in both cloud and self-hosted environments.

**Use when**: Users want to set up Laminar tracing, create quickstart demos, or integrate Laminar observability into their applications.

**Features**:
- Automatic detection of HTTP vs gRPC availability
- Supports both standard Laminar SDK and HTTP-only mode (for self-hosted without gRPC)
- Complete Python and Node.js examples
- Troubleshooting guides and diagnostic tools

**User prompt format**:
```
Set up Laminar tracing with:
- API key: [YOUR_KEY]
- Server: [URL or "cloud"]
- Language: [Python/Node.js]
```

See [laminar-quickstart-trace/SKILL.md](laminar-quickstart-trace/SKILL.md) for details.

## Adding New Skills

Skills should follow this structure:

```
skills/
  your-skill-name/
    SKILL.md                    # Main skill definition with frontmatter
    references/                 # Supporting documentation
      *.md                      # Reference materials
    scripts/                    # Optional scripts
```

### SKILL.md Format

```markdown
---
name: your-skill-name
description: "Brief description for when this skill should be used"
---

# Skill Title

## User Input Required

List what information you need from users.

## Workflow

Numbered steps explaining how the skill works.

## References

- Link to reference docs
```
