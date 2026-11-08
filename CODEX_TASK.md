# Task: package `index.html` as an Android APK (app name: StoreAffairs)

`index.html` is a finished, fully offline, single-file Arabic (RTL) web app (inventory count, "جرد الشئون").
All JS/CSS/images/libraries (SheetJS, html5-qrcode) are inlined. **Do not modify, split, minify or re-host index.html.**
Wrap it in a native Android WebView app and produce an installable APK that works with **no internet**.

## Deliverables
1. A complete Android Studio / Gradle project (Kotlin) in `/StoreAffairsApp`.
2. A built `StoreAffairs.apk` (release build signed with a generated keystore, or debug-signed if release is impossible). If the sandbox has no Android SDK, still deliver the full project plus `.github/workflows/build-apk.yml` (GitHub Actions: setup-java 17, android-actions/setup-android, `./gradlew assembleRelease` or `assembleDebug`, upload the APK as an artifact) and exact manual build steps.
3. Keep keystore password/alias in a short `SIGNING.md` (do not lose it: future updates need the same key).

## App identity
- App name: `StoreAffairs` · suggested applicationId: `com.fathallah.storeaffairs` · versionName 1.0 · minSdk 24 · targetSdk 34+ · Kotlin.
- Launcher icon: use `icon-1024-adaptive-foreground.png` as adaptive-icon foreground over a solid white background (`#FFFFFF`), and `icon-512.png` for the legacy/round icon. Generate all mipmap densities.
- Status bar / theme color: `#10243A`. Support RTL (`android:supportsRtl="true"`). Both portrait and landscape allowed. Keep screen on while the app is in the foreground (`FLAG_KEEP_SCREEN_ON`).
- Permissions: only `CAMERA`. Do NOT request INTERNET or storage permissions.

## WebView requirements (all mandatory)
1. **Load via WebViewAssetLoader** (androidx.webkit): put `index.html` in `assets/` and load `https://appassets.androidplatform.net/assets/index.html`. Reason: `getUserMedia` (barcode camera) and `localStorage` need a secure, stable origin; `file://` breaks them. Block all other network requests (offline app).
2. Enable: JavaScript, DOM storage (`localStorage` carries the saved count — it must survive app restarts), `mediaPlaybackRequiresUserGesture = false` (beep sounds via WebAudio), `allowFileAccess = false`.
3. **Camera:** request runtime `CAMERA` permission, then in `WebChromeClient.onPermissionRequest` grant `RESOURCE_VIDEO_CAPTURE` for the asset origin only. Rear camera barcode scanning must work (page uses `getUserMedia` with facingMode environment, torch and zoom constraints).
4. **File upload:** implement `WebChromeClient.onShowFileChooser` (system picker via `ACTION_GET_CONTENT`, `*/*`, multiple not needed). The app uploads an `.Xls`/`.xlsx` sheet and a `.json` backup through `<input type=file>`. Make sure files with the odd `.Xls` extension are selectable (do not filter by MIME).
5. **Native bridges** (the page already feature-detects these; names and signatures must match exactly; register with `addJavascriptInterface`, annotate methods `@JavascriptInterface`):
   - `AndroidShare.shareFile(name: String, mime: String, base64: String)` — decode base64 to a file in `cacheDir/shared/`, expose it through `FileProvider`, open the system share sheet with `ACTION_SEND` (`EXTRA_STREAM`, `FLAG_GRANT_READ_URI_PERMISSION`). Used for Excel `.xlsx` exports and `.json` backups.
   - `AndroidShare.shareText(text: String)` — `ACTION_SEND` with `text/plain` (WhatsApp summary).
   - `AndroidPrint.print()` — print the current WebView content: `PrintManager.print("StoreAffairs report", webView.createPrintDocumentAdapter("StoreAffairs report"), PrintAttributes.Builder().build())`. The page has `@media print` rules that print only its report element, so no extra work is needed. Run on the UI thread.
   - Plain `<a download>` / blob downloads are NOT used on Android (the bridge replaces them).
6. **Back button:** the page exposes `window.appBack()` which returns `true` if it handled the press (closes a dialog / camera / shows the "return to main menu" warning). In `onBackPressedDispatcher`: call `webView.evaluateJavascript("window.appBack ? window.appBack() : false")`; if the result is `"true"` do nothing, otherwise `finish()` the activity.
7. Keep the WebView state on rotation (`android:configChanges="orientation|screenSize|keyboardHidden"`) so the page is not reloaded mid-count.
8. Long-press / text selection can stay default. Disable WebView debugging in release.
9. Cut the keyboard-resize problem: `android:windowSoftInputMode="adjustResize"`.

## Acceptance test (run on a device/emulator, airplane mode ON)
- App opens, shows the StoreAffairs logo screen, no network errors.
- Choose the sheet `2645.Xls` (provided by the user separately) → ~126 items appear.
- Type last 4 digits of a barcode → quantity dialog opens; save → counter updates.
- Tap 📷 → camera strip opens and scans a barcode; beep plays.
- Close and reopen the app → the saved count is still there ("متابعة" / restored automatically).
- Hub → Excel export / WhatsApp summary / backup → Android share sheet appears with the file/text.
- "✔ إنهاء" → confirm → Android print dialog opens with the report (save as PDF works).
- Hub → restore backup (`.json`) works through the file picker.
- Hardware Back: closes dialogs first, then asks the warning, then exits from the main menu.

## Files in this package
- `index.html` — the app (copy into `app/src/main/assets/index.html` unchanged)
- `icon-1024-adaptive-foreground.png`, `icon-512.png` — launcher icons
