# 32 — Mobile delivery: accounts, builds, stores and device testing

[10-mobile](10-mobile.md) covers the code of the mobile app. This file covers everything
around the code:

- the accounts and identifiers the app needs before its first build;
- how the mobile package is laid out in git for the build tools;
- how a build gets its version, where it builds, and how it is signed;
- how a build reaches testers and the public, on each store;
- the store listing, kept as files in the repository;
- the device-flow tests on simulators and emulators;
- real-device measurement;
- recording help and training videos from the real app;
- what to watch after a release.

Every rule here comes from a failure that cost a release, a review cycle or a day of
diagnosis. The rule is stated first; the reason follows.

---

## 0. Reference stack and swap points

| Concern | Reference implementation | Swap point? |
|---|---|---|
| App framework | React Native on Expo, with a development client | Yes |
| Build tool | EAS Build, run **locally** on self-hosted runners (`eas build --local`) | Yes — the hosted EAS service or plain Xcode/Gradle work too |
| Upload to the Apple store | EAS Submit (to TestFlight) | Yes |
| Public release and listings | fastlane (`deliver` for iOS, `supply` for Android) | Yes |
| Device-flow tests | Maestro | Yes |
| Mobile-web tests | Playwright against the Expo web build | Yes |
| Real iPhone over Wi-Fi | `pymobiledevice3` | Yes |
| iOS simulator UI driver for the budget test | `idb` | Yes |
| Android device and emulator control | `adb`, `screenrecord`, `uiautomator` | No real alternative |
| Pipeline | GitHub Actions, self-hosted macOS and Linux runners | Yes |
| Crash and error reporting | Sentry, with source maps and native debug files uploaded per build | Yes |

**The parts that are NOT swap points:**

1. Every build has a version string and a build number that are higher than every build
   before it, set by the pipeline (§ 3).
2. Every write to a store is a deliberate, manual run with its write flag set on purpose.
   No push, schedule or default value writes to a store (§ 8).
3. The store listing is files in the repository (§ 9).
4. Device flows run against the real local backend, never against faked responses
   ([13-testing-e2e](13-testing-e2e.md) § 1.1).
5. Work on a repeating path is measured on a real device before it counts as done
   ([10-mobile](10-mobile.md) § 5, § 11 here).

---

## 1. Accounts and identifiers

Set these up before the first build. Each one is outside the repository, so write down in
the repository who owns each account and where its keys are kept.

| Account | What it gives | Key or credential |
|---|---|---|
| Expo / EAS | Build credentials store, push service, project id | `EXPO_TOKEN` as a CI secret |
| Apple Developer Program team | Bundle ids, capabilities, signing certificates, provisioning profiles | Team owner signs in interactively |
| App Store Connect | The app record, TestFlight, App Store versions, App Review | An API key (`.p8`, key id, issuer id) with the App Manager role |
| Google Play Console | The app record, tracks, listing, review | A service-account JSON key with release permissions |
| Crash reporting | One project for the mobile app, separate from the web project | An auth token for uploads, as a CI secret |
| Object storage (optional) | A direct-download location for an installable Android APK | A pipeline role through federated credentials ([28-supply-chain-and-build-inputs](28-supply-chain-and-build-inputs.md)) |

### 1.1 Identifiers are permanent

1. **Choose the bundle id and the Android package name once.** Neither store lets you
   change them. A new id is a new app, with no users and no reviews. Use a reverse-DNS
   name on a domain the project controls.
2. **Do not put an organisation's acronym, a government-like name or a national symbol in
   the app name or icon** unless you can prove the right to use it. A store reviewer reads
   these as impersonation. A rejection for impersonation can stop all distribution until
   the appeal is decided.
3. **The display name can differ per platform.** If one store forces a rename, pin the
   other platform to its shipped name in the platform config, so users on that platform do
   not see the app change name under them.

### 1.2 Register extensions and capabilities before the build that needs them

A share extension, a widget extension, a Live Activity or an App Group each needs its own
identifier and capability in the Apple developer portal:

1. Register each extension id as `<bundle id>.<ExtensionName>`. Many config plugins
   default to a different suffix; set the id explicitly in the app config so it matches
   the portal.
2. Register the App Group (`group.<bundle id>`) on the main app id **and** on every
   extension that shares data with it.
