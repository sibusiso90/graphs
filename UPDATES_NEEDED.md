# IntentLock — Updates Needed

Generated: 2026-04-29
Branch: main
Source of truth: direct code inspection of `lib/`, `ios/`, `android/`, `functions/`, `firestore.rules`.

---

## 1. Status overview

| Area | State |
|---|---|
| App shell, router, auth | Working |
| Firestore rules + indexes | Working |
| Cloud Functions (OTP, contacts, audit share) | Working |
| KMS / phone encryption | Working |
| Schedule editor + repository | Working |
| Settings → restricted apps picker | Working |
| Unlock ceremony (friction → OTP → grant) | Working |
| Audit screen + JWT share | Working |
| **Onboarding → contact persistence** | **Broken — not wired** |
| **Onboarding → restricted apps persistence** | **Broken — not wired** |
| iOS DeviceActivityMonitor extension | Partial — duplicate file, single-day window |
| iOS event channel back to Flutter | Dead — never emits |
| Android multi-permission flow | Partial — UI doesn't refresh after grant |
| Multi-schedule enforcement | Single-schedule only |
| Billing | Stub — must replace |
| Android release signing | Debug keystore — Play will reject |

`flutter analyze` → 0 issues. No real tests (only the default counter widget test).

---

## 2. Blockers (must fix before the app can work end-to-end)

### B1. Onboarding does not save the trusted contact
- **File:** `lib/features/onboarding/presentation/steps/step_contact.dart`
- **Problem:** "Continue" button only calls `OnboardingStep.apps`. The name + phone fields are never sent to `ContactRepository.add()`.
- **Consequence:** First unlock attempt throws `failed-precondition: No trusted contact set.` (functions/src/otp.ts:58).
- **Fix:** Call `ref.read(contactRepositoryProvider).add(name: ..., phoneE164: ..., isPrimary: true)` before navigating; gate "Continue" on success; surface errors.

### B2. Onboarding does not save selected apps and uses wrong data on iOS
- **File:** `lib/features/onboarding/presentation/steps/step_apps.dart`
- **Problem:** Uses a hardcoded list of 8 Android package names. `_continue()` never calls `RestrictedAppsRepository.setAll()` or `lockPlatform.applyRestrictions()` or `showAppPicker()`. iOS Family Controls operates on opaque tokens, so those Android package strings cannot map to anything.
- **Fix:** Replace the hardcoded sheet with `lockPlatform.showAppPicker()` (already opens FamilyActivityPicker on iOS, AppPickerActivity on Android). Then `setAll(uid, picked)` + `applyRestrictions(picked)`. Mirror the working flow in `lib/features/settings/presentation/settings_screen.dart` lines 60–78.

### B3. iOS event channel never emits
- **File:** `ios/Runner/LockPlatformBridge.swift`
- **Problem:** `LockEventStreamHandler` is created but `eventHandler.send(...)` is never called. The DeviceActivityMonitor extension runs in a separate process and cannot post directly to a Flutter MethodChannel.
- **Fix:** Use a Darwin notification (`CFNotificationCenter`) or App Group write that the host listens for on app resume; then forward into the event channel. Alternatively, on each `applicationDidBecomeActive`, read SharedState and emit synthesized events.

### B4. iOS schedule supports only one daily window
- **File:** `ios/Runner/LockPlatformBridge.swift` (`setSchedule`)
- **Problem:** Code comment admits a single `DeviceActivitySchedule` with `repeats: true` is registered. So a Mon-only schedule still fires every day.
- **Fix:** Iterate ISO weekdays, register one `DeviceActivityName` + schedule per active day (e.g., `intentlock.focus.monday`, `.tuesday`, …).

### B5. Multi-schedule support is UI-only
- **Files:** `ios/Runner/LockPlatformBridge.swift`, `android/app/src/main/kotlin/.../lock/LockBridge.kt`
- **Problem:** Both platforms' `setSchedule` overwrite the previous registration. With 3 schedules in Firestore, only the last saved is enforced by the OS.
- **Fix:** Either restrict Free tier to 1 schedule and gate the editor, or have the bridge accept a list of schedules and register all of them.

### B6. Duplicate iOS extension file
- **Files:** `ios/IntentLockMonitor/DeviceActivityMonitorExtension.swift` (Xcode-generated empty stub) and `ios/IntentLockMonitor/IntentLockMonitor.swift` (the real logic).
- **Problem:** Only one can be `NSExtensionPrincipalClass`. SETUP.md §4 said to replace the stub — both currently coexist.
- **Fix:** Verify `Info.plist` for the extension target points `NSExtensionPrincipalClass` at `IntentLockMonitor`; delete `DeviceActivityMonitorExtension.swift`.

### B7. Paywall writes `tier` from the client
- **File:** `lib/features/settings/presentation/paywall_screen.dart` lines 43–46
- **Problem:** Calls `userRepository.upsert(profile.copyWith(tier: tier))` after a "successful" purchase. Firestore rules deny client `tier` updates (firestore.rules:19–21). Even when billing is real, this 403s.
- **Fix:** Add a Cloud Function (e.g. `setSubscriptionTier`) that verifies the receipt with the App Store / Play Store and writes `tier`. Remove the client-side `upsert(tier)`.

