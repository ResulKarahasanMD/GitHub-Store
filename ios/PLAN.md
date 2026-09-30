# GitHub-Store iOS Port — Plan

> This document records the architectural contract amendments for the iOS port of GitHub-Store (a.k.a. GitHub Releases Explorer on iOS). If anything here conflicts with upstream, resolve it here before proceeding.

## Source of Truth

1. **Upstream repo:** `https://github.com/komi-store/komi-store` (moved from `OpenHub-Store/GitHub-Store` in upstream commit `e26fb15f`). This checkout is the fork `ResulKarahasanMD/GitHub-Store`.
   - Treat existing `core/` and `feature/` Kotlin modules as the behavioral spec.
   - Reuse GitHub API integration, search scoring, release parsing, caching logic via KMP.
2. **This document** is the architectural contract. Any drift from upstream must be resolved here.

---

## Status (2026-09-30)

- Phase 0 is partially started, nothing else.
- Commit `7be72cbf`: `iosArm64` / `iosX64` / `iosSimulatorArm64` targets added to `core/domain` only. `skie` (0.10.2) and `ktor-client-darwin` added to `gradle/libs.versions.toml`, but no build script applies or depends on them yet.
- Not done: `:shared` module, `iosApp/`, iOS targets on `core/data` and feature modules. An iOS compile of `core/domain` has not been verified.
- Amendment 2 (auth) was rewritten on 2026-09-30 after upstream moved to web OAuth + PKCE + backend handoff. The earlier version assumed Device Flow was primary and that iOS had to implement PKCE and the code exchange itself.

---

## Locked Architectural Decisions

- **Shared logic:** Kotlin Multiplatform
- **iOS targets:** `iosArm64`, `iosX64`, `iosSimulatorArm64`
- **Output:** XCFramework
- **iOS UI:** SwiftUI (not Compose Multiplatform for iOS)
- **Kotlin ↔ Swift bridge:** SKIE (for Flow, sealed classes, default args, nullable generics)
- **Networking:** Ktor with Darwin engine on iOS
- **DI:** Koin in shared + Swift-side initializer
- **Token storage:** shared `DefaultTokenStore` (`core/data`) on KSafe, same as Android/Desktop. KSafe 2.0.0 publishes `iosArm64`, `iosX64` and `iosSimulatorArm64` artifacts on Maven Central (README: iOS 13+). No Swift Keychain code for tokens (see Amendment 2).
- **Minimum iOS:** 16.0 (for `NavigationStack`)
- **Markdown:** `gonzalezreal/swift-markdown-ui`
- **Images:** Kingfisher
- **Distribution:** TestFlight for beta, App Store for release

---

## Contract Amendments

### Amendment 1 — Persistence (BLOCKED → RESOLVED)

**Status:** Option B accepted.

**Decision:** Keep Room for Android/Desktop; add SQLDelight on iOS with a thin abstraction.

- Define persistence interfaces in `core/domain` as pure Kotlin interfaces (e.g. `FavouritesDao`, `InstalledAppsDao`, `StarredDao`).
- Room DAOs (Android/JVM) implement these interfaces.
- SQLDelight-backed implementations (iOS) implement the same interfaces.
- Repositories in `core/data` depend only on the interface, not on Room directly.
- Schema migrations tracked twice (manually). Structural parity enforced and documented in `PERSISTENCE.md`.

**Reasoning:**
- Option A (full SQLDelight migration) is a 2–4 week rewrite with zero user-facing value.
- Option C (Room KMP on Native) is experimental (`@ExperimentalRoomApi`), KSP-on-Native has known code-generation timing issues, and toolchain updates could break the iOS build unpredictably.

### Amendment 2: Authentication (rewritten 2026-09-30)

**Status:** Reuse the shared web-OAuth handoff flow. iOS contributes only the browser session.

**How upstream signs in today** (from `feature/auth/CLAUDE.md` and the code):

1. `AuthenticationRepository.registerWebAuth()` generates `(state, codeVerifier, codeChallenge)` with `PkceGenerator` and POSTs them to `https://github-store.org/auth/register` (Cloudflare Worker). The Worker returns `auth_url` (`github.com/login/oauth/authorize?...`).
2. The user authorizes on GitHub. GitHub redirects to `https://github-store.org/auth/callback`. The Worker, via `api.github-store.org`, exchanges the code with the stored verifier and keeps `handoffId -> access_token` for 60 s.
3. The Worker redirects to the `githubstore://auth` custom scheme. `DeepLinkParser.parseAuthCallback` accepts `?handoff=<id>&state=<state>` or `?error=<reason>&state=<state>`.
4. The app compares `state` with the pending one, then `exchangeWebAuthHandoff(handoffId)` POSTs `api.github-store.org/v1/oauth/handoff/<id>` (single-use) and saves the token through `TokenStore`.