3. **Every capability change invalidates the existing provisioning profiles.** The next
   CI build fails at signing.
4. **CI cannot repair profiles.** CI builds run `--non-interactive`. A new profile needs
   an interactive `eas credentials` run with the team owner's Apple sign-in.
5. **An extension's profile can only be made from a branch whose app config contains that
   extension.** Run the interactive credentials step on the branch that adds it, before
   that branch merges.

### 1.3 Create the store records with a script

Use the App Store Connect API to register the bundle id and create the app record, and
print the numeric app id that the submit profile needs. Make the script idempotent, so a
re-run changes nothing. Keep the `.p8` and the script's environment file git-ignored.

---

## 2. The mobile package in git

```
mobile/
  app.config.js          the app config, computed from environment variables
  app.config.test.js     tests for the config itself
  eas.json               build profiles and submit profiles
  plugins/               local config plugins (build-time changes to native projects)
  modules/               local native modules, each with its own podspec / Gradle file
  targets/               iOS extension targets (widgets, Live Activities)
  fastlane/
    Fastfile             store lanes
    Appfile
    metadata/<platform>/<locale>/   listing copy, screenshots, graphics
  maestro/               device flows, sub-flows, fixtures, host-side helper scripts
  scripts/               build wrappers, store readers, device harnesses
  screenshots/           the store screenshot sets, per device class
  Gemfile                fastlane and its Ruby dependencies, pinned
```

### 2.1 The app config is code, and it has tests

- `app.config.js` reads the version, the build number and any build-variant flag from
  environment variables, with safe defaults for a developer's machine.
- Test the config: the identifiers, the extension ids, the plugin list, and that a store
  build never carries a test-only setting.
- A **release test build** — a release-mode build that may talk to the local backend over
  plain HTTP for device tests — is a separate flag that CI never sets. Assert that in a
  test.

### 2.2 Build profiles

Keep three profiles, no more until there is a reason:

| Profile | Use |
|---|---|
| `development` | A development client for a simulator or a developer's device. Internal distribution. |
| `production` | The store build. Android builds an app bundle (AAB). |
| `production-apk` | Extends `production`; builds an installable APK for direct download or sideloading. |

Set `appVersionSource` to `local`, so the version comes from the app config (§ 3) and not
from a counter held by the build service.

### 2.3 Generated client files and builds that pack from git

[06-api-contract-and-codegen](06-api-contract-and-codegen.md) says generated files are
never committed. A build tool that packs the project **from the git tree** (EAS local
builds do this) does not see a git-ignored file, and the app builds with no API client.
Pick one, and write the choice down:

1. **Generate inside the build**, in the build tool's pre-install hook. Preferred when the
   generator can run there.
2. **Commit the mobile copies only**, regenerate them in the same commit as the contract
   change, and run a CI check that fails a pull request whose committed copies differ
   from a fresh generation.

Option 2 is the reference implementation. The web and server copies stay uncommitted.

### 2.4 Native modules must be linked, and nothing says so when they are not

A local native module with no podspec builds with no error on iOS and is missing at run
time: `requireNativeModule` throws `Cannot find native module`. Add a test that walks
`modules/` and fails when a module folder has no podspec (iOS) or no Gradle file
(Android).

### 2.5 Adding a native dependency

- Add it with `expo install <package>`, which picks the version that matches the SDK.
- `expo install --check` answers 0 for a package that is only a hoisted transitive, for a
  misspelled name, and when it is offline. Wrap it in a script that also checks the name
  is in `mobile/package.json` and refuses an offline answer.
- The dependency rule in [28-supply-chain-and-build-inputs](28-supply-chain-and-build-inputs.md)
  applies to `mobile/package.json`, the `Gemfile`, every config plugin and every
  `Podfile` entry.

---

## 3. Build identity

**Rule: the pipeline sets both the version string and the build number on every build,
and both are higher than on any build before.**

| Value | iOS field | Android field | Reference source |
|---|---|---|---|
| Version string | `CFBundleShortVersionString` | `versionName` | `1.<CI run number>.0` |
| Build number | `CFBundleVersion` | `versionCode` | Unix epoch seconds at build time |

Reasons:

- The Apple upload rejects a second build with the same **version string**, even with a
  new build number ("You've already submitted this version of the app"). A unique build
  number alone is not enough.
- The run number in the minor segment makes every build a new version and lets anyone read
  the build run from the version.