### B8. Android release signing
- **File:** `android/app/build.gradle.kts` lines 35–37
- **Problem:** `signingConfig = signingConfigs.getByName("debug")` for release. Play Store will reject.
- **Fix:** Generate a release keystore, configure via `signingConfigs { release { ... } }`, load credentials from environment / `key.properties` (don't commit).

---

## 3. Connected but incomplete

| # | File / Area | Issue | Fix |
|---|---|---|---|
| C1 | `ios/Runner/LockPlatformBridge.swift` lines 108–111 | Display names hardcoded as "App 1", "App 2"… | Persist a user-given alias when adding to the picker, or label as "Restricted app". |
| C2 | `android/.../LockBridge.kt` lines 40–46 | `requestAuthorization` returns success immediately after launching Settings. | Recheck on `AppLifecycleState.resumed`; invalidate `lockAuthorizedProvider`. |
| C3 | iOS `LockPlatformBridge.swift` + Android `LockBridge.kt` `setSchedule` | `timezone` field is ignored. | Parse the IANA timezone, convert window times, or document that device-local time is used. |
| C4 | `android/app/src/main/AndroidManifest.xml` lines 37–43 | Only `intentlock://` scheme declared. No `https://intentlock.app/...` autoVerify intent filter. | Add intent filter + host `assetlinks.json` per SETUP.md §8. |
| C5 | `functions/src/otp.ts` lines 51–60 | OTP send does not gate on `verificationStatus === "verified"`. | Refuse with `failed-precondition` if contact is not verified. |
| C6 | `functions/src/audit.ts` + `otp.ts` | Audit UI handles `sessionStarted`/`sessionEnded`/`scheduleStarted`/`scheduleEnded`/`bypassDetected` but nothing emits them. | Either emit from the platform → Function path, or remove from the enum. |
| C7 | `lib/features/onboarding/presentation/steps/step_friction.dart` lines 24–43 | Only `reflection` is wired. `delay`/`breathing` show SOON; `miniGame`/`socialApproval` exist in enum but unused. | Implement `delay` + `breathing` in the unlock flow, or remove the unused enum cases. |
| C8 | `lib/data/repositories/billing_service.dart` | `StubBillingService` always returns `billing/not-configured`. | Implement with `in_app_purchase` or RevenueCat per SETUP.md §9. |
| C9 | `lib/features/settings/presentation/settings_screen.dart` lines 142–150 | Privacy and Terms rows have `onTap: () {}`. | Use `url_launcher` to open hosted URLs. |
| C10 | `android/.../lock/LockMonitorService.kt` line 30 | 1.5 s polling cadence — battery + race-prone. | Slower poll + JobScheduler / WorkManager waking at schedule boundaries. |
| C11 | `android/.../lock/AppPickerActivity.kt` lines 21–23 | Pending `MethodChannel.Result` held in a static var across activity lifecycle. | Wire via Activity result API or use a coroutine continuation that survives recreation. |
| C12 | `lib/app/bootstrap.dart` | `firebase_remote_config` and `firebase_analytics` imported but unused. | Either log key events / fetch flags, or drop the dependencies. |
| C13 | `functions/.env` | File missing. | Create with `KMS_PROJECT`, `KMS_LOCATION`, `KMS_KEYRING`, `KMS_KEY`, `PUBLIC_BASE_URL` per SETUP.md §3. |
| C14 | `README.md` | Still the default Flutter scaffold text. | Replace with a real project README pointing to SETUP.md. |

---

## 4. Documentation / config drift

These do not break anything (the code is internally consistent), but the docs disagree with the code.

| Symbol | SETUP.md says | Actual code |
|---|---|---|
| Bundle ID | `co.za.mahlanguelites.intentlock` | `za.co.mahlanguelites.intentlock` |
| App Group | `group.co.za.mahlanguelites.intentlock` | `group.za.co.mahlanguelites.intentlock` |
| Android source path | (implicit) | `android/app/src/main/kotlin/co/za/mahlanguelites/intentlock/` (path) but Kotlin `package za.co.mahlanguelites.intentlock` (declaration) |

**Action:** Update SETUP.md everywhere to `za.co.mahlanguelites.intentlock` and `group.za.co.mahlanguelites.intentlock`. Do not touch the code — would be a multi-day rename.

---

## 5. Recommended order of work

1. **B1, B2** — wire onboarding to repositories. ~30 min. Unblocks all downstream testing.
2. **B6** — clean up duplicate iOS extension file; verify a debug build launches with FamilyControls + App Group capabilities.
3. **B3, B4** — emit events from native, register per-weekday schedules.
4. **B5** — multi-schedule enforcement (or gate at 1 for Free).
5. **C2, C4** — Android post-permission refresh + universal links.
6. **B7** — server-side `setSubscriptionTier` callable; strip client-side `tier` writes.
7. **B8, C14** — release signing + real README before TestFlight / Play upload.
8. **C5, C6** — verification gate + audit event emission.
9. **C8** — wire real billing.

---

## 6. Verification checklist (after fixes)

- [ ] Sign in → onboarding → finish → home shows the picked contact's masked phone in Contacts screen.
- [ ] Home → Restricted apps shows the apps selected during onboarding (not just from Settings).
- [ ] iOS: opening a restricted app during a schedule window triggers the FamilyControls shield.
- [ ] iOS: weekly schedule with only Mon selected does **not** fire on Tue.
- [ ] iOS: unlock session re-arms shield at expiry across app termination + reboot.
- [ ] Android: opening a restricted app during a schedule window deep-links into `intentlock://unlock/<pkg>`.
- [ ] Android: after granting USAGE_STATS in Settings, returning to app shows authorized state without re-tapping "Grant access".
- [ ] OTP: requesting an unlock when the contact is unverified is refused.
- [ ] OTP delivery confirmed end-to-end via Meta WhatsApp template.
- [ ] Audit share link resolves on web (`https://intentlock.app/share/audit/<token>`).
- [ ] Paywall purchase happy-path writes `tier` server-side; client `upsert(tier)` removed.
- [ ] Release Android build is signed with a non-debug key.