Fallbacks: device flow (backend proxy, escalates to direct `github.com` on infra errors) and PAT paste. The app never receives the OAuth `code` and never calls GitHub's token endpoint. The code exchange is server-side.

**Decision:**

- iOS uses the same flow. No iOS-specific GitHub OAuth App, redirect URI or backend change. Tokens come from the same OAuth App (`GITHUB_CLIENT_ID`).
- Swift's only job (`iosApp/Core/Auth/`): open `auth_url` in `ASWebAuthenticationSession` with `callbackURLScheme: "githubstore"`, then pass the returned callback URL to shared code. The session hands the callback URL to its completion handler, so sign-in does not depend on Info.plist URL routing. `githubstore` is still declared in `CFBundleURLTypes` for the `repo` / `apps` deep links. A stray `githubstore://auth` URL arriving through `onOpenURL` goes through the same `state` check.
- `prefersEphemeralWebBrowserSession = false`, so an existing github.com login in Safari is reused. Cost: the system consent alert ("wants to use github-store.org to sign in").
- `:feature:auth:domain` and `:feature:auth:data` are included in `:shared`. `:feature:auth:presentation` is excluded (Compose UI, `BrowserHelper`, Compose resources). A SwiftUI auth screen replaces it.
- iOS v1 sign-in paths: web OAuth (primary) and PAT paste (advanced). PAT is the fallback when `github-store.org` is unreachable. Device flow code ships inside the shared framework but is not shown in the v1 iOS UI. It can be turned on later without Kotlin changes.

**Shared-code changes required** (upstream-owned code; send as upstream PRs so Android/Desktop keep one implementation):

1. `feature/auth/data/.../crypto/PkceGenerator.kt` imports `java.security.SecureRandom` and `java.security.MessageDigest` in `commonMain`, which cannot compile for iOS. Put random bytes and SHA-256 behind `expect`/`actual` (JVM/Android: current code; iOS: `SecRandomCopyBytes` + CommonCrypto `CC_SHA256`) or use a KMP crypto library. Keep S256, 32-byte state, 48-byte verifier, unpadded base64url.
2. `feature/auth/data/.../repository/AuthenticationRepositoryImpl.kt` uses `java.util.concurrent.TimeoutException` and `System.currentTimeMillis()` in `commonMain`. Replace them with a common exception type and `kotlin.time.Clock` (already used by `DefaultTokenStore`). It also uses `Dispatchers.IO`; confirm it resolves in the iOS compilation.
3. Move auth-callback parsing (`parseAuthCallback` in `composeApp/.../app/deeplink/DeepLinkParser.kt`: `state` required, base64url, max 256 chars; `error` cut to 64 chars of `[A-Za-z0-9_-]`) into `feature/auth`, so iOS does not re-implement the validation. `composeApp` calls the moved function.
4. The pending `state` and the comparison live in `AuthenticationViewModel` (`SavedStateHandle`, `KEY_WEB_AUTH_STATE`), which iOS does not use. Preferred: extract a small shared coordinator (register, remember expected state, validate callback, exchange) that both the ViewModel and Swift call. Alternative: keep the expected state in the Swift coordinator.
5. `DefaultTokenStore` takes a `legacyDataStore` for a one-time DataStore-to-KSafe migration that has no source on iOS. The iOS `PlatformModule` must supply an empty one, or the parameter becomes optional. Decide in Phase 1.

**Open verification items (before writing the Swift side):**

- Redirect format. `feature/auth/CLAUDE.md` says the Worker redirects to `githubstore://auth?h=<handoffId>`, but `DeepLinkParser` requires `handoff=` and `state=` and returns `None` otherwise. The Worker source is not in this repo. Confirm with a live sign-in, then fix whichever upstream doc is wrong.
- How the Worker reaches the custom scheme (HTTP 302 or an HTML/JS page). `ASWebAuthenticationSession` should catch either; confirm on a simulator or device.
- What KSafe uses for key storage on iOS (Keychain accessibility class). The README says AES-256-GCM, "hardware-backed where available"; the iOS specifics were not checked.
- App Review guideline 4.8 (Sign in with Apple). GitHub login here gives access to the user's own GitHub content, which should fall under the third-party-client exception. Recheck against the current guideline text before submission.

