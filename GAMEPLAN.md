# Gameplan: Allmother

**Allmother** (a Coheed and Cambria reference) is a custom Android launcher built on this fork of
[LawnchairLauncher/lawnchair](https://github.com/LawnchairLauncher/lawnchair). It has two goals:

1. **Fix the design choices I dislike** in stock launchers and Lawnchair.
2. **Work with my ADHD, my way.** The home screen is the first thing seen on every unlock. Allmother deliberately
   avoids the traditional ADHD-management playbook; features come from my own requests, not a checklist.

Items are ordered by priority: the ones near the top are cheap now and painful later.

## Where things stand

- Fork: `HonkBonkus/lawnchair`, based on Lawnchair 16 (Android 16 Launcher3).
- Branches:
  - `16-dev`: clean mirror of upstream Lawnchair. Only ever fast-forwarded from `upstream/16-dev`, never committed to.
  - `main`: Allmother. Upstream is merged into it regularly (merge, not rebase).
  - Feature branches come off `main`.
- Upstream sync (on a local clone):
  `git remote add upstream https://github.com/LawnchairLauncher/lawnchair.git` (once), then
  `git fetch upstream && git push origin upstream/16-dev:16-dev && git checkout main && git merge upstream/16-dev`
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

1. **Name and identity.** ✅ Name **Allmother**, `applicationId` **`com.honkbonkus.allmother`**
   (`.nightly` / `.play` for the other channels, plus `.debug` for debug builds). Never change it again:
   that would wipe users' layouts and settings.
2. **What makes it different?** ✅ Design fixes plus ADHD support (see Priority 4). Still needed: a concrete
   list of the design choices that bother me, so they can be split into "already a Lawnchair setting"
   (just change the default) and "needs code".
3. **Branch strategy.** ✅ See "Where things stand".
4. **Distribution target.** Sideload/GitHub only, F-Droid, or Play Store? Play forces stricter rules
   (Lawnchair already has a `play` flavor that disables things). This decides which flavors we keep.

## Priority 2: rebrand and make the fork safe to install

These are mechanical but touch "phone home" behavior, so they must be done before handing a build to anyone.

Every Allmother change to upstream files is marked with an `Allmother:` comment, so `git grep Allmother:`
lists them all during merges.

1. ✅ **applicationId and app name.** `build.gradle`: flavor IDs, `derived_app_name`, APK filename.
   The Java/Kotlin package stays `app.lawnchair.*`: renaming ~500 files would make every upstream merge conflict.
2. ✅ **In-app updater retargeted.** `ui/preferences/about/GithubService.kt` now has `REPO_OWNER` / `REPO_NAME` /
   `REPO_URL` constants used by the updater, changelog, and About links. Before this, a nightly build would have
   offered to "update" itself into upstream Lawnchair. It only appears in `nightly` builds, and it looks for a
   release tagged `nightly` on our repo, which doesn't exist yet. So for now it finds nothing.
3. ✅ **Announcements feed off by default** (`config_default_live_information_enabled` in
   `lawnchair/res/values/config.xml`), so nothing is fetched from lawnchair.app.
4. **Settings text still says "Lawnchair"** in ~20 English strings (onboarding, accessibility explanations, etc.),
   translated into 160 languages. Changing them creates merge conflicts on every translation update, so this is
   deferred. Best approach later: override just the English strings from a separate Allmother resource folder.
5. **About screen:** the GitHub/commit links now point to our repo. The Lawnchair team credits and community
   links are kept as attribution for now; to be replaced with an Allmother "built on Lawnchair" section later.
6. **Icons and visual identity.** Launcher icons are in `lawnchair/res/mipmap-*` and `nightly/res/mipmap-*`.
7. **Licensing.** The project is Apache 2.0, so we can ship it under our own name, but we must keep
   `LICENSE.txt` and existing copyright headers, and add our own notice. Don't use the Lawnchair name or logo
   for our product.
8. ✅ **CI.** `ci.yml` now just builds a debug APK and runs the style check on pushes to `main` / `*-dev` and on PRs.
   Removed: Crowdin, Telegram notifications, nightly publishing, issue auto-closers, Lawnchair's release
   workflows, funding links and issue templates. Upstream occasionally edits those files; on merge, keep them
   deleted (`git rm` them again). A release workflow comes back once distribution is decided.
9. **Release signing.** Generate *our own* keystore (`keystore.properties` + `.jks`, never committed).
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

### 4a. Design fixes (cheapest wins first)
1. List every design choice that bothers me.
2. For each, check whether Lawnchair already has a setting. If it does, Allmother just ships a different
   **default** (in `PreferenceManager2.kt` / `lawnchair/res/values/config.xml`). That's no new code and no merge pain.
3. Whatever's left becomes real code changes, done one at a time in Allmother's own package.
4. Default home layout and theme: `res/xml/default_workspace_*.xml`, `lawnchair/src/app/lawnchair/theme/`.

### 4b. ADHD features: build one end to end first
**My own feature requests come first** (being finalized; they'll be added here). The table below is only a pool of
early brainstorm ideas. Some will be used and many won't. We pick the first feature together, then build it fully
(setting → behavior → settings screen) to learn the codebase before tackling bigger ones.

| # | Feature | Why it helps | Builds on |
|---|---|---|---|
| 1 | **Calm home screen**: minimal default layout, notification dots/badges off by default, an option to show only the essentials | Fewer visual hooks pulling attention on every unlock | Existing settings (`NotificationDotsPreference`, grid/dock prefs) |
| 2 | **Distracting-apps list**: tag apps as "distracting"; they're hidden from the home screen and suggestions, and can be shown grayscale in the drawer | Out of sight, out of mind, without uninstalling | Hidden apps (`HiddenAppsPreferences.kt`), app filter, icon pipeline |
| 3 | **Intention pause**: opening a distracting app shows a short "what are you here for?" / a few-seconds pause screen first | Turns autopilot opens into decisions | Launch interception in `LawnchairLauncher` / app-launch path |
| 4 | **Time made visible**: big clock, "time since unlock" / "time in current app" cues, a quick-start timer | Helps with time blindness | Smartspace providers (`smartspace/`) |
| 5 | **Focus modes / routines**: different home layouts or app sets for work, evening, etc., switched manually or on a schedule | Context-appropriate phone | Gestures, prefs; bigger feature |
| 6 | **Today's task(s) on home**: one to three tasks pinned to the home screen | Keeps the priority in view | Smartspace or a custom widget |
| 7 | **Usage insight**: daily screen time for distracting apps, soft limits | Awareness, gentle accountability | `UsageStatsManager` (already used in `predictions/UsageStatsRanker.kt`) |

Design principle for all of these: **gentle friction, never punishment.** Easy to override, nothing that makes
the phone feel broken, and few settings (each extra setting is itself a distraction).

### 4c. Distribution
GitHub Releases first (simplest, good for personal use), then F-Droid/IzzyOnDroid or Play if wanted.

## Open questions

- [x] App name: **Allmother**
- [x] The core idea: design fixes + ADHD support
- [x] Package ID: `com.honkbonkus.allmother`
- [ ] List of the design choices that bother me
- [ ] Which ADHD feature to build first
- [ ] Personal use only, or published for others too?
- [ ] Keep the in-app updater (pointed at our releases) or remove it?
