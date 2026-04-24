# skillforge

Personal Claude Code skills, crafted for real-world DevOps and development workflows.

---

## Skills

| Skill | Description |
|-------|-------------|
| [security-review-nextjs-supabase](./security-review-nextjs-supabase/) | Full security review for Next.js apps with Supabase — RLS, auth, middleware, OWASP, secrets |

---

## Installation

### Claude Code (recommended)

```bash
npx skills add [your-username]/skillforge --skill security-review-nextjs-supabase -a claude-code -g
```

### Cursor

```bash
npx skills add [your-username]/skillforge --skill security-review-nextjs-supabase -a cursor -g
```

### Gemini CLI

```bash
npx skills add [your-username]/skillforge --skill security-review-nextjs-supabase -a gemini -g
```

### Manual

```bash
# Clone the repo
git clone https://github.com/[your-username]/skillforge.git

# Copy the skill to Claude Code
cp -r skillforge/security-review-nextjs-supabase ~/.claude/skills/
```

---

## Compatibility

| Agent | Supported |
|-------|-----------|
| Claude Code | ✅ |
| Cursor | ✅ |
| Gemini CLI | ✅ |
| Codex CLI | ✅ |
| ChatGPT | ❌ |
| GitHub Copilot | ❌ |

---

## Usage

Skills trigger automatically when relevant context is detected. You can also invoke them directly:

```
/security-review-nextjs-supabase
```

Or just ask naturally:

```
"do a security review of this app"
"check the Supabase RLS policies"
"are there any OWASP vulnerabilities?"
```