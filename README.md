# Vitals Viewer

I built a phone app for comparing fictional patients' measurements because I wanted to check my medical-record bridge without reading raw responses, and keep the last visit visible when the connection dropped.

The app shows heart rate, temperature and blood oxygen (SpO₂) as a small chart and a list of readings for each visit. I sign in, approve access to fictional records, and choose a visit. Refresh checks for new readings. If the bridge goes offline or my session expires, I can still read the saved copy and see when it was saved.

[See the measurements screen](docs/screenshots/vitals.png). The readings shown are fictional.

## Try it

I use the included local sandbox first. It has two fictional visits with 18 measurements, a consent page and short-lived access so I can exercise sign-in and renewal without an account. It runs the actual [fhir-emr-bridge](https://github.com/jackaI1/fhir-emr-bridge) service with a simulated record server. It does not contact a real patient database.

With [Android Studio](https://developer.android.com/studio), Java 17 and [uv](https://docs.astral.sh/uv/getting-started/installation/) installed:

```sh
git clone https://github.com/jackaI1/android-vitals-viewer.git
cd android-vitals-viewer
uv venv --python 3.12
uv pip install -r sandbox/requirements.lock
uv run --no-project uvicorn sandbox.server:app --host 127.0.0.1 --port 8000 --no-access-log
```

Leave that terminal open. Open this folder in Android Studio, let it install SDK 35, and run `app` on an Android 8.0 or newer emulator. In another terminal, connect the emulator to the local service:

```sh
adb reverse tcp:8000 tcp:8000
```

Keep the app's default bridge URL and fictional patient UUID. Tap **Sign in with SMART on FHIR** (the record system's permission process), then **Approve fictional records**. Choose either visit and scroll to the three charts. The values are also written underneath each chart. Cancel or deny returns no records.

Stop the sandbox with Ctrl-C when finished, then run `adb reverse --remove tcp:8000`. [The setup guide](docs/setup.md) includes command-line builds, testing session expiry, and connecting to my existing OpenEMR demo.

## What I can check

- A visit's readings stay together using their saved visit link, even when dates overlap.
- Missing measurements stay missing. A single reading appears as a dot; several appear as a time-spaced line.
- Different units get separate charts. Fahrenheit stays Fahrenheit, and preliminary readings are labelled.
- The saved copy survives closing and reopening the app. Sign-in is required again after the app process ends.
- **Sign out and clear cache** removes all saved snapshots from this app and asks the bridge to discard the current session.

The existing OpenEMR fixture supplies heart rate and temperature but no oxygen reading. The app shows “No measurement recorded” for SpO₂ there. The included sandbox supplies all three so I can check the complete display without inventing missing values in the real bridge response.

## How the parts fit

SMART on FHIR is the permission process used to sign in and approve record access. FHIR is the shared format for medical records. My existing bridge handles that process, holds the access and refresh tokens, and returns a report containing only its fixed fictional patient records. The Android app holds the bridge's session cookie in memory; it never receives those tokens or a client secret.

I wrote the app in Kotlin with Jetpack Compose, Android's interface toolkit. Room, an Android database library, stores one validated report per bridge and patient. A successful refresh replaces the whole snapshot so removed readings do not linger. A network error or rejected report keeps the previous copy. The screen labels that copy with its saved time instead of presenting it as a fresh result.

[Design and boundaries](docs/design.md) explain the files, units, cache, and sign-in decisions. This is a local demonstration with fictional records, not a clinical tool. Its embedded sign-in page reuses the bridge's browser-bound cookie flow. A deployed app would need an external-browser handoff and additional controls before handling real records.

## Build and checks

```sh
./gradlew --no-daemon assembleDebug testDebugUnitTest lintDebug
uv run --no-project pytest -q tests
# With a running emulator and the sandbox terminal stopped:
bash scripts/instrumented.sh
```

The APK is at `app/build/outputs/apk/debug/app-debug.apk`. [GitHub Actions](https://github.com/jackaI1/android-vitals-viewer/actions) builds it, runs Android lint and the tests, and uploads the APK and reports. The checks cover parsing, Room persistence and cache isolation, HTTP session failures, Compose screens, and a real browser-consent-to-chart journey through the bridge. The sandbox job also runs the pinned bridge's existing 89 tests. There are 32 JUnit/Robolectric tests, 6 instrumented UI tests and 4 sandbox integration tests, plus the bridge’s 89 regression tests. Measured results are recorded in [verification notes](docs/verification.md).