**Reasoning:**

- The handoff design already keeps the code exchange server-side and gives the app only a single-use, 60 s handoff id. Re-implementing PKCE and the token exchange in Swift would duplicate that and move the exchange into the app.
- `ASWebAuthenticationSession` is the platform-idiomatic in-app browser and keeps the UX goal of the old plan (no device code copy-paste).
- Keeping repository logic shared means iOS gets future auth fixes (fallback rules, error mapping) without Swift changes.

### Amendment 3 — Module Structure

**Status:** Create `:shared` umbrella module.

**Decision:** New `:shared` Gradle module:
1. Aggregates `:core:domain`, `:core:data`, and iOS-compatible feature modules.
2. Configures `iosArm64`, `iosX64`, `iosSimulatorArm64` targets.
3. Produces XCFramework via `assembleXCFramework`.
4. Applies SKIE plugin at this level.

**Exclusion list for iOS (maintained in `shared/README_iOS.md`):**
- `:feature:auth:presentation`: Compose UI; iOS has a SwiftUI auth screen. `:feature:auth:domain` and `:feature:auth:data` are **included** once the JVM-only APIs listed in Amendment 2 are removed from `commonMain`.
- `:feature:apps:data`, `:feature:apps:presentation` — Android-only (PackageManager, ApplicationInfo, Shizuku, etc.).
- Any module touching `PackageManager`, `ApplicationInfo`, `Context`, or JVM-only APIs.

---

## App Store Compliance Checklist

- [ ] App identity: **"GitHub Releases Explorer"** — no use of the word "Store" in app name, App Store Connect metadata, screenshots, or on-device copy.
- [ ] Never market, mention, or UX-optimize for IPA sideloading.
- [ ] Treat all asset types uniformly in UI (APK, EXE, DMG, DEB, RPM, AppImage, IPA).
- [ ] Default asset action set: **Share / Copy URL / Open in Browser** (via `SFSafariViewController`).
- [ ] AltStore / TrollStore / Sideloadly handoff is a **developer-mode feature**, default OFF in Settings, with clear "advanced users only" copy.
- [ ] No auto-download of IPAs, ever.
- [ ] Never bundle, package, or ship IPAs.
- [ ] Include `PrivacyInfo.xcprivacy`.
- [ ] Marketing screenshots prioritize DMG/EXE/APK views, not IPA.

---

## Code Quality Gates

- No TODO/FIXME in committed code unless flagged in the commit message and listed in `OPEN_ISSUES.md`.
- No hardcoded secrets. `GITHUB_CLIENT_ID` stays in `local.properties` and reaches the shared framework through BuildKonfig (`feature/auth/data`), same as Android/Desktop. Only the device-flow code reads it (`startDeviceFlow` refuses to start when it is blank); the web OAuth path does not. Swift needs no OAuth config. `.xcconfig` (gitignored) is only for signing settings such as the team ID.
- Every repository method in the shared module: unit test + exposed to Swift + documented.
- Every non-trivial SwiftUI view: `PreviewProvider`.
- No force unwraps (`try!`, `as!`, `!`) in production code, except in preview/test files.
- Every tappable element has `.accessibilityLabel`.
- Run swiftformat + swiftlint before every commit.
- Run ktlint / detekt on shared module before every commit.

---

## Git Hygiene

- Conventional commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`).
- Each phase = feature branch, PR to main.
- Never commit `.xcconfig` with secrets, `*.mobileprovision`, signing certs, or any file matching `*secret*` / `*.p8` / `*.p12`.
- `.gitignore` must include `xcuserdata/`, `DerivedData/`, `Pods/` (if used), `*.xcworkspace/xcuserdata`.

---

## Phases

### Phase 0 — Setup (1 week)

| # | Task |
|---|------|
| 1 | Fork upstream; add `iosApp/` alongside existing `composeApp/` |
| 2 | Extend shared/core Gradle modules with iOS targets (`iosArm64`, `iosX64`, `iosSimulatorArm64`) |
| 3 | Configure SKIE plugin in version catalog and `:shared` module |
| 4 | Create `:shared` umbrella module with XCFramework output |
| 5 | Write `SETUP.md` covering Apple Developer Program, `local.properties` (`GITHUB_CLIENT_ID`, same value as Android/Desktop), `.xcconfig` signing setup. No new GitHub OAuth App: the existing one redirects to `https://github-store.org/auth/callback` |
| 6 | Create minimal `iosApp` Xcode project (xcodegen) with bundle ID `org.github-store.ios` |
| 7 | Empty SwiftUI app opens, calls `GreetingKt.greet()`, renders result |
| 8 | Build check: `./gradlew :shared:assembleXCFramework` succeeds on clean machine |
| 9 | PR opened → pause for human review |

