# kmp-release

A [Claude Code skill](https://code.claude.com/docs/en/skills) that sets up a complete release pipeline for Kotlin Multiplatform projects.

## What it does

When invoked, the skill guides Claude through setting up:

- **Version management** — Gradle property + generated Kotlin constants (version & git revision)
- **Android signing** — Environment-based keystore config for CI (no impact on local builds)
- **ProGuard / R8** — Shared rules for Android & Desktop, covering kotlinx-serialization, Ktor, Netty, and other common KMP dependencies
- **APK cleanup** — Excludes unnecessary metadata and native libraries from the Android APK
- **GitHub Actions workflow** — Tag-triggered release with matrix builds:
  - CLI via GraalVM native-image (macOS, Linux, Windows × x64/arm64)
  - Android APK with release signing
  - Compose Desktop packages (DMG, DEB, MSI)
  - WasmJs browser distribution
- **Auto-generated changelog** via GitHub's release notes

## Install

```bash
npx skills add linroid/kmp-release
```

Or manually copy `SKILL.md` into your project's `.claude/skills/kmp-release/` directory.

## Usage

In Claude Code, run:

```
/kmp-release
```

You can optionally specify which platforms to include:

```
/kmp-release android,desktop,web
```

If no platforms are specified, all are included by default.

## Requirements

- A Kotlin Multiplatform project using Gradle with a version catalog (`libs.versions.toml`)
- GitHub repository (for the Actions workflow)

## License

MIT
