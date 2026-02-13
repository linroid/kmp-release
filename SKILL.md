---
name: kmp-release
description: Set up a complete release pipeline for a Kotlin Multiplatform project with GitHub Actions, version injection, ProGuard, signing, and multi-platform artifacts.
argument-hint: "[platforms: android,desktop,web,cli,ios]"
---

Set up a production release pipeline for this Kotlin Multiplatform project. The user may specify which platforms to include as arguments (default: all).

Platforms requested: $ARGUMENTS

## Prerequisites

Before starting, read the project's existing build files (`build.gradle.kts`, `gradle.properties`, `settings.gradle.kts`, version catalog) to understand the module structure, existing dependencies, and naming conventions.

## Step-by-step checklist

Work through each applicable step. Skip steps for platforms not requested. Always read existing build files before modifying them.

### 1. Version Management

- Add a `<project>.version` property to `gradle.properties` with a dev default (e.g., `0.0.1-dev`)
- In the shared/API module's `build.gradle.kts`, create a `generateVersion` task that:
  - Reads the version from the Gradle property
  - Reads the git short hash via `providers.exec { commandLine("git", "rev-parse", "--short", "HEAD") }`
  - Generates a Kotlin file with `internal const val` for both version and revision
  - Declares `inputs.property` for both values so Gradle invalidates correctly
  - Registers the output dir as a `kotlin.srcDir` in `commonMain`
- Expose version/revision via public companion object vals in the shared module
- Remove any hardcoded version constants elsewhere — reference the generated values
- For Android: use the Gradle property for `versionName`, use `GITHUB_RUN_NUMBER` env var for `versionCode` (default `1`)
- For Desktop (Compose): derive `packageVersion` from the Gradle property, strip pre-release suffix, clamp MAJOR > 0 (DMG/MSI requirement)
- For iOS: create a `Config.xcconfig` with `MARKETING_VERSION` and `CURRENT_PROJECT_VERSION`

### 2. Android Signing

- Add a `signingConfigs` block that reads from environment variables:
  - `ANDROID_KEYSTORE_FILE`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`
  - Only create the config if `ANDROID_KEYSTORE_FILE` env var is set (so local builds are unaffected)
- Set `buildTypes.release.signingConfig` conditionally via `signingConfigs.findByName("release")`
- Required GitHub secrets: `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`
- CI step to decode: `echo "$ANDROID_KEYSTORE_BASE64" | base64 -d > /tmp/release.keystore`

### 3. ProGuard / R8

- Create a shared `proguard-rules.pro` at the project root. Inspect the project's dependencies and add rules for:
  - **kotlinx-serialization** (if used): keep `Companion`, `$$serializer`, and `@Serializable` class members
  - **dontwarn** rules for common KMP libraries that have optional/platform-specific dependencies: Ktor, kotlinx-coroutines, Netty, SLF4J, Logback, Log4J2, OSGi, SQLDelight, etc.
- Android: enable `isMinifyEnabled = true` and `isShrinkResources = true`, reference `rootProject.file("proguard-rules.pro")`
- Desktop: enable via `buildTypes.release.proguard { configurationFiles.from(rootProject.file("proguard-rules.pro")) }`
- Desktop with Ktor/Netty: add `runtimeOnly("org.apache.logging.log4j:log4j-api:<version>")` so ProGuard can resolve Netty's Log4J2Logger class hierarchy (it gets shrunk away in the final output)

### 4. Logging (Ktor Server modules)

If the project uses Ktor server (e.g., embedded server, daemon mode):

- Add `ch.qos.logback:logback-classic` as `implementation` to any JVM module that runs a Ktor server
- Add the dependency to the version catalog if not already there

### 5. APK Packaging Cleanup

Exclude unnecessary files from the Android APK via `packaging.resources.excludes`. Common candidates:
```
META-INF/{INDEX.LIST,io.netty.versions.properties}
META-INF/*.version
META-INF/native-image/**
META-INF/version-control-info.textproto
META-INF/com/android/build/gradle/app-metadata.properties
META-INF/androidx/**
kotlin/**
DebugProbesKt.bin
**/*.properties
```

Inspect the APK contents (`unzip -l *.apk`) to identify additional files that can be excluded.

### 6. GitHub Actions Release Workflow

Create `.github/workflows/release.yml`:
- **Trigger**: `on: push: tags: ['v*']`
- **Permissions**: `contents: write`
- **Top-level env**: extract version from tag name
- **All Gradle commands** get `-P<project>.version=<version>` (strip the `v` prefix)

#### Build Jobs

Include only the jobs for requested platforms:

**CLI (GraalVM native-image)** — if the project has a CLI module with GraalVM native-image support:

Matrix across target platforms:
| Name | Runner | OS | Arch | Package |
|------|--------|----|------|---------|
| macOS-arm64 | `macos-14` | macos | aarch64 | tar.gz |
| macOS-x64 | `macos-13` | macos | x64 | tar.gz |
| Linux-x64 | `ubuntu-latest` | linux | x64 | tar.gz |
| Linux-arm64 | `ubuntu-24.04-arm` | linux | aarch64 | tar.gz |
| Windows-x64 | `windows-latest` | windows | x64 | zip |
| Windows-arm64 | `windows-11-arm` | windows | aarch64 | zip |

Setup: `graalvm/setup-graalvm@v1` with java-version 21.
Build: `./gradlew :<cli-module>:nativeCompile`.
Package: `tar -czf` (Unix) or `Compress-Archive` (Windows).

**Android APK**:
- Runner: `ubuntu-latest`, Java 17
- Decode keystore from secret, build with `assembleRelease`
- Rename APK to a consistent name (e.g., `<project>-android.apk`)

**Desktop (Compose packaging)** — matrix:
| Name | Runner | Task | Artifact |
|------|--------|------|----------|
| macOS | `macos-latest` | packageDmg | *.dmg |
| Linux-x64 | `ubuntu-latest` | packageDeb | *.deb |
| Linux-arm64 | `ubuntu-24.04-arm` | packageDeb | *.deb |
| Windows-x64 | `windows-latest` | packageMsi | *.msi |
| Windows-arm64 | `windows-11-arm` | packageMsi | *.msi |

Setup: Java 17.

**Web (WasmJs)**:
- Runner: `ubuntu-latest`, Java 17
- Build: `wasmJsBrowserDistribution`
- Zip the output directory

#### Release Job
- `needs: [all build jobs]`
- Download all artifacts with `actions/download-artifact@v7` and `merge-multiple: true`
- Create release with `softprops/action-gh-release@v2` and `generate_release_notes: true`

### 7. Verification

After setup, instruct the user to:
1. Push a test tag: `git tag v0.0.1-test && git push origin v0.0.1-test`
2. Verify all jobs pass and artifacts appear on the GitHub Release
3. Delete the test release/tag afterward

## Important notes

- Match action versions to what the project already uses (check existing workflows)
- Use `gradle/actions/setup-gradle@v4` or later for Gradle caching
- Use `actions/upload-artifact@v4` / `actions/download-artifact@v4` or later
- Always verify builds compile locally before committing
- Use Conventional Commits for commit messages
- Adapt module paths (`:app:android`, `:cli`, etc.) to match the project's actual structure
