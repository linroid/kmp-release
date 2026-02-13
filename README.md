# kmp-release

An [agent skill](https://skills.sh) that sets up a complete release pipeline for Kotlin Multiplatform projects. Works with any AI coding agent that supports the [Skills](https://github.com/vercel-labs/skills) ecosystem — Claude Code, Cursor, Codex, Windsurf, and [30+ more](https://github.com/vercel-labs/skills#supported-agents).

## What it does

When invoked, the skill guides your AI agent through setting up:

- **Version management** — Gradle property + generated Kotlin constants (version & git revision)
- **Android signing** — Environment-based keystore config for CI (no impact on local builds)
- **ProGuard / R8** — Shared rules for Android & Desktop, covering kotlinx-serialization, Ktor, Netty, and other common KMP dependencies
- **APK cleanup** — Excludes unnecessary metadata and native libraries from the Android APK
- **GitHub Actions workflow** — Tag-triggered release with matrix builds:
  - CLI via GraalVM native-image (macOS, Linux, Windows x x64/arm64)
  - Android APK with release signing
  - Compose Desktop packages (DMG, DEB, MSI)
  - WasmJs browser distribution
- **Auto-generated changelog** via GitHub's release notes

## Install

```bash
npx skills add linroid/kmp-release
```

This installs the skill for all supported agents detected on your system. Use `-a <agent>` to target a specific agent (e.g., `-a claude-code`, `-a cursor`).

## Usage

Invoke the skill through your agent's skill/command system:

```
/kmp-release
```

Optionally specify which platforms to include:

```
/kmp-release android,desktop,web
```

If no platforms are specified, all are included by default.

## Requirements

- A Kotlin Multiplatform project using Gradle with a version catalog (`libs.versions.toml`)
- GitHub repository (for the Actions workflow)

## License

MIT