- Epoch seconds is monotonic across machines and needs no stored counter. Play Console
  names a release after the version code by default, which is unreadable with epoch
  seconds, so pass a readable release name to the upload.

---

## 4. Builds

### 4.1 Where builds run

| Platform | Runner | Why |
|---|---|---|
| iOS | Self-hosted macOS ARM64 runner with Xcode | Signing, simulators and Xcode need macOS |
| Android | A dedicated self-hosted Linux runner with the Android SDK, NDK and a JDK | One box with the SDK; keep other work off it |

1. **Give the Android runner its own labels** and remove the generic pool label, so no
   pool job is scheduled onto the one box Android builds need. Runner labels are stored
   on the CI host, not in the repository: a re-registered runner loses them. Write the
   label command in the runner provisioning document.
2. **One job builds and publishes.** Do not pass a build between jobs as a CI artifact.
   Artifact storage is a shared account quota; when it is full, every build succeeds and
   every publish fails.
3. **Start a mobile build by hand, once per batch of merges.** A push trigger builds once
   per merge and fills the build queue and the runner disk. If a push trigger is kept,
   give it a path filter that excludes flows, listing files and docs, and a guard test
   that pins that filter.
4. **Use `concurrency` with cancel-in-progress for builds**, and queue (no cancel) for
   store releases — a cancelled release can leave a half-made version in the store.

### 4.2 Runner hygiene for local builds

Local EAS builds leave state on the runner. Clean it in the workflow, before the build:

- **Prune old build working directories** (`$TMPDIR/eas-build-local-nodejs/<uuid>`,
  several GB each) older than an hour. Do not keep build directories "for forensics"; dump
  the Xcode log on failure instead.
- **Delete the Metro cache in `$TMPDIR`** (`metro-*`, `haste-map-*`) before each build. A
  cached entry can hold the path of a build directory that the prune removed, and the
  bundle step then fails with `Unable to resolve module`.
- **Remove stale build keychains before and after each iOS build.** A cancelled build
  leaves a locked temporary keychain in the user's search list. `codesign` meets it, has
  no screen to ask for the password, and fails with `errSecInternalComponent`.
- **Remove secrets and build outputs from the workspace** in an `if: always()` step. A
  self-hosted runner keeps its workspace between jobs.

### 4.3 Crash-report uploads

- The release build uploads JS source maps and native debug files (dSYMs, the R8 mapping)
  to the mobile crash-report project.
- In CI the upload token is required, and a failed upload fails the build.
- If the upload service is unreachable, a wrapper may build again with the upload off so
  the release can continue — but it must set a flag and **fail the job at the end**. A
  release with no maps shows minified stack traces; that is a failure to report, not to
  hide. Pin the wrapper with a test.

---

## 5. Credentials and signing

- Signing certificates and provisioning profiles live in the build service's credential
  store. CI reads them with `EXPO_TOKEN`; it never creates or changes them.
- Store API keys are CI secrets. The workflow writes each key to a git-ignored file with
  mode `600` **after** the build and before the step that needs it, and deletes it in an
  `if: always()` step.
- A developer's local copy of a key lives in a git-ignored path under `mobile/scripts/`.
  Add every such path to `.gitignore` before the first key exists.
- Use the **App Store Connect API key** for every Apple call from CI. Never a person's
  Apple ID and app-specific password.

---

## 6. iOS distribution

```
build (IPA) → eas submit → TestFlight → fastlane deliver → App Review → App Store
```

1. **TestFlight**: the build workflow builds the IPA and runs `eas submit`. That uploads
   the build to App Store Connect and stops. Testers in TestFlight get it after Apple's
   processing.
2. **App Store**: `eas submit` cannot create the App Store version, attach a build, write
   the release notes or submit for review. A separate manual workflow runs a fastlane lane
   that does all four.

### 6.1 The release lane

Inputs: the release notes (required), the version (blank selects the newest TestFlight
build), the build number (blank selects the newest build of that version), and a
submit-for-review flag.

What the lane does, in order:

1. Signs in with the API key.
2. Resolves the version and build from TestFlight when they are blank.
3. Reads the listing locales from App Store Connect, rather than assuming them, and writes
   the release notes for each locale.
