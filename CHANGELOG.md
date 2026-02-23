# Changelog

All notable changes to the GSD Cursor adaptation will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed

- **Hooks**: Corrected path references from `~/.claude/` to `~/.cursor/` in `gsd-check-update.js` and `gsd-statusline.js`. The hooks were still pointing to Claude Code's config directory, causing cache and VERSION file lookups to fail on Cursor.

## [1.0.0] - 2026-01-25

### Added

- Initial Cursor IDE adaptation of GSD (based on [glittercowboy/get-shit-done](https://github.com/glittercowboy/get-shit-done))
- Complete adaptation guide (`GSD-CURSOR-ADAPTATION.md`)
- Migration documentation for future updates
- Installation scripts for Windows and macOS/Linux
- Migration scripts for Windows (PowerShell) and macOS/Linux (Bash)
- All 27 commands, 11 agents, 12 workflows, 20+ templates, 9 references, and 2 hooks

### Changed

- Command prefix from `/gsd:` to `/gsd-` (Cursor convention)
- Configuration directory from `~/.claude/` to `~/.cursor/`
- Tool names from PascalCase to snake_case
- Frontmatter tools format from array to object with booleans
- Color values from names to hex codes

### Mapping

| Original | Adapted |
|----------|---------|
| `gsd:new-project` | `gsd/new-project` |
| `~/.claude/` | `~/.cursor/` |
| `allowed-tools: [Read, Write]` | `tools: { read: true, write: true }` |
| `color: yellow` | `color: "#FFFF00"` |

---

## Migration Notes

When updating from GSD master, see [MIGRATION.md](./MIGRATION.md) for detailed instructions.


