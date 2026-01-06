# Claude Skills

A collection of slash commands for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that enhance your development workflow with structured processes for PRDs, task management, PR workflows, and autonomous coding.

## Installation

Copy the `commands/` directory to your project's `.claude/commands/` folder:

```bash
# Clone the repository
git clone https://github.com/daniel-scrivner/claude-skills.git

# Copy commands to your project
cp -r claude-skills/commands/ your-project/.claude/commands/
```

Or install globally for all projects:

```bash
cp -r claude-skills/commands/ ~/.claude/commands/
```

## Available Commands

| Command | Description |
|---------|-------------|
| `/generate-prd` | Create detailed Product Requirements Documents through structured conversation |
| `/generate-tasks` | Generate step-by-step task lists from requirements or PRDs |
| `/pr-workflow` | Non-negotiable PR workflow with autonomous monitoring and merge verification |
| `/ralph` | Autonomous coding loop that works through tasks one at a time |

---

## `/generate-prd`

**Generate a Product Requirements Document through structured conversation.**

### Usage

```
/generate-prd Add a dark mode toggle to the settings page
```

### Process

1. **Analyze Request** - Identifies what's clear vs. unclear
2. **Ask Questions** - 3-5 numbered questions with lettered options (respond like "1A, 2C, 3B")
3. **Generate PRD** - Creates comprehensive document with goals, user stories, requirements
4. **Iterate** - Refine based on feedback
5. **Save** - Writes to `./tasks/prd-[feature-name].md`

### PRD Sections

- Overview & Problem Statement
- Goals with Success Metrics
- User Stories (Primary & Secondary)
- Functional Requirements (P0/P1/P2)
- Non-Goals (Out of Scope)
- Design & Technical Considerations
- Open Questions

---

## `/generate-tasks`

**Generate a detailed task list from user requirements.**

### Usage

```
/generate-tasks User profile editing feature
```

Or reference an existing PRD:

```
/generate-tasks @tasks/prd-user-profile.md
```

### Process

1. **Phase 1** - Generate high-level parent tasks
2. **Wait for "Go"** - User confirms before sub-task generation
3. **Phase 2** - Break down into atomic sub-tasks with file references

### Output Format

```markdown
# [Feature] Implementation Tasks

## Relevant Files
- `path/to/component.tsx` - Description

## Tasks
- [ ] 0.0 Create feature branch
  - [ ] 0.1 Create and checkout branch
- [ ] 1.0 [First Major Task]
  - [ ] 1.1 [Sub-task]
- [ ] N.0 Create PR and merge
  - [ ] N.5 Create PR following /pr-workflow
```

### Task Sizing

| Level | Duration | Scope |
|-------|----------|-------|
| Parent (X.0) | 30-60 min | Logical milestone |
| Sub-task (X.Y) | 5-15 min | Atomic action |

---

## `/pr-workflow`

**Non-negotiable PR workflow that MUST be followed for every pull request.**

### Phase 1: Before Creating PR

1. **Run Validations** - typecheck, build, test based on project type
2. **Security Scan** - No hardcoded paths, secrets, or machine-specific values
3. **Create PR** - Use repository's PR template

### Phase 2: Autonomous Monitoring

4. **Monitor Bot Reviews** - Poll status until all checks complete
5. **Address ALL Feedback** - Implement fixes, push, reply to comments
6. **Wait for Re-validation** - All checks must show SUCCESS

### Phase 3: Merge Criteria

7. **Final Verification** - `mergeable_state` must be `"clean"`
8. **Execute Merge** - Only when all conditions are met
9. **Verify Success** - Confirm merge completed

### Key Insight

```
mergeable: true       → No git conflicts
mergeable_state: "clean" → All checks passed, safe to merge

THESE ARE DIFFERENT. Only merge when mergeable_state is "clean".
```

---

## `/ralph`

**Autonomous coding loop using the Ralph Wiggum technique.**

Named after [Ralph Wiggum](https://www.youtube.com/watch?v=CxH0zVDr3mQ) - simple but effective.

### Quick Start

```bash
# 1. Scaffold the Ralph directory structure
/ralph scaffold

# 2. Edit the PRD with your tasks
# Edit scripts/ralph/prd.json

# 3. Run autonomously
./scripts/ralph/ralph.sh 25
```

### How It Works

```
┌─────────────────────────────────────────────────────────┐
│                    Ralph Loop                           │
├─────────────────────────────────────────────────────────┤
│  1. Read prd.json and progress.txt                      │
│  2. Pick ONE task (highest priority, passes: false)     │
│  3. Implement the task                                  │
│  4. Run typecheck + tests                               │
│  5. Commit changes                                      │
│  6. Mark task complete in prd.json                      │
│  7. Log learnings to progress.txt                       │
│  8. If tasks remain → Loop                              │
│  9. If all done → Verify PR readiness → COMPLETE        │
└─────────────────────────────────────────────────────────┘
```

### Directory Structure

```
scripts/ralph/
├── ralph.sh      # The loop runner
├── prompt.md     # Instructions for Claude
├── prd.json      # Task list with pass/fail status
└── progress.txt  # Learnings and patterns discovered
```

### prd.json Format

```json
{
  "branchName": "ralph/feature-name",
  "description": "What this feature accomplishes",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add login form",
      "acceptanceCriteria": [
        "Email and password fields exist",
        "typecheck passes",
        "tests pass"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

### Task Sizing

**Too Big:**
- "Build entire auth system"
- "Implement user dashboard"

**Right Size:**
- "Add login form UI"
- "Add email validation"
- "Add auth API hook"

### Two-Phase Completion

1. **Task Completion** - All stories have `passes: true`
2. **PR Readiness** - All comments addressed, CI green, no conflicts

Only outputs `<promise>COMPLETE</promise>` when BOTH phases pass.

---

## Command Compatibility

These commands use Claude Code's slash command features:

| Feature | Usage |
|---------|-------|
| `$ARGUMENTS` | Captures all user input after command |
| `allowed-tools` | Restricts available tools for safety |
| `argument-hint` | Shows placeholder in command palette |

### Frontmatter Format

```yaml
---
description: Brief description shown in command palette
allowed-tools: Read, Write, Bash(git:*), mcp__github__*
argument-hint: [feature-description]
---
```

---

## Requirements

- [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) installed
- For `/ralph`: `claude --dangerously-skip-permissions` capability
- For `/pr-workflow`: GitHub MCP server configured

---

## License

MIT