4. Uploads the version with the build attached and `submit_for_review: false`.
5. **Reads the release notes back** and fails if any locale is empty. App Store Connect
   reports a missing note as a state error on the version (`is not in valid state`), not
   as a missing file.
6. Submits for review in a second call, if the flag is set.

Traps the lane must guard against, each with a test:

- **`metadata_path` must be absolute.** fastlane runs the lane body in `fastlane/` and
  runs `deliver` in the project folder; a relative path names two different folders.
- **Every `deliver` call needs `metadata_path`**, including the submit call.
  `deliver` validates the folder even when it skips metadata, and the default folder
  holds the Android listing, which it rejects.
- **Skip screenshots** unless the lane manages them. `deliver` deletes every screenshot
  of a locale it uploads, so an empty local folder clears the listing.
- Set `force: true` (no terminal for the HTML preview on a runner) and the UTF-8 locale
  environment variables.

Decide and write down whether an approved version releases automatically and whether it
uses a phased release. There is no undo for a released version.

---

## 7. Android distribution

```
build (AAB) → upload to a track → Play review → testers or production
build (APK) → object storage → direct-download link
```

### 7.1 Tracks and testers

| Track | Testers | Can it be automated? |
|---|---|---|
| Internal testing | Email list only, up to 100 | No. The tester API accepts Google Groups only, and this track does not accept groups |
| Closed testing | Email lists or Google Groups | Yes, through a group the pipeline manages |
| Production | Everyone | — |

- To automate tester enrolment, use closed testing with a Workspace or Cloud Identity
  group, and add members through the groups API with a service account that is a manager
  of that group. A consumer group cannot be managed by API.
- A new tester still has to open the opt-in link once. Adding them to the group does not
  remove that step.
- The email a user signs in to the app with is often not the Google account on their
  phone. An enrolment form asks for the Play account specifically.

### 7.2 A green upload is not a release

The upload succeeds when Play **accepts** the bundle. Play then reviews it, and testers
keep the previous release until the review passes. Write a script, and a manual workflow
that runs it, that reads each track and prints one line per release: the name, the state
in plain words and the version code with its build time. Read the store state from that
script, never from a note or a memory.

### 7.3 Direct-download APK

An installable APK in object storage, behind a stable download link, gives testers a
build with no store review. Keep two links: production and beta. Publish with federated
pipeline credentials from the same job that builds the APK (§ 4.1, rule 2). Refuse a
publish request that has no build in the same run.

---

## 8. Store writes are deliberate

**Rule: no store write happens unless a person starts a manual run and sets its write
flag on purpose. Every write flag defaults to off.**

Reason: on Play, every upload to a track and every listing edit is a new pending change,
and Play **cancels the review in progress and starts it again**. A pipeline that uploads on
every run means no release is ever approved, and users stay on old versions. On Apple, a
submission sends the build to App Review and a release cannot be withdrawn.

Enforce it with guard tests over the workflow files ([12-testing-integration-and-guards](12-testing-integration-and-guards.md)):

- [ ] The store-release workflows have only a manual trigger — no `push`, `schedule` or
      `workflow_run`.
- [ ] Every store-write input (`submit`, `publish_apk`, `submit_for_review` where review
      is not the default) defaults to `false`.
- [ ] A fastlane lane that writes to a store can be switched off with a single guard
      function, and a test pins whether it is on or off.

Upload to a store **once per release**, not once per build.

---

## 9. Store listings as code

- Keep the listing copy, icon, feature graphic and screenshots under
  `fastlane/metadata/<platform>/<locale>/`: title, short and full description, video URL,
  changelog, and screenshot folders per device class (phone, 7-inch, 10-inch tablet).
- `fetch` lanes pull the store's current state into those files once, after the app record
  exists. `validate` lanes check the files against the store with no write. Both are safe
  to run at any time.
- The lane that writes the listing follows § 8: manual, flag off by default.
- **Make the screenshots with device flows, not by hand.** A flow that signs in to a
  seeded local backend and takes a screenshot at each screen gives the same set every
  time, and a UI change is one re-run. Show realistic, varied seed data (§ 14.3).
- A rejection is then diagnosable from a diff of the listing files.

### 9.1 Store review

Read each store's policy before the first submission, and keep a short checklist in the
repository:

- name and icon: no implied affiliation (§ 1.1);
- background location: an in-app disclosure screen before the system prompt, and a
  stated reason in the permission strings;
