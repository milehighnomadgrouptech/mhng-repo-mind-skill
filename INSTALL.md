# Installing the mhng-repo-mind Claude skill

This skill drops into Claude Code's skills directory so Claude can use mhng-repo-mind methodology in any session. It works **with or without** the `mhng-repo-mind` CLI installed — the CLI is faster, the skill alone still works.

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed and signed in
- Git (for the recommended install method) OR a way to download zip files
- Optional: the [`mhng-repo-mind` CLI](https://github.com/milehighnomadgrouptech/mhng-repo-mind) for the full experience

## macOS / Linux

```bash
# Make sure the skills directory exists
mkdir -p ~/.claude/skills

# Clone the skill into it
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill.git ~/.claude/skills/mhng-repo-mind

# Verify it landed
ls ~/.claude/skills/mhng-repo-mind
# You should see SKILL.md, prompts/, schemas/, analyses/, reference/
```

Restart Claude Code or run `/skills` to refresh. The skill is now active.

## Windows (PowerShell)

```powershell
mkdir -Force "$HOME\.claude\skills"
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill.git "$HOME\.claude\skills\mhng-repo-mind"
ls "$HOME\.claude\skills\mhng-repo-mind"
```

If you prefer Git Bash or WSL, use the macOS/Linux instructions above with `~` instead of `$HOME`.

## Without git (zip install)

1. Download the latest release zip from https://github.com/milehighnomadgrouptech/mhng-repo-mind-skill/releases
2. Unzip it
3. Move the unzipped folder to:
   - **macOS/Linux:** `~/.claude/skills/mhng-repo-mind/`
   - **Windows:** `%USERPROFILE%\.claude\skills\mhng-repo-mind\`
4. Verify `SKILL.md` is at the root of that folder, not nested inside another `mhng-repo-mind-skill-vX.Y.Z/` folder. If it's nested, move the contents up one level.
5. Restart Claude Code.

## Verifying it works

In Claude Code, type:

> "What skills do I have installed?"

You should see `mhng-repo-mind` in the list. If it's not there:

- Make sure `SKILL.md` is at the root of `~/.claude/skills/mhng-repo-mind/` (not nested)
- Make sure `SKILL.md` starts with the YAML frontmatter (`---` block at the top)
- Restart Claude Code completely (not just close the window)
- Run `/skills` if your version supports it

## Optional: install the CLI

The skill alone is enough to use mhng-repo-mind methodology. But the CLI is **faster**, **incremental**, and ships with **deterministic commands** the skill can't replicate (compliance reports, manifest rebuilding, CI hooks).

```bash
git clone https://github.com/milehighnomadgrouptech/mhng-repo-mind.git
cd mhng-repo-mind
npm install
npm link    # exposes `mhng-repo-mind` globally

# verify
mhng-repo-mind --version
```

Once both the skill and CLI are installed, Claude will automatically prefer the CLI for heavy work and fall back to the prompts when the CLI isn't available.

## Updating

```bash
cd ~/.claude/skills/mhng-repo-mind
git pull
```

Or download the newer release zip and replace the folder contents.

## Uninstalling

```bash
rm -rf ~/.claude/skills/mhng-repo-mind
```

Or on Windows:

```powershell
Remove-Item -Recurse -Force "$HOME\.claude\skills\mhng-repo-mind"
```

## Troubleshooting

**"Claude doesn't see the skill"**
Check that `SKILL.md` is at the root of `~/.claude/skills/mhng-repo-mind/`. If it's at `~/.claude/skills/mhng-repo-mind/mhng-repo-mind-skill/SKILL.md`, the folder is nested too deeply — move everything up one level.

**"The skill loads but the prompts don't work"**
Make sure all the files in `prompts/`, `schemas/`, and `analyses/` are present. The skill is self-contained — missing files break it. Re-clone or re-download to fix.

**"I want to use a different LLM, not Claude"**
The skill is Claude-Code-specific by design. The prompts in `prompts/` are plain markdown and portable to any LLM, but the skill loader format is Claude Code's. To use the prompts in another tool, copy them out manually.

**"Compliance reports are inaccurate"**
The skill produces structured AI-generated claims, not legal certifications. Every compliance tag should be reviewed by a human compliance officer. See the disclaimer in [`reference/security-deployment.md`](reference/security-deployment.md).
