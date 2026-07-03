# Infomaniak Agent Skills

**Development skills for AI coding agents at Infomaniak.**

Skills encode the architecture conventions, quality gates, and best practices that Infomaniak engineers follow when building software. These are packaged so AI agents follow them consistently across every Infomaniak project.

> **Prefix convention:** Skills are prefixed with `infomaniak-` when the name would otherwise be too generic, so they remain clearly identifiable when used alongside other skill providers. This prefix is not required for skills with a sufficiently specific name.

---

## Available Skills

| Skill | Platform | Status | Description |
|-------|----------|--------|-------------|
| [infomaniak-swiftui](infomaniak-swiftui/SKILL.md) | iOS (SwiftUI) | Available | House conventions for modular SwiftUI apps built with Tuist: multi-module architecture (App/Core/CoreUI/Features/Resources), DI via InfomaniakDI, Realm/KMP patterns, InfomaniakCoreUI paddings, view naming conventions, sheet/modal presentation, Swift 6 concurrency. |
| [infomaniak-uikit](infomaniak-uikit/SKILL.md) | iOS (UIKit) | Available | House conventions for UIKit iOS apps built with Tuist: flat App/Core/Resources target layout, UI organized by domain under UI/Controller & UI/View, MVVM with Combine, AppRouter coordinator, DI via InfomaniakDI/CoreTargetAssembly, Realm via Transactionable + schema migrations, UIConstants paddings. |

---

## Installation

### OpenCode (primary)

Copy or symlink the skill directory into one of the OpenCode skills directories.

**Global installation:**

```bash
# From the root of this repository
mkdir -p ~/.config/opencode/skills
cp -r infomaniak-swiftui ~/.config/opencode/skills/infomaniak-swiftui
```
**Project-specific installation**:

```bash
# From the root of this repository
mkdir -p /path/to/your/project/.opencode/skills
cp -r infomaniak-swiftui /path/to/your/project/.opencode/skills/infomaniak-swiftui
```

**Important:** The folder name inside the skills directory must match the `name` field in the skill's frontmatter exactly.

OpenCode discovers skills automatically from any of the following paths:
- `~/.config/opencode/skills/<name>/SKILL.md`
- `.opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md`
- `.agents/skills/<name>/SKILL.md`

### GitHub Copilot

For GitHub Copilot, copy the relevant sections from a `SKILL.md` into your repository's `.github/copilot-instructions.md` file, or reference the skill content directly in your agent configuration.

---

## Skill Structure

Every skill in this repository follows the same structure:

```
<name>/
└── SKILL.md          # The complete skill definition
```

### Required format

The `SKILL.md` must include:

1. **Frontmatter block** at the very top:
   ```yaml
   ---
   name: <name>
   description: Concise description of what this skill covers and when to use it.
   ---
   ```
2. **The `name` field must identically match the directory name.**
3. **Markdown body** with workflows, conventions, checklists, and installation instructions.

---

## Contributing

To add a new skill:

1. Create a directory `<name>/` at the repository root.
2. Add a `SKILL.md` file inside with:
   - Frontmatter containing `name: <name>` and `description`.
   - Architecture conventions, code patterns, and verification checklists specific to the skill's domain.
3. Ensure the `name` in the frontmatter **exactly matches** the directory name.
4. Open a pull request for review.

Skills should be **specific** (actionable steps, not vague advice), **verifiable** (clear exit criteria with evidence requirements), **battle-tested** (based on real workflows), and **minimal** (only what's needed to guide the agent).