- a privacy-policy link in the app and in the listing;
- export-compliance and advertising-id answers set in the submit lane, so the submission
  is the same every time.

---

## 10. Device-flow tests

Device flows drive a real build of the app on a simulator or emulator, against the real
local backend ([14-local-dev-and-simulators](14-local-dev-and-simulators.md)). The rules of
[13-testing-e2e](13-testing-e2e.md) apply: no faked API responses, select by test id,
seed through the API, and every flow independent.

### 10.1 Layout of the flows

```
maestro/
  config.yaml              the ordered list of flows a full run includes
  <feature>.yaml           a flow that runs on both platforms
  <feature>-ios.yaml       a flow for one platform only
  <feature>-android.yaml
  login.yaml               shared sub-flows, run with runFlow:
  reset-server.yaml
  open-drawer.yaml
  fixtures/                images, PDFs, a QR poster
  scripts/                 host-side helpers: seed, sign in as admin, check a result through the API
```

- **Every flow starts with the reset sub-flow**, which calls the local reset endpoint so
  the flow begins from the same seeded data. A flow never depends on what the previous
  flow left.
- **The sign-in sub-flow is gated** on the email field being visible, so it does nothing
  when tokens are already stored.
- **A flow that needs a special device set-up is not in `config.yaml`** — for example a
  virtual camera scene for QR scanning, or files pushed to the emulator first. Each such
  exclusion has a comment in `config.yaml` that says why and points at the steps.
- **Check a result through the API**, from a host-side script, as well as on the screen.

### 10.2 Running

- **Run flows one at a time against one device.** A tool that runs a directory in
  parallel on one device makes the flows fight over the one app install, and the failures
  have nothing to do with the code.
- Wrap the flow tool in one script (`run-<tool>.sh <ios|android> <path>`) that keeps the
  tool's exit code and reports the run to the test-run log.
- Let the bundler finish its first bundle before the first flow. A cold bundle makes the
  first launch show a progress screen, and the flow fails on an unrelated assertion.
- The local reset can take longer than the flow tool's own HTTP timeout on a slow machine.
  Call the reset from the host first, wait for its answer, then start the flow.
- Failed runs leave screenshots and logs in the tool's output folder. Attach them to the
  report.

### 10.3 Pointing the device at the local backend

**iOS simulator**: `localhost` is the host. Nothing to do.

**Android emulator**: map each port the app calls with `adb reverse`:

```bash
adb reverse tcp:<bundler port> tcp:<bundler port>
adb reverse tcp:<api port> tcp:<api port>
adb reverse tcp:<socket port> tcp:<socket port>
adb reverse tcp:<identity emulator port> tcp:<identity emulator port>
adb reverse tcp:<object storage port> tcp:<object storage port>
```

Traps:

- **The object storage port is easy to miss.** Without it, sync and the metadata calls
  answer, and only the file bytes fail. The screen shows the app's own placeholder and the
  flow fails on an image assertion, not on a connection error.
- **A presigned URL carries the storage server's own port.** When the local stack runs on
  shifted ports, reverse each shifted port to itself as well.
- **The emulator loads JS from the host's bundler port through `10.0.2.2`**, and
  `adb reverse` does not change that. If another bundler holds that port, the app runs the
  other bundler's code with no error. Build the debug APK with the bundler port you use
  (`-PreactNativeDevServerPort=<port>`).
- Host-side helper scripts run on the host, so `adb reverse` does not move them. Pass
  them the shifted ports as flow environment variables.

### 10.4 Environment selection

- The app's default environment is production. A flow that launches with cleared state
  must switch to the local environment before it signs in, or it signs in against the real
  identity provider and fails.
- Hide the environment selector in a store build behind a gesture that the flow performs
  (for example, three taps on the build label). Do **not** add a test-only flag to reach
  it: the flow then tests a path that users do not have.

### 10.5 Emulator set-up

- Choose the emulator profile for the screen size the layout tests need, and write the
  profile down.
- **Use hardware acceleration.** A software-rendered emulator is too slow to give a
  meaningful screenshot and hides timing defects. On a Linux box where the user cannot
  open `/dev/kvm`, run the emulator in a container with the KVM device passed in and host
  networking.
- A plain AOSP system image is about half the size of a Google APIs image. It has no Play
  services, so push-token registration fails there. Make push registration treat that
  failure as "no push" and continue, and the app runs on the smaller image.
