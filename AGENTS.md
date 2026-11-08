# AGENTS.md

Guidance for AI coding agents working in this repo. Human-facing rules live in `CONTRIBUTING.md` — don't duplicate them here.

## Fork automation (this repo = zhnsanso/Sesame-AG)

### Architecture
- `main` = default branch, carries ONLY fork-owned files (`.github/workflows/fork-release.yml`, this AGENTS.md). All fork changes go here.
- `sync` = pure fast-forward mirror of upstream `Sesame-AG/Sesame-AG` `dev`. The weekly workflow (Mon 11:00 Beijing) updates it via direct ref push (`git push <upstream-sha>:refs/heads/sync`, no merge), builds, signs, and publishes a GitHub Release tagged `v<version>-g<shortsha>` with `arm64-v8a` + `universal` APKs. Weeks with no upstream change skip in ~2 min.
- All three paths are verified working: fresh build/release, no-change skip, and `force_build` rerun (idempotent: existing tag is not recreated, release assets are clobber-uploaded).

### Invariants — breaking these silently breaks the fork
- **Never commit anything to `sync`.** It must stay byte-identical to upstream `dev`; any local commit breaks no-change detection, ff-sync, and `versionCode = git rev-list --count HEAD` (valid only on a pure mirror).
- **Signing keystore**: GitHub secrets `FORK_KEYSTORE_*`; the only offline backup is `/root/.sesame-fork/` (jks + password). Losing it means future APKs can no longer update-install over existing ones. If the keystore is ever regenerated, the cert SHA-256 hardcoded in the workflow's release body must be updated too.
- Fork tags must never match `[0-9]*-*` (upstream's release-workflow trigger; that workflow is disabled here — don't re-enable it or the issue-bot workflows).
- Don't "simplify" two workflow details: (1) the 3-attempt retry around `./gradlew` (services.gradle.org wrapper downloads time out transiently), (2) reading `appVersion` from `git show <upstream-sha>:gradle.properties` rather than the working tree (the checked-out mirror may be stale when the tag is computed).
- `gh` CLI is authenticated as `zhnsanso` (scopes incl. `repo`, `workflow`).

## What this is

LSPosed/Xposed module (Kotlin + Java, single Gradle module `:app`) that automates Alipay (`com.eg.android.AlipayGphone`) features (Ant Forest, Ant Farm, etc.) by hooking Alipay's RPC layer. Docs, commit messages, and many code comments are in Chinese.

## Branches — read before doing anything

- **All development happens on `dev`; PRs must target `dev`.** Release tags must point at commits on `dev`.
- The default branch `main` is a docs/CI mirror that lags far behind `dev` (600+ commits) and has a **different package layout**: `fansirsqi.xposed.sesame.*` on `main` vs `io.github.aoguai.sesameag.*` on `dev`. Check which branch you're on before grepping for classes or editing code. `CONTRIBUTING.md` and the workflows describe the `dev` baseline.

## Build & verification

- JDK 17 (CI pins Oracle JDK 17.0.1); Java/Kotlin target 17.
- Debug build: `./gradlew :app:assembleDebug` — no NDK needed; there are no committed C++ sources.
- Mirror the CI path: `CI=true ./gradlew --no-daemon --stacktrace :app:assembleRelease`. The `CI` env var switches build mode (skips CMake, pins ABI filters).
- **There is no test suite.** Unit tests are force-disabled in `app/build.gradle.kts` (`testOptions.unitTests.all { it.enabled = false }`). PR CI "verification" is compile-only:
  `./gradlew --no-daemon --stacktrace :app:compileDebugKotlin :app:compileDebugJavaWithJavac`
  Run that before pushing; it is the gate your PR will face.
- ABI splits are on: every build produces armeabi-v7a / arm64-v8a / x86 / x86_64 + universal APKs.

## Toolchain pins — do not bump casually

CI (`pull_request_check.yml`, `android.yml`) greps for **exact** versions in `gradle/wrapper/gradle-wrapper.properties`, `gradle/libs.versions.toml`, and `app/build.gradle.kts` (Gradle 9.5.1 / AGP 9.2.1 / Kotlin 2.2.10 / compileSdk 37 / build-tools 37.0.0 / Java 17 compat on `dev`). Upgrading any of them requires updating the env block in **both** workflows (and `CONTRIBUTING.md`) in the same change, or CI fails at "Validate Project Toolchain Baseline". Don't upgrade toolchain versions "while you're in there".

## Versioning & release

- On `dev`, `appVersion` in `gradle.properties` drives `versionName`; `versionCode` is computed at build time from `git rev-list --count HEAD` (requires full git history — don't shallow-clone).
- Release = push tag `<versionCode>-<versionName>` (e.g. `410-0.2.6`). Preflight rejects the tag unless the version code matches the commit count, the name matches `appVersion`, and the commit is on `dev`.
- APKs are signed in CI with repo secrets and fingerprint-verified. Local release builds silently fall back to the debug keystore.

## Repo layout

- `app/src/main/java/.../sesame/`:
  - `hook/` — Xposed entrypoints (`xp82/HookEntry`, `lsp100/HookEntry` on `main`; `libxposed/HookEntry` via `resources/META-INF/xposed/java_init.list` on `dev`), `ApplicationHook` orchestration, `rpc/` (hooked RPC bridge), `server/` (embedded NanoHTTPD server exposing the WebUI/RPC over HTTP)
  - `model/` — `ModelField(s)` definitions that generate both task settings and their UI; UI code often follows automatically from model changes
  - `task/` — one package per Alipay feature (antForest, antFarm, antOcean, ...) plus the scheduler (`ModelTask`, `TaskExecutor`)
  - `ui/` — Compose settings UI, log viewer, manual task runner
- `app/libs/` — bundled Xposed API aar/jars on `main`; `dev` uses Maven Central artifacts `io.github.libxposed:api/service:102.0.0` (`mavenLocal` is only a fallback in `settings.gradle.kts`).
- `libxposed/` at repo root — orphaned gitlinks (no `.gitmodules`, not a Gradle project). Ignore; not part of the build.
- `serve-debug/` — standalone Python tooling for debugging the module's embedded Web UI locally (`uv run webui.py`); unrelated to the Android build.
- `AppIdMap.txt` — Alipay appId → name map used for targeting; note it's tracked despite `*.txt` being ignored (see below).

## .gitignore traps

- `*.txt` and `*.json` are **globally gitignored** (AppIdMap.txt only survives because it was force-added). New `.txt`/`.json` files are silently skipped by `git add` — use `git add -f` or another extension.
- `/app/src/main/cpp/` is intentionally ignored: native sources, when they exist, are local-only and never pushed.
- `*.apk`, `*.jks` keystores, and `local.properties` are ignored as usual.

## Conventions

- Use conventional-commit types (`feat:`, `fix:`, ...). Release notes are generated from commit subjects; `docs`/`style`/`ci`/`test`/`build` commits are excluded from them.
- Keep changes focused; never mix reformatting with functional changes (maintainers reject large unexplained refactors).
- Supported environment is deliberately narrow: LSPosed with libxposed API 102+, rooted, Android 16+. Don't add compatibility code for other Xposed frameworks or older Android without being asked.