### Phase 1 — Core Shared Migration & Persistence Abstraction

| # | Task |
|---|------|
| 1 | Audit all `core/data` repositories — extract Room DAO dependencies into `core/domain` interfaces |
| 2 | Make `core/domain` iOS-compatible (ensure no JVM-only dependencies) |
| 3 | Add `iosMain` source sets to `core:data`, implementing Darwin Ktor engine, iOS-specific file paths, etc. |
| 4 | Create SQLDelight schema mirroring Room entities; write SQLDelight DAO implementations |
| 5 | Refactor repositories in `core:data` to depend on DAO interfaces, not Room types |
| 6 | Add unit tests for all repository methods |
| 7 | PR opened → pause |

### Phase 2 — Feature Module Audit & iOS Inclusion

| # | Task |
|---|------|
| 1 | Audit every `feature/x` module for iOS compatibility |
| 2 | Move Android-only dependencies behind `expect`/`actual` interfaces where needed |
| 3 | Include compatible feature modules in `:shared` dependencies |
| 4 | Land the Amendment 2 shared-code changes (PKCE `expect`/`actual`, JVM API cleanup, callback parser move, optional shared coordinator); include `:feature:auth:domain` + `:feature:auth:data`, exclude `:feature:auth:presentation` |
| 5 | PR opened → pause |

### Phase 3 — SwiftUI Shell + Navigation

| # | Task |
|---|------|
| 1 | Implement `NavigationStack`-based routing matching `GithubStoreGraph` |
| 2 | Create `HomeView`, `SearchView`, `DetailsView`, `ProfileView`, `SettingsView` shells |
| 3 | Bind shared ViewModels (via SKIE-exposed `StateFlow`) to SwiftUI `@State` |
| 4 | PR opened → pause |

### Phase 4: iOS Auth (shared web OAuth + `ASWebAuthenticationSession`)

| # | Task |
|---|------|
| 1 | Resolve the Amendment 2 open verification items (live redirect format, Worker redirect style, KSafe iOS storage) |
| 2 | `WebAuthSession` in Swift (`iosApp/Core/Auth/`): call shared `registerWebAuth()`, open `auth_url` in `ASWebAuthenticationSession` (`callbackURLScheme: "githubstore"`, non-ephemeral), return the callback URL; treat user cancel as a silent return to logged-out |
| 3 | Pass the callback URL to the shared parser + state check, then shared `exchangeWebAuthHandoff(handoffId)` right away (60 s TTL). Token lands in the shared KSafe `TokenStore` |
| 4 | SwiftUI auth screen: web sign-in button, PAT paste under "Advanced" (`signInWithPat`, show `PatRejectedException` kinds), error/retry states |
| 5 | Declare `githubstore` in `CFBundleURLTypes`; route `githubstore://auth` from `onOpenURL` through the same state check, `repo` / `apps` to navigation |
| 6 | Test on simulator: sign in, cancel, wrong `state`, expired handoff, sign out, relaunch keeps the token |
| 7 | PR opened, pause |

### Phase 5 — Screens & Polish

| # | Task |
|---|------|
| 1 | Asset list with uniform action sheet (Share / Copy / Open in Browser) |
| 2 | Markdown rendering for README via `swift-markdown-ui` |
| 3 | Kingfisher integration for avatar/repo image loading |
| 4 | Settings screen: developer-mode toggle for sideloading handoff + `PrivacyInfo.xcprivacy` |
| 5 | PR opened → pause |

### Phase 6 — Compliance & Submission

| # | Task |
|---|------|
| 1 | Final App Store metadata scrub (no "Store" wording) |
| 2 | Marketing screenshots prioritizing DMG/EXE/APK |
| 3 | Build signing + TestFlight upload |
| 4 | App Store submission |
| 5 | PR opened → pause |

---

## OPEN_ISSUES.md

Placeholder. No open issues at Phase 0 start.

