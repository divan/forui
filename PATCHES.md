# Sportity fork patches

This is `divan/forui`, a soft fork of [`duobaseio/forui`](https://github.com/duobaseio/forui)
(formerly `forus-labs/forui`)
maintained by [Sportity](https://sportity.com) for use in the Sportity
Admin SDS (`sportity/sds`). Branch [`sportity-patches`](https://github.com/divan/forui/tree/sportity-patches)
carries the local delta on top of upstream.

The intent is **temporary** — every patch on this branch should ship as
an upstream PR and be removed from the fork once merged. Fork ↔ upstream
rebases are expected.

**Current base:** `forui/0.22.3` release tag. Requires Flutter ≥3.44 /
Dart ≥3.12 (upstream bumped the SDK constraints in 0.22.0).

## Workflow

```bash
# Add upstream and rebase from main:
git remote add upstream https://github.com/duobaseio/forui
git fetch upstream
git checkout main
git merge --ff-only upstream/main
git checkout sportity-patches
git rebase main          # or: git rebase upstream/main
```

When upstream merges one of the patches, drop the corresponding commit
during rebase. When the branch is empty, retire the fork.

## Pinning from `sportity/sds`

`sds/pubspec.yaml` references this fork via `git:` dependency:

```yaml
dependencies:
  forui:
    git:
      url: https://github.com/divan/forui
      ref: sportity-patches    # pin to a SHA for stability
      path: forui
```

## Active patches

Each commit on `sportity-patches` has a detailed message; this file is
the index. Order matches branch history (oldest → newest).

### 1. `FDialog`: stop dialog dismiss when popover-inside-dialog tap-outside fires (web)

Commit subject above; full message in `git log`.

**Symptom (web):** Open an `FDialog` containing a popover-menu trigger.
Tap to open the popover, then tap empty space inside the dialog. The
*entire dialog* dismisses instead of just the popover.

**Cause:** A popover's `TapRegion.onTapOutside` is *notification-only*
on web — it doesn't consume the pointer event. The same tap therefore
travels up to the `FAnimatedModalBarrier` rendered behind the dialog,
which calls `Navigator.pop`.

**Fix:** Wrap the dialog content in `Listener(behavior: opaque,
onPointerDown: noop, child: …)`. Pointer events that bubble out of the
dialog body are absorbed before the barrier sees them. Children inside
the dialog still receive pointer events because Flutter dispatches to
the deepest hit child first.

**Upstream PR:** *not yet filed.*  Open a repro project, file an issue
with reproducible steps, then propose this patch (or a more targeted
fix inside `FAnimatedModalBarrier`) as a PR.

### 2. `FAutocomplete`: force `EditableText` focus on web semantic-tap path

**Symptom (web):** Reach an `FAutocomplete` via screen reader,
keyboard navigation, or e2e tooling that taps via the accessibility
layer (e.g. Maestro Web). The suggestions popover opens but typing
into the field does nothing.

**Cause:** Flutter Web's standard pointer-event path establishes
`EditableText` focus on first `pointerDown`. The semantic-tap path
invokes `FTextFormField.onTap` *without* doing that. The popover opens
(because `_popoverController.show` runs), but the underlying
`EditableText` is unfocused and ignores keystrokes.

**Fix:** Explicitly call `_fieldFocus.requestFocus()` in two places —
the outer `Listener.onPointerDown` (covers the regular path; harmless
if focus is already held) and `FTextFormField.onTap` (covers the
semantic path). Also swallow `onTapOutside` *while the suggestions
popover is shown* so a popover tap doesn't auto-blur the field on web.
Note the popover is already wrapped in `TextFieldTapRegion`, so ordinary
pointer taps inside it count as "inside" the field's tap region — the
swallow is a web fallback for event paths (semantic taps, certain DOM
paths) that bypass the tap-region grouping. When the popover is hidden,
the callback mirrors `EditableText`'s default tap-outside behavior
(unfocus, except touch taps on native mobile), since the framework
default is unreachable once a callback is provided.

Additionally forwards the field's `groupId` to the popover's
`TextFieldTapRegion` (upstream defaults it to `EditableText`, so a
custom `FAutocomplete.groupId` splits the field and popover into
different tap-region groups — every popover tap then fires the field's
`onTapOutside` on all platforms). This is an upstream bug, PR-worthy
on its own.

**Upstream PR:** *not yet filed.* Easiest repro: a Maestro Web flow
matching the field via `text:` and tapping it. The `groupId` forward
can be repro'd on any platform by passing a custom `groupId`.

### 3. `FCalendar` header: add `semanticsLabel` to prev/next chevrons

**Symptom:** The `FCalendar` previous/next month chevrons are
`FButton.icon` with no visible text. Combined with `FButton`'s
"`excludeSemantics` when `semanticsLabel` is set" behavior, the result
is buttons that are invisible to assistive tech and to e2e tooling
that matches via accessibility.

**Fix:** Pass `semanticsLabel: 'Previous month'` / `'Next month'`. Uses
the `FButton.semanticsLabel` parameter that upstream already merged in
0.21.3 (#979).

**Open follow-up:** Strings are hard-coded English. A follow-up should
route them through `FCalendarLocalizations` (likely an
`Iru` getter). PR-worthy as a small i18n-cleanup change.

**Upstream PR:** trivial; should land easily once #979 is part of a
release.

## Dropped from earlier forks

These were in the previous `tmp_deps/forui_0.21.2_fork8` and **do not
need to be re-applied**:

- `FButton.semanticsLabel` parameter on `FButton`/`FButton.icon`/`FButton.raw`.
  Merged upstream as part of [#979](https://github.com/forus-labs/forui/pull/979).
- `pubspec.yaml` SDK constraint widening (`>=3.11.0` → `>=3.11.0-0`).
  Local Flutter SDK no longer requires it.
