# Gameplan: building our launcher on Lawnchair 16

Working notes for turning this fork of [LawnchairLauncher/lawnchair](https://github.com/LawnchairLauncher/lawnchair)
into our own launcher. Items are ordered by priority: the ones near the top are cheap now and painful later.

## Where things stand

- Fork: `HonkBonkus/lawnchair`, branch `16-dev` (Lawnchair 16, based on Android 16 Launcher3).
- Upstream was 23 commits ahead of the fork. Those commits are now merged (fast-forward, no conflicts) into
  `claude/zealous-thompson-il4z7t`. The fork's own `16-dev` is still behind until that branch is merged
  or GitHub's **Sync fork** button is used.
- The fork had no `upstream` remote configured locally. For future syncs:
  `git remote add upstream https://github.com/LawnchairLauncher/lawnchair.git`
- Submodule `platform_frameworks_libs_systemui` still points at Lawnchair's repo. That's fine unless we need
  to change it.

### Codebase map

| Directory | What it is | Size | Should we edit it? |
|---|---|---|---|
| `src/` | AOSP Launcher3 core (workspace, drag/drop, folders, all apps) | ~600 files | Rarely. Every edit here is a future merge conflict |
| `quickstep/` | Recents/gesture navigation (AOSP) | ~680 files | Rarely |
| `systemUI/`, `wmshell/` | Ported SystemUI / WindowManager shell libs | ~1,900 files | Almost never |
| `platform_frameworks_libs_systemui/` | Submodule: icon loader, animation, search UI libs | ~160 files | Almost never |
| `lawnchair/` | Lawnchair's own layer (`app.lawnchair.*`): prefs, theming, icons, search, smartspace, gestures, Compose settings UI | ~510 files | Yes. This is how Lawnchair itself customizes Launcher3 |
| `compose/` | Compose facade for Launcher3 | small | Sometimes |

Key entry points:
- `lawnchair/src/app/lawnchair/LawnchairLauncher.kt`: the launcher activity (extends Quickstep's launcher)
- `lawnchair/src/app/lawnchair/LawnchairApp.kt`: application class
- `lawnchair/src/app/lawnchair/preferences2/PreferenceManager2.kt`: DataStore-backed settings
- `lawnchair/src/app/lawnchair/ui/preferences/`: Compose settings screens
- `lawnchair/res/values/config.xml`: swaps AOSP classes for Lawnchair ones (app filter, icon caching, overlays…)
- `build.gradle`: flavors are `lawn` × `withQuickstep` × `github|nightly|play`

## Priority 1: decisions to make before writing any feature code

1. **Name and identity.** Pick the app name and a unique `applicationId` (for example `com.honkbonkus.<name>`).
   Right now the build produces `app.lawnchair`, which means:
   - it installs *over* real Lawnchair on any phone that has it, and they can't coexist;
   - it can never be published under that ID (Play Store, F-Droid).
   Changing it later wipes users' layouts and settings, so this goes first.
2. **What makes it different?** Write 3–5 sentences on what this launcher does that Lawnchair doesn't.
   That decides how far we drift from upstream, and therefore how expensive syncing stays.
3. **Branch strategy.** Recommended:
   - `16-dev` stays a clean mirror of upstream (only ever fast-forwarded from `upstream/16-dev`).
   - A `main` (or `dev`) branch carries our launcher. Upstream is merged into it regularly (merge, not rebase).
   - Feature branches come off `main`.
4. **Distribution target.** Sideload/GitHub only, F-Droid, or Play Store? Play forces stricter rules
   (Lawnchair already has a `play` flavor that disables things). This decides which flavors we keep.

## Priority 2: rebrand and make the fork safe to install

These are mechanical but touch "phone home" behavior, so they must be done before handing a build to anyone.

1. **applicationId and app name.** In `build.gradle` (`productFlavors` and `derived_app_name` resValues),
   plus the APK output filename. Keep the Java/Kotlin package `app.lawnchair.*` for now. Renaming ~500 files
   would make every upstream merge conflict. Only the `applicationId` needs to change.
2. **Disable or retarget the in-app updater.** `ui/preferences/about/GithubService.kt` and
   `NightlyBuildsRepository.kt` fetch releases and commits from `LawnchairLauncher/lawnchair`. A nightly build
   of our fork would offer to "update" itself to upstream Lawnchair. Point it at our repo, or turn it off.
3. **Announcements feed.** `ui/preferences/data/liveinfo/LiveInformationRequest.kt` pulls announcements from
   `https://lawnchair.app/`. Disable it or host our own.
4. **About screen, links and bug reports.** `ui/preferences/about/` (team list, Discord/Telegram, commit links),
   `bugreport/`, and strings that point to Lawnchair's community.
5. **Icons and visual identity.** Launcher icons are in `lawnchair/res/mipmap-*` and `nightly/res/mipmap-*`.
6. **Licensing.** The project is Apache 2.0, so we can ship it under our own name, but we must keep
   `LICENSE.txt` and existing copyright headers, and add our own notice. Don't use the Lawnchair name or logo
   for our product.
7. **CI.** `.github/workflows/` are mostly gated on `github.repository_owner == 'LawnchairLauncher'`, so they're
   harmless on the fork, but `ci.yml` will still build on every push to `*-dev`. Decide which workflows to keep;
   remove the Crowdin, Telegram and stale-issue ones. Add our own signing secrets when we want signed releases.
8. **Release signing.** Generate *our own* keystore (`keystore.properties` + `.jks`, never committed).
   Without it, release builds fall back to the debug key.

## Priority 3: development workflow

1. **Local build.** JDK 21 and the Android SDK. Run `git submodule update --init`, then
   `./gradlew assembleLawnWithQuickstepGithubDebug`. The output lands in `build/outputs/apk/`.
   Debug builds get a `.debug` suffix so they can sit next to a release build.
2. **Testing on a device.** A launcher has to be tested on real hardware or an emulator; set it as the default home app.
   Recents/QuickSwitch integration needs root (Android 15–16). Without it, the launcher still works but
   gesture recents stay with the system launcher.
3. **Style.** `./gradlew spotlessCheck` / `spotlessApply` (CI enforces it).
4. **Upstream sync cadence.** Every 1–2 weeks: fast-forward `16-dev` from upstream, merge into `main`, build, smoke test.
   Small, frequent merges are far easier than big ones.
5. **Where to put our code.** New features go in a new package under `lawnchair/src/` (for example
   `app/lawnchair/<ourname>/` or our own top-level package), hooked in through the existing extension
   points (`config.xml` class overrides, `PreferenceManager2`, Compose preference destinations). Touch `src/`
   and `quickstep/` only when there's no other way, and mark those edits with a comment such as
   `// <OurName>:` so they're easy to find during merges, the same way Lawnchair uses `Lawnchair-TODO`.

## Priority 4: product work

Pending the "what makes it different" decision (Priority 1.2). A reasonable sequence:

1. Pick one signature feature and build it end to end (preference → behavior → settings UI) to learn the codebase.
2. Trim settings/features we don't want (hiding them is safer than deleting them, to keep merges clean).
3. Default layout and theming: `res/xml/default_workspace_*.xml`, the `lawnchair/src/app/lawnchair/theme/` defaults.
4. Distribution: GitHub Releases first (simplest), then F-Droid/IzzyOnDroid or Play if wanted.

## Open questions for us

- [ ] App name
- [ ] Package ID (`applicationId`)
- [ ] The core idea: what's different from Lawnchair?
- [ ] Distribution channels
- [ ] Keep the in-app updater (pointed at our releases) or remove it?