- On recent Android versions, the on-screen keyboard can cover the sign-in fields. Turn
  the hardware keyboard off in the AVD config, and add a flow that asserts the focused
  field stays above the keyboard.
- `uiautomator dump` fails with "could not get idle state" on a screen that never stops
  changing (a map, a timer). Read element bounds from the flow tool's hierarchy command
  instead.
- A touch within about 80 px of a side edge is Android's Back gesture.

### 10.6 iOS simulator set-up

- A flow tool's iOS driver can lag behind the newest simulator OS. Pin the simulator OS
  version that the tool supports, or drive that test with `idb`.
- Add photos with `xcrun simctl addmedia` and set the location with
  `xcrun simctl location`.

### 10.7 What a flow cannot reach

Some states need a held touch, the system camera app or a system dialog the flow tool
cannot drive. Take those screenshots with a small script that uses `adb` input commands
and the element bounds from the hierarchy. List each such state, and the script that
covers it, in the flows README.

### 10.8 The mobile-web suite

Run the app's Expo web build under Playwright for flows that use no real native API. Give
it its own config and npm script, so the slow cold bundle does not slow the main suite.
Mount a typed test bridge under a test-only build flag for GPS, permissions and photos.
Keep a table of each mobile feature, its web spec and its device flow, so a gap is
visible.

---

## 11. Real-device measurement

A simulator does not apply the background CPU limit, thermal pressure or battery
pressure. A defect from those only reproduces on a real device ([10-mobile](10-mobile.md)
§ 5).

### 11.1 The device harness

Keep a harness in `mobile/scripts/device-walk/` that, in one command:

1. connects to the phone over Wi-Fi;
2. starts the system-log capture;
3. replays a recorded GPX route as the phone's location;
4. samples the app's CPU to a CSV file;
5. after the run, pulls the crash reports and the app's own log.

Add a route drawer (a page that exports GPX) and a filter for the system log. Report mean
CPU, any resource-warning or suspension line, and the longest gap in the recorded data.

Facts for an iPhone driven from a non-Mac machine with `pymobiledevice3`:

- Pair once over the cable, and enable Wi-Fi connections. Over Wi-Fi the phone accepts the
  remote-pairing tunnel, not the USB proxy.
- Read the pair record by device UDID. A match by Wi-Fi MAC fails when the phone uses a
  private Wi-Fi address.
- Crash reports and the app container need the tunnel.
- Mount the developer disk image again after each phone restart.

### 11.2 The simulator background-budget test

On the simulator, a test can still measure the **cause** — work the app does while it is
not active — even though the simulator does not enforce the limit. Make it a separate
script, not part of the flow list:

1. sign in to the local stack with a debug quick-login;
2. start the session and play a walk;
3. send the app to the background;
4. sample CPU and a probe that counts renders and per-event work while not active;
5. assert each count against a written budget.

The test guards the ports it uses and starts the local stack itself.

### 11.3 Simulator location replay

A script that moves the simulator location along a route with `xcrun simctl location`, at
walking pace, tests the location-task and geofence logic without a walk. It does not
reproduce real-device suspension; say that in its header.

---

## 12. Runner and device placement

- **One device-test suite per machine at a time.** A suite stops processes on the ports it
  needs; a second suite on the same box loses its servers, and its failures read as code
  regressions.
- If the macOS box is also a CI runner, it runs the iOS jobs and the iOS flows only. Run
  web end-to-end suites elsewhere.
- Write each runner's label set and the tools it must have in a runner provisioning
  document. A workflow cannot install system tools on a self-hosted box.

---

## 13. Observability after release

- One crash-report release per build, named from the version and build number, with
  uploaded maps (§ 4.3).
- An in-app diagnostics screen that support can read from: sync state, queue sizes, cache
  size, the location-task log ([10-mobile](10-mobile.md) § 3).
- The store-state readers of § 7.2, so "do testers have it?" has an answer from a
  workflow run.
- A version check at startup against a minimum supported version
  ([10-mobile](10-mobile.md) § 6).

---

## 14. Help and training videos from the real app

Record help videos from the real web app and the real mobile app, driven by a script, so a
UI change is a re-record and not a re-shoot.

### 14.1 The tool

