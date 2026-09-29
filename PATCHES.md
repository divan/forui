# Sportity fork patches

This is `divan/forui`, a soft fork of [`duobaseio/forui`](https://github.com/duobaseio/forui)
(formerly `forus-labs/forui`)
maintained by [Sportity](https://sportity.com) for use in the Sportity
Admin SDS (`sportity/sds`). Branch
[`sportity-patches-0.27.3`](https://github.com/divan/forui/tree/sportity-patches-0.27.3)
carries the local delta on top of upstream. (Previous branches —
`sportity-patches-0.24.3` and the 0.22.3-based `sportity-patches` — are
kept for rollback.)

The intent is **temporary** — every patch on this branch should ship as
an upstream PR and be removed from the fork once merged. Fork ↔ upstream
rebases are expected.

**Current base:** `forui/0.27.3` release tag. Requires Flutter ≥3.47 /
Dart ≥3.13 (up from 3.44 / 3.12 at 0.24.3): since 0.26 forui builds on
the `material_ui` / `cupertino_ui` packages that Flutter 3.47 split out
of the framework, and its sources use Dart 3.13 primary constructors.
The icon package was renamed `forui_assets` → `forui_lucide` at 0.26; it
is a normal hosted dependency and needs nothing from this fork.

## Workflow

```bash
# Add upstream and rebase from main:
git remote add upstream https://github.com/duobaseio/forui
git fetch upstream --tags
# For the next upstream release, branch from its tag and re-apply:
git checkout -b sportity-patches-<ver> forui/<ver>
git cherry-pick <patch commits>
# Un-ignore the generated files (see "Generated files commit"), then:
flutter pub get                            # at the repo root (pub workspace)
cd forui && dart run build_runner build    # refresh generated files, commit them
flutter analyze lib && flutter test test/src/widgets/dialog/ --exclude-tags golden
```

When upstream merges the patch, retire the fork (switch `sportity/sds`
back to the pub.dev release — the generated-files commit is only needed
for `git:` consumption).

## Pinning from `sportity/sds`

`sds/pubspec.yaml` references this fork via `git:` dependency:

```yaml
dependencies:
  forui:
    git:
      url: https://github.com/divan/forui
      ref: sportity-patches-0.27.3    # pin to a SHA for stability
      path: forui
```

## Active patches

### 1. `FDialog`: stop dialog dismiss when popover-inside-dialog tap-outside fires (web)

**Symptom (web):** Open an `FDialog` containing a popover-menu trigger.
Tap to open the popover, then tap empty space inside the dialog. The
*entire dialog* dismisses instead of just the popover.

**Cause:** A popover's `TapRegion.onTapOutside` is *notification-only*
on web — it doesn't consume the pointer event. The same tap therefore
travels up to the `FAnimatedModalBarrier` rendered behind the dialog,
which calls `Navigator.pop`.

**Fix:** Wrap the dialog content in `Listener(behavior: opaque,
onPointerDown: noop, child: …)` inside `_FDialogState.build`
(`forui/lib/src/widgets/dialog.dart`). Pointer events that bubble out of
the dialog body are absorbed before the barrier sees them. Children
inside the dialog still receive pointer events because Flutter
dispatches to the deepest hit child first.

Re-verified against 0.27.3 (by reading the source, not by reproducing in
a browser): the five upstream commits that touched `dialog.dart` since
0.24.3 changed theming (`capturedScopes`, context-taking filters) and
syntax only. `_FDialogState.build` still has no pointer-absorbing
wrapper, and `FModalBarrier` still dismisses on any tap that reaches it.
The patch cherry-picked without conflicts and upstream's 12 dialog
widget tests pass with it applied.

**Upstream PR:** *not yet filed.* Open a repro project, file an issue
with reproducible steps, then propose this patch (or a more targeted
fix inside `FAnimatedModalBarrier`) as a PR.

## Generated files commit

Upstream excludes `**.design.dart` / `**.control.dart` from VCS because
pub.dev's publish pipeline runs build_runner and bundles the generated
files into the package archive. That path isn't taken for `git:`
dependencies — pub clones the repo as-is. This branch therefore loosens
`forui/.gitignore` and commits all generated files (100 at 0.27.3: 78
`.design.dart` + 22 `.control.dart`). After any rebase, regenerate
rather than cherry-pick this commit:

```bash
cd forui && dart run build_runner build
```

The generator (`forui_internal_gen`) resolves from the workspace, so it
always matches the checked-out sources. Localization files
(`lib/src/localizations/`) are tracked upstream and need no step here.

## Dropped at the 0.24.3 rebase (still dropped at 0.27.3)

- **`FAutocomplete`: force `EditableText` focus on web semantic-tap path**
  (0.22.3-era patch #2). Dropped without replacement: the admin repo's
  `e2e/docs/flutter-web-known-issues.md` investigation concluded the
  patch never actually fixed the Maestro-Web typing failure it targeted
  (the framework-level `MaterialApp.router` focus bug, flutter#119849);
  the e2e-side double-tap workaround is the effective fix. One sub-issue
  from that patch remains PR-worthy on its own: `FAutocomplete` does not
  forward the field's `groupId` to the popover's `TextFieldTapRegion`
  (still the default `EditableText` group at 0.27.3), so a custom
  `groupId` splits field and popover into different tap-region groups.
- **`FCalendar` header: `semanticsLabel` on prev/next chevrons**
  (0.22.3-era patch #3). Subsumed upstream: the re-implemented calendar
  passes localized `calendarPrevious/NextMonthSemanticsLabel` (and
  year/years variants) into the header buttons; English strings are
  identical to what the patch hard-coded (`'Previous month'` /
  `'Next month'`). Still present at 0.27.3.

## Dropped in earlier forks

These were in the previous `tmp_deps/forui_0.21.2_fork8` and **do not
need to be re-applied**:

- `FButton.semanticsLabel` parameter on `FButton`/`FButton.icon`/`FButton.raw`.
  Merged upstream as part of [#979](https://github.com/forus-labs/forui/pull/979).
- `pubspec.yaml` SDK constraint widening (`>=3.11.0` → `>=3.11.0-0`).
  Local Flutter SDK no longer requires it.