- One YAML script per video in `help-videos/scripts/<id>.yaml`: a list of steps, each a
  line of narration plus the browser and phone actions that go with it, and the data the
  video needs.
- One command per stage:

| Command | What it does |
|---|---|
| `check` | Parses every script and lists the problems |
| `seed <id>` | Makes the script's data and nothing else |
| `voice <id>` | Makes the narration with a free local voice |
| `record <id>` | Records the browser and the phone, with no voice |
| `compose <id>` | Fits the recording to the voice and draws the annotations |
| `make <id>` | voice, record, compose: the draft |
| `approve <id> --note "..."` | Records the owner's approval of the draft, in the owner's words |
| `final <id>` | The paid voice, fitted to the same recording |
| `uses <testid>` | Lists the scripts that use a control, so you re-record those after a UI change |

- **The recording does not depend on the voice.** The recorder marks the start, end and
  settle time of each action and keeps a short still tail after it. Compose plays each
  action on its word, cuts the tails to fit, and draws circles and highlights from
  rectangles measured during the take. A reworded line with the same actions needs no new
  take.
- **The paid voice runs only after approval.** `final` refuses with no approval, and
  when the words changed after the approval. It prints the number of characters it will
  send. Cache every clip by its full request, and commit the paid clips (through Git LFS),
  so a re-take of an approved script costs nothing.
- **One lossy encode.** Capture as PNG frames, keep every intermediate file lossless, and
  encode once at the end with screen-tuned settings and a bitrate ceiling.
- Keep a rules file (pace, navigation, annotations, data on screen) and a voice file
  (voice settings and delivery) next to the scripts. Read both before a take. Add each new
  rule from the owner with its date, and enforce a rule in the tool where you can.

### 14.2 Recording the phone

- Android `screenrecord` stops at 180 s. Record in pieces of up to 170 s, and line the
  pieces up by an end mark: the recording starts late but stops at once.
- Set the status bar to a fixed time, full battery and Wi-Fi only. Some vendor builds
  ignore demo mode; crop the status bar in compose for those.
- Turn "show touches" on.
- Record from a **debug build of the current code**. An older build fails with
  `Cannot find native module` on a new native module.
- Simulated GPS jitters by a few metres, as a real phone does. Keep the step length under
  the app's trace-split distance, or no trace is drawn. On a vendor build that removes test
  location providers, use a small location-provider app on the device.
- A screen read takes seconds on a real phone. Keep the place of a control that never
  moves after the first take, and re-read only when the control is not at its kept place.

### 14.3 Data on screen

- Record from the local stack, reset to a known state, with a mirror of the staging map
  data where the video needs real geography.
- Show every feature as if it has been in use for a while: many varied, realistic items
  with dates spread back in time. Never one test item.
- Every email address and phone number on screen is a test value the project owns. No
  blur: show data that may be seen.
- The same rules apply to store screenshots (§ 9) and to release-email media.

### 14.4 Videos bundled in the app

If the app shows help videos offline, bundle the video files in the app and keep them in
plain git (not LFS), because the build packs from git (§ 2.3). Record that choice in a
decision record, with the size budget.

---

## 15. New-project checklist

- [ ] Accounts of § 1 exist; owner and key location written down.
- [ ] Bundle id and package name chosen once, on a domain the project controls.
- [ ] Name and icon checked against store impersonation policy.
- [ ] Extensions and App Groups registered, and profiles made interactively on the branch
      that adds them.
- [ ] `app.config.js` computes version and build number from the pipeline; config has
      tests.
- [ ] Generated mobile client reaches the build (§ 2.3), with a freshness check.
- [ ] Native-module link test in place.
- [ ] iOS and Android build workflows on their own runners, with the hygiene of § 4.2.
- [ ] Crash-report uploads required in CI; a fallback fails the job at the end.
- [ ] TestFlight upload and a separate manual App Store release lane.
- [ ] Store writes manual, default off, pinned by guard tests (§ 8).
- [ ] Store-state reader workflow for Play.
- [ ] Listing files under `fastlane/metadata/`; screenshots made by flows.
- [ ] Device flows: reset, gated sign-in, one at a time, `config.yaml` with commented
      exclusions.
- [ ] `adb reverse` list written down, including object storage.
- [ ] Real-device harness, and one measured run of the main repeating path.
- [ ] Help-video tool with a rules file, if the product has help videos.
