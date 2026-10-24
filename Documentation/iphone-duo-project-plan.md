# Swiftfin for iPhone Duo: Engineering and UX Project Plan

Research date: October 24, 2026  
Repository: https://github.com/tvan10/Swiftfin  
Upstream: https://github.com/jellyfin/Swiftfin  
Baseline: `main` at `b0cb3127b1cef57b6508581be1cfe9addd5539a2`

> This document preserves the complete research readout, including the final open questions, prototypes, and source-link definitions. It is a planning document, not a runtime compatibility certification. The research was read-only; this document was subsequently saved to the fork at the owner's request.

**Swiftfin has a useful foundation for Duo, but it is not yet ready to claim a designed-for-Duo experience.** The most important work is to preserve playback while changing layout, replace private cutout measurements, remove phone-specific layout assumptions, and let system navigation controls adapt naturally. Pose-aware playback should follow those changes.

I recommend **one adaptive scene, system browsing/navigation bars, and a custom media surface with system-level actions**. Multiple windows and simultaneous use of both displays are optional extensions.

This was research and static source inspection only. No repository code was changed, and no issues, builds, or publications were created during the research.

## 1. Reproducible baseline and evidence limits

Inspected on **October 24, 2026**:

| Repository | Branch | Commit | Committer date, UTC | Divergence |
|---|---|---|---|---|
| [tvan10/Swiftfin](https://github.com/tvan10/Swiftfin/tree/b0cb3127b1cef57b6508581be1cfe9addd5539a2) | `main` | `b0cb3127b1cef57b6508581be1cfe9addd5539a2` | 2026-10-24 07:30:45 | 0 ahead / 0 behind upstream |
| [jellyfin/Swiftfin](https://github.com/jellyfin/Swiftfin/tree/b0cb3127b1cef57b6508581be1cfe9addd5539a2) | `main` | Same SHA | Same | Identical commit and tree |

The audit covers the default branches, not any unpublished or nondefault development branch.

I read [AGENTS.md][agents], the README, [contribution and architecture guidance][contributing], and the library, playback, OS-support, and TestFlight documentation. The architecture is shared SwiftUI application logic with platform-specific views and UIKit bridges. The repository instructions conditionally require the XcodeBuildMCP skill when using that tool; it was not used.

Evidence classifications used below:

- **Confirmed:** directly established from the pinned source.
- **Documented:** established by Apple or dependency-maintainer sources.
- **Recommendation:** proposed engineering or UX behavior.
- **Validation required:** cannot be established by static inspection.

No Xcode compilation, SDK-header inspection, simulator execution, or physical-device testing was performed. API declarations below were checked against Apple documentation or identified explicitly as video sample usage. This report does **not** certify runtime compatibility.

## 2. Platform behavior: compatibility is not optimization

Apple documents three materially different build behaviors:

| Build SDK | Documented Duo behavior | Implication |
|---|---|---|
| Before iOS 27 | Compatibility presentation; restricted screen utilization | Existing installation may run, but that is not full Duo support |
| iOS 27 | Greater inner-display utilization | Resizing assumptions become more visible |
| iOS 27.1 | Edge-to-edge utilization and vertical adaptation of standard navigation/toolbar controls | Rebuilding changes layout behavior even before adding Duo-specific code |

These are **build-SDK behaviors**, not reasons to raise the minimum deployment target. Keep Swiftfin's current **iOS 18.6** minimum initially, and availability-gate newer functionality. [Apple preparation video](https://developer.apple.com/videos/play/tech-talks/111461/), [preparation documentation](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo).

Important configuration distinctions:

- The outer display follows conventional iPhone orientation behavior.
- For an adaptive inner-display app, supported orientations cannot be treated as a layout constraint.
- `UIRequiresFullScreen` remains relevant to compatibility behavior; it does **not** prevent open/close resizing. Apple describes scaling on the inner display for that compatibility path.
- Multitasking with another app does not require Swiftfin to support multiple instances of itself.
- `UIApplicationSupportsMultipleScenes = false` is therefore compatible with the recommended first release.
- Remove `UIRequiresFullScreen` only alongside resizing validation, rather than treating the plist edit as sufficient support. Its behavior also varies across older iPad multitasking modes. [UIRequiresFullScreen documentation](https://developer.apple.com/documentation/bundleresources/information-property-list/uirequiresfullscreen).

Apple's terminology is **outer display**, **inner display**, **closed**, **fully open**, and **partially open/folded**, including holding it **like a book**, propping it on a surface, and standing it on its edges. “Tabletop” below is descriptive shorthand, not a proposed pose enum. Apple explicitly advises against inventing a separate screen for every pose. [Designing for iPhone Duo](https://developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo).

**Toolchain status.** Apple lists Xcode **27.1 (27A9275)** as released October 15 and **27.2 RC (27B48)** on October 19. Xcode 27.1 includes Swift 6.4, requires macOS Tahoe 26.6 or later, and provides the Duo SDK/runtime. Use a pinned 27.1 release build for the initial baseline and add a separate 27.2 RC validation lane. Do not equate “newer” with an already validated replacement. [Apple releases](https://developer.apple.com/news/releases/?id=06092025d), [27.1 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_1-release-notes), [27.2 notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_2-release-notes).

The iOS release-note index currently exposes 27 and 27.2 RC; the attempted standalone 27.1 release-note URL returned 404. One relevant 27.2-note clarification applies from **iOS 27.1**: ordinary `UIView` layout margins default to zero; view controllers still supply system minimum margins, and subviews can inherit them with `preservesSuperviewLayoutMargins`. Audit manually constructed player containers accordingly. [iOS 27.2 release notes](https://developer.apple.com/documentation/ios-ipados-release-notes/ios-ipados-27_2-release-notes).

## 3. Actual application and playback paths

The current path is:

`SwiftfinApp → RootView → UserSessionRootView → MainTabView → NavigationInjectionView → destination`

Each tab owns a `NavigationCoordinator` with a route path and sheet/full-screen presentation state. `MainTabView` already uses the modern `Tab` API, including a search-role tab.

Playback routes create/register a `MediaPlayerManager`, publish it through Factory, and present `VideoPlayer` full-screen through Transmission. `VideoPlayer` creates the selected engine proxy and uses a UIKit container for video, controls, supplements, and overlay actions. [Main tabs][tabs], [navigation injection][navigation], [playback route][playroute], [player][player].

This supports incremental improvement. A navigation rewrite or engine replacement is unnecessary.

### Source audit

P0 = prerequisite for a credible Duo release; P1 = core adaptive UX; P2 = optional enhancement. Confidence describes the finding, not whether its proposed fix has been tested.

| File / symbol and lines | Confirmed finding | User impact / risk | Recommended change | Priority / confidence |
|---|---|---|---|---|
| [`SwiftfinApp.body`, 27–37][app] | Root requests portrait for non-iPads; outer wrapper ignores safe areas | Restricts outer-display landscape; encourages assumptions that do not hold inside | Permit adaptive browsing orientations; audit foreground safe-area ownership | P0 / High |
| [`Info.plist`, 56–95][plist] | Multiple scenes disabled; full-screen requirement enabled; plist already lists portrait and both landscapes | Compatibility scaling/resizing behavior differs from fully adaptive mode | Remove full-screen requirement after resize work; retain single-scene initially | P0 / High |
| [`UIContainerViewController`, 605, 671, 744–781][container] | Compact mode is `UIDevice.isPhone && bounds.size.isPortrait`; layout cache tracks only size/insets | Large inner portrait can receive phone-compact UI; fold/camera changes may occur without those cache keys changing | Use local traits, usable bounds, reserved regions, and content requirements | P0 / High |
| [`NotchDeviceReader.measure/privateRect/privateCornerRadius`, 50–173][notch] | Uses private selectors, physical-screen checks, `UIScreen.main`, and orientation | Fragile display/camera assumptions; private-API distribution risk | Remove private measurements; public regions on 27.1, safe-area fallback earlier | P0 / High |
| [`VideoLayout.padding`, 90–102; `VideoViewport`, 136–164][videolayout] | Symmetrizes horizontal insets; geometry changes reset zoom | Wasted space with asymmetric bars; unexpected zoom loss on resize | Separate cinematic centering from control safety; reproject or deliberately clamp zoom | P0 / High |
| [`VideoPlayer.body`, 63–66, 92–97][player]; [`MediaPlayerManager._start`, 420–429][manager] | Appearance calls `start`; dismissal stops; initial provider is consumed and absent-provider start stops | Recreating/reappearing player is a continuity hazard | Make session start idempotent; separate view attachment from playback start/stop | P0 / High finding; runtime trigger unproven |
| [`PlaybackControls.Toolbar`, 29–73][playtoolbar]; [`PlaybackControls`, 40–85][controls] | Custom `HStack`, fixed button sizes and 50-point toolbar height; custom animations | Does not become a system vertical toolbar; text/controls may crowd | System actions plus purpose-built transport controls; flexible heights | P1 / High |
| [`MainTabView`, 69–125][tabs]; [`NavigationInjectionView`, 43–124][navigation] | System tabs/stacks; system sheets; Transmission full-screen zoom with zero preferred presentation insets | Good automatic adaptation baseline; custom full-screen presentation remains a risk | Preserve containers; validate Transmission transition and inset behavior | P0/P1 / High |
| [`View.topBarTrailing`, 411–417][viewext] | Wrapper creates real `ToolbarItemGroup` | Not inherently a custom-bar problem | Keep wrapper where useful; add semantic placement/priority options | P1 / High |
| [`LibraryElement.layout`, 69–101][libraryelement] | iPhone grids fixed at two landscape or three portrait-poster columns; other devices use minimum widths | Inner display can show oversized posters and too little content | Apply available-width/minimum-card sizing across idioms | P1 / High |
| [`SelectUserView.GridView`, 66–71][usergrid] | Phone-specific two-column grid | Large Duo layouts underused | Geometry-driven columns, accessibility-aware minimum size | P1 / High |
| [`ItemView.isCompact`, 50–52][itemview] | Uses measured width `< 600`, despite an outdated TODO | Better than idiom gating; abrupt transition still needs scrutiny | Retain content-driven breakpoint initially; test around 600 and preserve identity/scroll anchor | P1 / High |
| [`ContentGroupView`, 45–79][contentgroup]; [`SearchView`, 100–121][search] | Horizontal safe areas ignored; search also ignores keyboard safe area | Content may sit beneath side bars/keyboard unless children compensate | Restrict bleed to artwork; verify foreground insets through nested content groups | P0 / High code, Medium impact |
| [`ScrollViewOffsetModifier`, 22–28][scrolloffset] | Open-ended Introspect version range; replaces scroll-view delegate | Future framework changes or delegate replacement can affect scrolling | Prefer public scroll observation where suitable; otherwise explicitly validate and preserve delegate behavior | P1 / High |
| [`PanGesture`, 237–261][pangesture] | Brightness reads/writes `UIScreen.main` | Ambiguous target after display changes | Resolve current scene's screen dynamically; validate brightness semantics on hardware | P1 / High |
| [`AVMediaPlayerProxy`, 200–239, 267–272, 639–652][avproxy] | PiP integration and layer resizing exist; restore callback immediately reports success | Useful foundation, but UI restoration is not actually verified | Restore the correct scene/presentation before reporting success | P0 / High |
| [`TextSubtitleOverlay`, 19–24, 39–99][subtitles]; [`KeyCommandsModifier`, 35–185][keys] | Caption preferences, accessibility labels and keyboard commands exist | Avoid losing existing functionality; fixed sizing and command scope still require testing | Preserve caption settings; validate resize, focus, held keys, and editable-field conflicts | P1 / High |
| [CI line 35][ci], [TestFlight line 86][testflight], [project settings][project] | Xcode 26.6 defaults; iOS 18.6 target; app Swift language mode 5 | 27.1 build behavior is not exercised by defaults | Pin new SDK separately; retain language mode and deployment floor initially | P0 / High |

The earlier issue lead is confirmed: [Duo issue #2289](https://github.com/jellyfin/Swiftfin/issues/2289) references [iPad resizing #321](https://github.com/jellyfin/Swiftfin/issues/321), and #321 reports #2289 as its parent. Both were open. That establishes related work, not implemented support.

### Playback engines and dependencies

| Engine | Current integration | Implication |
|---|---|---|
| AVPlayer | Supported choice; custom `AVPlayerLayer` presentation, not a wholesale `AVPlayerViewController` UI | Existing transport UX must be adapted by Swiftfin; current PiP-capable adapter |
| VLC | Supported choice through SwiftVLC 1.0.0 | Preserve broad-format playback; independently validate drawable resizing |
| MPV | Implemented through MPVUI 0.5.0, but exposed only with `Experimental.mpvPlayer` | Include in experimental validation; do not advertise as equivalent standard support |

MPV currently shares VLC's advertised playback profiles. That is not proof of identical codec behavior. [Engine enum and supported cases][enginetype], [MPV adapter][mpvproxy].

The checked-in playback guide describes two standard players and omits MPV. It also should not be treated as a precise description of every current adapter capability.

Relevant dependency pins include Transmission **2.13.4**, Engine **2.12.4**, SwiftUI Introspect **27.0.0**, CollectionVGrid and CollectionHStack pinned to specific `main` revisions, MPVUI **0.5.0**, SwiftVLC **1.0.0**, MediaAccessibilityKit **0.2.0**, and local PreferencesView. LNPopupUI **2.0.2** is also resolved; a resolved package alone does not establish that it owns the active player UI. [Package.resolved][packages].

Maintainer findings:

- [Transmission](https://github.com/nathantannar4/Transmission/tree/f5a531fcece5d65c75397d01cf2910b31347e0b9) bridges UIKit presentations and interactive transitions. Test those paths before deciding to upgrade or replace it.
- [SwiftUI Introspect](https://github.com/siteline/SwiftUI-Introspect/tree/a37eca1db7bebc9437ee89508e7b4433ff34adb4) documents explicit validation for future OS versions. Its public hierarchy inspection is distinct from Swiftfin's private screen selectors.
- [SwiftVLC](https://github.com/harflabs/SwiftVLC/tree/192393a45cdcd7ebb6befac26586360b8b1676b3) wraps libVLC 4 directly, requires Swift 6.3+/Xcode 26.4+, and advertises iOS PiP. **Swiftfin's VLC proxy does not implement its PiP-capable protocol**, so library capability is not current app capability.
- [MPVUI](https://github.com/LePips/MPVUI/tree/cf277d979465b919b34bbdb33d62fdb24c377fbe) is mpv-backed. Its rendering-surface lifecycle still needs direct prototype validation.

There is no evidence supporting a blanket dependency upgrade or application-wide Swift 6 migration.

## 4. Display and pose behavior

The following is the recommended product behavior, using documented size classes, arrangement APIs, safe areas, and reserved regions. Do not select layouts from model names or raw hinge-angle thresholds.

| Configuration | Browsing recommendation | Playback recommendation | Signals / validation |
|---|---|---|---|
| Closed outer, portrait | Familiar tabs and stack; system vertical controls; responsive cards | One video viewport; fit by default; transport near video; system actions adapt | Local size classes/insets; test title truncation, search, overflow |
| Closed outer, landscape | Enable landscape; preserve destinations and query | Prioritize video and scrubbing; overflow secondary actions | Compact vertical space; test keyboard/PiP competing for bar space |
| Fully open inner, wide/landscape | More cards; optionally expose library hierarchy alongside detail | Continuous video across usable area by default; optional queue/info panel | Regular size classes plus actual width; no mandatory split because device opened |
| Fully open inner, tall/portrait | Expanded content, familiar horizontal system bars | Fit video with intentional letterboxing; show requested supplements where useful | Actual viewport and content requirements; avoid phone-compact idiom rule |
| Partially folded like a book | Preserve hierarchy; let system presentations move; keep scrolling content continuous | Default continuity for modest folds; offer region-separated video and controls/queue when it improves viewing | Active `.division` region; test reachability and whether automatic separation is desirable |
| Propped on a surface, inner display | Do not redesign browsing just because it folds | Video in upper region, controls/scrubber in lower region; queue/chapters available there | Overlay arrangement or region layout; test stable placement and subtitles |
| Standing like a tent, outer display in use | Same outer-display layout | Landscape viewing, large accessible transport targets | Active scene geometry, not a special “tent screen” |
| Split View multitasking, either side | Reduce columns; retain navigation, query and selection | Fit within app's actual window; collapse supplemental panels before transport | Either-side bars, asymmetric safe areas, continuous resizing |
| Inner camera becomes active | Keep foreground out of active occlusion | Reposition controls/subtitles; adjust video viewport only as needed; never restart | `.occlusion` changes, even without size change |
| iPhone Mirroring | Same adaptive hierarchy with mouse/keyboard support | Preserve session; support indirect input and resize | Real Mac mirroring test; custom gestures/presentations need special attention |

Apple documents outer portrait as compact-horizontal/regular-vertical, outer landscape as compact in both, and the full inner display as regular in both. Those are starting configurations, **not exhaustive classifications for every window size**. [Preparation video](https://developer.apple.com/videos/play/tech-talks/111461/).

For every row, older OS versions and conventional devices retain the same content/state model, using local geometry and safe areas without Duo reserved-region behavior. This avoids separate feature implementations.

Video-specific policy:

- **Aspect fit is the default.** Black letterboxing is intentional for preserving cinematic composition; do not apply game-oriented advice to eliminate all letterboxing.
- Preserve the user's fit/fill choice. Filling may crop content; folding must not silently select fill.
- Keep one continuous video on a flat display. A fold does not automatically justify splitting every layout.
- In a useful separated presentation, keep the video intact in one region—never render two independently cropped halves.
- Keep subtitles attached to the selected video viewport. Recompute text-caption layout; do not assume Swiftfin can independently reposition bitmap or engine-rendered subtitles.
- Scrubbing stays a horizontal time operation. Do not rotate the timeline into a side toolbar.
- Queue, chapters, people and media information remain the existing supplements, with stable selection and scroll anchors.
- During geometry changes, cancel/rebase in-progress gesture coordinates deliberately; never commit a seek merely because a view moved.

## 5. Recommended architecture and alternatives

### Layout architecture

Introduce a small, scene-local layout context, separate from playback state:

- Container bounds and usable content rectangle.
- Independent safe-area edges and layout margins.
- Horizontal/vertical size classes.
- Active division and occlusion regions, in a defined coordinate space.
- Preferred vertical-bar edge.
- Display scale, layout direction, Dynamic Type and accessibility preferences.

Use that context to calculate a layout plan: video rectangle, transport rectangle, supplement presentation and available action space.

Extend the existing `viewDidLayoutSubviews`/geometry-driven machinery rather than replacing it with device-orientation notifications. The cache must invalidate for **region, trait and scale changes**, not just size and safe-area changes.

Avoid treating all insets as equivalent. A camera exclusion, toolbar inset, cinematic centering margin and caption margin have different purposes. In particular, the current `max(leading, trailing)` approach should not become the shared safe-area policy.

### Verified API surface

| API | Verified declaration / usage | Availability and limitations |
|---|---|---|
| `GeometryProxy.reservedRegions(kind:options:layoutDirectionBehavior:)` | Returns `[ReservedRegion]`; defaults: `options: []`, layout direction `.mirrors` | iOS 27.1+. Active regions only by default; frames local to queried view |
| `UIView.reservedRegions(kind:options:)` | Returns `[UIView.ReservedRegion]` | iOS 27.1+. Use UIKit-local coordinates; do not use a guessed `UIViewReservedRegion` Swift spelling |
| `ArrangementView` | Primary and secondary view closures; `.arrangementViewStyle(.overlay)` or `.split` | iOS 27.1+. Layout container, not navigation infrastructure |
| `ToolbarContent.axisBehavior(_:)` | Accepts `ToolbarItemAxisBehavior` | iOS 27.1+; `.verticalPreferred` is a preference, not forced placement |
| `EnvironmentValues.toolbarVerticalEdge` | `HorizontalEdge?` | iOS 27.1+; preferred edge even when bar is hidden |
| `UITraitCollection.verticalBarEdge` | `UIVerticalBarEdge` | iOS 27.1+; same visibility caveat |
| `ToolbarContent.visibilityPriority(_:)` | Accepts `ToolbarItemVisibilityPriority` | iOS 27.0+ |
| `ToolbarOverflowMenu` | Toolbar content containing persistent overflow actions | iOS 27.0+ |
| `.topBarPinnedTrailing` | `ToolbarItemPlacement` | iOS 27.0+; can still overflow during active search when space is insufficient |
| `View.toolbarVerticalCompressionBehavior(_:)` | Accepts `ToolbarVerticalCompressionBehavior` | iOS 27.1+ |
| `View.toolbarVerticalBehavior(_:)` | Accepts `ToolbarVerticalBehavior` | iOS 27.1+; use `.disabled` as a stable policy, not a frequently toggled state |

Direct references: [SwiftUI regions](https://developer.apple.com/documentation/swiftui/geometryproxy/reservedregions(kind:options:layoutdirectionbehavior:)), [UIKit regions](https://developer.apple.com/documentation/uikit/uiview/reservedregions(kind:options:)), [arrangements](https://developer.apple.com/documentation/swiftui/arrangementview), [axis behavior](https://developer.apple.com/documentation/swiftui/toolbarcontent/axisbehavior(_:)), [vertical edge](https://developer.apple.com/documentation/swiftui/environmentvalues/toolbarverticaledge), [UIKit edge](https://developer.apple.com/documentation/uikit/uitraitcollection/verticalbaredge), [priority](https://developer.apple.com/documentation/swiftui/toolbarcontent/visibilitypriority(_:)), [overflow](https://developer.apple.com/documentation/swiftui/toolbaroverflowmenu), [pinned placement](https://developer.apple.com/documentation/swiftui/toolbaritemplacement/topbarpinnedtrailing), [compression](https://developer.apple.com/documentation/swiftui/view/toolbarverticalcompressionbehavior(_:)), [vertical behavior](https://developer.apple.com/documentation/swiftui/view/toolbarverticalbehavior(_:)).

Use `.division` for the fold and `.occlusion` for camera/system exclusions. `.includeInactive` is useful for anticipating layout structure, not for excluding nonexistent visible content. SwiftUI region frames mirror by default in RTL; use `.fixed` only when the positioning code uses unmirrored physical coordinates. This matters for the UIKit/SwiftUI boundary.

Apple's material differs in how broadly it warns about arrangement nesting. The conservative implementation here is a **standalone player `NavigationStack` containing one arrangement**, outside browsing split-view machinery and outside scrolling containers.

### Browsing alternatives

| Approach | Benefit | Cost / tradeoff |
|---|---|---|
| **A. Keep tabs and per-tab stacks; improve responsive content** | Lowest migration risk; existing routes and search state survive; system bars adapt | Does not automatically expose a second hierarchy level |
| B. Add `NavigationSplitView` for library/list/detail navigation | More useful hierarchy on the inner display | Requires explicit selection/path mapping during collapse; larger state-restoration and deep-link scope |

**Recommendation:** ship A first. Prototype B for library browsing later, preserving the existing route model. Do not swap whole navigation roots whenever width crosses a breakpoint.

The video demonstrates `.defaultTabBarPlacement(.sidebar)` as an optional richer navigation treatment. It is not necessary for compatibility; confirm its complete SDK declaration before adopting it. [Preparation video](https://developer.apple.com/videos/play/tech-talks/111461/).

### Toolbar rules for Swiftfin

Automatic vertical behavior comes from **navigation containers**: SwiftUI `TabView`, `NavigationStack`, `NavigationSplitView`, and UIKit `UITabBarController`/`UINavigationController`. Standalone custom `UIToolbar`, `UINavigationBar`, `UITabBar`, and Swiftfin's `HStack` toolbar do not acquire that behavior just by resembling system bars.

Use:

- System back navigation wherever possible.
- `.cancellationAction` for an explicit Close.
- A prominent pinned trailing action only when justified.
- Separate semantic items/groups for secondary actions.
- Existing search-role tab and `.searchable`; do not build a narrow vertical search text field.
- Both localized title and symbol, allowing the system to choose presentation.
- Visibility priorities for frequently used actions; persistent secondary actions in system overflow.
- Default tab-preserving compression while browsing.

Text-only and complex custom items normally remain horizontal. Explicitly opt a custom item into vertical preference only after giving it a real narrow representation. Keep meaningful numeric text, sliders and transport timelines horizontal. Group controls instead of inserting arbitrary spacers. [Raise the bar](https://developer.apple.com/videos/play/tech-talks/111462/).

Only the appropriate edge-aligned container participates: split-view detail bars can be vertical while other columns and expanded inspectors remain horizontal. Outer sheets can have vertical bars; centered/leading inner sheets remain horizontal, while trailing placement can participate vertically. Let system sheets/popovers handle fold displacement. [Preparation documentation](https://developer.apple.com/documentation/technologyoverviews/preparing-your-app-for-iphone-duo).

### Playback alternatives

| Approach | Benefit | Tradeoff |
|---|---|---|
| **A. System navigation/actions plus custom transport and video** | Familiar Close/overflow; preserves all engines and Swiftfin's scrubbing/supplements | Must coordinate system-bar visibility, reserved regions and custom overlays |
| B. Entirely custom immersive controls with stable horizontal-bar opt-out | Maximum video width; fewer mixed layout systems | Swiftfin owns displacement, overflow, accessibility, placement and consistency |
| C. Native AVKit controller for AVPlayer only | Less native-player UI ownership | Divergent engine UX; not a replacement for VLC/MPV functionality |

**Recommend A**, with B as a prototype comparator for full-screen media.

For pose-aware playback, Apple's documented overlay arrangement is particularly appropriate:

```swift
ArrangementView {
    PlayerControls()
} secondary: {
    VideoSurface()
}
.arrangementViewStyle(.overlay)
```

This is an architectural sketch, not compiled Swiftfin code. Flat layouts overlay controls on video; folded layouts put the secondary video in the leading/top region and primary controls in trailing/bottom. Keep engine identity outside both children. [ArrangementView documentation](https://developer.apple.com/documentation/swiftui/arrangementview).

For queue/details alongside video, use a split relationship instead of an overlay relationship, or retain the existing supplement container with region-aware constraints. Do not nest arrangements speculatively.

Disabling vertical bars is defensible for an immersive player or a control-heavy sheet when it demonstrably improves space. Apple explicitly lists full-screen video as a possible case. The cost is losing the usual side-control convention. Make it a stable presentation-level decision; hiding controls is a visibility operation, not a reason to toggle vertical behavior. [Vertical behavior documentation](https://developer.apple.com/documentation/swiftui/view/toolbarverticalbehavior(_:)).

### Playback ownership and continuity

Create a durable playback-session owner above adaptive layout branches. It should retain:

- Engine proxy and playback item/session ID.
- Requested playback state, time, rate and selected tracks.
- Queue position and playback offsets.
- Audio-session/Now Playing ownership.
- PiP controller and restoration target.
- Fit/fill preference and meaningful zoom state.

Keep transient presentation state separate: visible controls, active supplement, pending scrub, focused control and panel scroll positions.

Required invariants:

1. Geometry changes never call engine load/start/stop.
2. A layout branch does not instantiate another proxy.
3. Session start is idempotent.
4. Presentation dismissal and playback termination are distinct operations.
5. Backgrounding follows the existing user preference and PiP policy; folding alone is not a pause command.
6. Only one owner sends playback progress and manages audio activation.

The existing manager is a good starting point, but its global Factory registration and view-owned proxy are unsuitable for casually enabling multiple scenes.

Retain AVPlayer's layer and update its bounds. For VLC/MPV, verify whether moving or rebuilding the host detaches the drawable or restarts rendering. Re-read display scale from traits/current scene when needed; do not cache the initial screen. A single layer or drawable must not be attached to two simultaneous hosts.

### Scenes

A single adaptive scene is sufficient for the recommended release.

Optional multiwindow browsing can come later, with independent navigation/search state per scene and explicit ownership of the single playback session. New-scene availability is dynamic: Apple documents that new windows cannot be created on the outer display. Its video recommends `UIWindowSceneActivationAction` to reflect availability; exact integration should be compiled in a small prototype before use.

Do **not** assume scene accessories permit a general outer-display media remote. Apple's `CameraCaptureAccessory` example requires a full-screen inner-display camera experience and an active capture session. The 27.1 simulator explicitly relaxes that capture-session requirement, so a simulator success would be misleading. [Multiple displays/scenes video](https://developer.apple.com/videos/play/tech-talks/111464/), [27.1 release notes](https://developer.apple.com/documentation/xcode-release-notes/xcode-27_1-release-notes).

Raw hinge APIs are intended for interactions/effects. Layout should use arrangements and regions. No hinge-angle observer is needed for the baseline.

## 6. Phased implementation backlog and effort

Estimates assume one experienced SwiftUI/UIKit engineer, access to a Duo and Mac, representative Jellyfin media, and no major upstream binary-engine defect. They include focused testing but not prolonged TestFlight feedback or upstream review. Ranges overlap; they are not commitments.

| Phase | Concrete work | Acceptance criteria | Effort |
|---|---|---|---|
| **1. SDK/build compatibility — must fix** | Pin Xcode 27.1; retain iOS 18.6/Swift 5 mode; resolve existing pins; validate macros, binary frameworks, scripts and signing | Clean iOS simulator/device builds and archive; tvOS regression build; older-OS launch; exact Xcode/SDK recorded | **2–5 days** |
| **2. Resizing, safe areas, continuity — must fix** | Remove private cutout APIs; replace idiom rules; introduce layout context; make session attachment/start safe; validate full-screen-key removal | Repeated fold/resize preserves engine/session/time/tracks; no hidden foreground controls; camera-only changes update layout; no constraint failures | **7–15 days** |
| **3. Navigation and toolbars — core UX** | Retain system tabs/stacks; semantic actions, labels, priority/overflow; responsive grids; fix search/inset handling | All actions reachable in smallest layouts; Close/back correct; search survives resize; sheets/popovers stay usable; RTL and large text pass | **5–10 days** |
| **4. Pose-aware playback — enhancement after stability** | Prototype overlay versus custom region layout; video/control separation; supplements and subtitles; zoom/gesture policy | No engine recreation; video remains intact; captions avoid exclusions; fold changes do not seek; user can retain understandable continuous viewing | **7–15 days** |
| **5. Advanced capabilities — optional** | Library split navigation, browsing while watching, additional scenes, VLC PiP adapter, external-display workflows | Each independently preserves one playback owner and has explicit restore/close semantics | **5–15 days per capability** |

A realistic initial release covering phases 1–3 is approximately **3–6 engineer-weeks**. Adding polished pose-aware playback makes **5–9 weeks** plausible. Engine lifecycle defects could extend that substantially.

Do not make a sidebar redesign, VLC PiP, or multiwindow support release blockers for basic Duo readiness.

### Build, signing and Xcode Cloud

The current GitHub workflows default to 26.6; TestFlight supports an override. Their manual certificate/profile workflow is separate from Xcode Cloud.

For your own TestFlight account:

- Use your own bundle identifier, team and App Store Connect app record. The repository supports a local override through `DevelopmentTeam.xcconfig`; do not rely on an ignored local file being present in Cloud.
- Select the shared `Swiftfin` scheme and an explicitly supported Xcode version. Confirm 27.1 availability in your Cloud account rather than assuming the GitHub setting controls it.
- Keep `Package.resolved` committed. Apple says Xcode Cloud relies on it instead of automatic resolution.
- Install required tools through an executable `ci_scripts/ci_post_clone.sh` if needed. Audit the existing translation generator, formatter, linter and branch-information build phases. The formatter can modify checkout files during a build; prefer deterministic checking in CI.
- Exercise cold dependency downloads, especially large SwiftVLC/MPV artifacts, and macro compilation with the new compiler.
- Ensure build numbers are unique, signing/entitlements match your app, and the archived product—not merely a simulator build—installs through TestFlight.
- Retain symbols and record the source SHA, SDK, package revisions and engine in test results.

[Repository signing configuration][xcconfig], [build phases][project], [Cloud dependency guidance](https://developer.apple.com/documentation/xcode/making-dependencies-available-to-xcode-cloud), [Cloud scripts](https://developer.apple.com/documentation/xcode/writing-custom-build-scripts).

## 7. Verification matrix

Use simulator testing for rapid geometry coverage and physical devices for playback, display transfer, audio, PiP, thermal behavior and accessibility. Apple's own contribution guidance warns that simulator behavior differs for PiP and other features.

| Test area | Simulator / automation | Physical-device acceptance |
|---|---|---|
| Displays and poses | Closed/open, wide/tall, book-like, propped; both multitasking sides | Same item/session continues through 20 repeated transitions |
| Continuous resizing | Sweep small-to-large sizes and across 600-point detail breakpoint | No blank surface, lost selection, unintended navigation or restart |
| Presentations | Fold/resize while sheet, menu, popover, alert or zoom transition is active | Presentation remains attached to correct scene; dismiss works once |
| Playback states | Playing, paused, buffering, seeking, end-of-item, next episode | No duplicate audio/progress reports or accidental resume/pause |
| Engines | AVPlayer, VLC; MPV with experimental flag | H.264/HEVC, relevant HDR, MKV/HLS/live/transcoded samples appropriate to each engine |
| Captions | SRT/WebVTT, long text, RTL text, caption-size changes | ASS/bitmap/embedded/burned-in subtitles; no clipping or fold/camera overlap |
| Gestures | Resize during scrub, pan, pinch, double-tap and supplement drag | No accidental seek/dismiss; touch targets remain reachable |
| PiP | Availability and state-machine coverage | Start/stop, fold, background, restore, close; AVPlayer first; unsupported engines do not promise PiP |
| Camera/reserved regions | Activate/deactivate occlusion without changing bounds | Controls/captions react; playback and navigation stay intact |
| Keyboard and pointer | Space/arrows/Command shortcuts; tab focus; search active | Hardware keyboard, held keys, trackpad pinch/scroll, discrete mouse wheel |
| Accessibility | Largest text sizes, RTL, reduced motion/transparency, contrast | VoiceOver order/focus across fold; Switch Control/Full Keyboard Access; captions remain usable |
| Lifecycle | Background/foreground, interruption, scene restoration | Lock, audio route changes, interruptions, memory pressure; no duplicate audio-session ownership |
| Mirroring / regression | Conventional iPhones, narrow/wide iPads, iOS 18.6 and 26.x | Mac iPhone Mirroring, biometric-login behavior, real display-scale changes; tvOS unchanged |

Add focused tests for:

- Layout calculations with asymmetric edges and nonzero coordinate origins.
- Region changes with unchanged bounds.
- Playback start/attach/detach invariants.
- Route/selection restoration across layout changes.
- Scrub cancellation/rebasing.

Measure session identity, load/start/stop counts, server play-session IDs and audio discontinuities. A screenshot alone cannot establish continuity.

Apple documents specific indirect-input concerns for custom gestures and presentations in Mirroring. Swiftfin already enables indirect input, but that does not validate its recognizers. [TN3210: iPhone Mirroring](https://developer.apple.com/documentation/technotes/tn3210-optimizing-your-app-for-iphone-mirroring).

## 8. Open questions and smallest prototypes

| Unresolved question | Smallest useful prototype | Decision it enables |
|---|---|---|
| Does full-screen Transmission presentation retain the player during display transitions? | Existing player with lifecycle counters; compare current transition to standard full-screen presentation | Retain Transmission unchanged, patch narrowly, or replace only this path |
| Can all three renderers resize/rehost without reload? | One engine instance per test, local media, resizable host and display-scale logging | Exact surface-ownership design |
| Does overlay arrangement preserve custom player identity and gesture behavior? | Two-child player/controls arrangement, no browsing rewrite | Choose arrangement or explicit region constraints |
| Is automatic book-like separation actually better for video? | Compare continuous viewport and separated mode with identical session state | Default policy and whether an explicit viewing-mode control is needed |
| How should active-camera occlusion affect fit/fill? | Fixed-size player with toggled occlusion; captions and transport visible | Minimal displacement versus viewport resizing |
| Can current PiP reliably restore the correct UI? | AVPlayer PiP with real restoration routing instead of unconditional success | Required session/presentation changes |
| Is SwiftVLC PiP practical without a dependency change? | Adapter-only experiment against pinned 1.0.0 | Separate optional feature scope |
| What does the installed SDK actually accept? | Compile-only sample covering listed APIs on 27.1, with iOS 18.6 deployment and tvOS build | Confirm headers, overloads, actor annotations and availability guards |
| Does Xcode Cloud build the pinned graph reproducibly? | Clean archive workflow with exact Xcode and dependency pins | Necessary script/signing changes only |

The highest-value first prototype is **a stable player session inside a continuously resizable host, instrumented across open/close, active camera regions, backgrounding and PiP**. That resolves the largest architectural risks before investing in a new visual layout.

## 9. Direct source links

Platform claims are linked inline above. The following pinned source links support the code audit and preserve the exact research baseline.

- [Repository instructions][agents]
- [Contribution and architecture guidance][contributing]
- [Application entry point][app]
- [iOS Info.plist][plist]
- [Main tabs][tabs]
- [Navigation and presentation injection][navigation]
- [Playback routes][playroute]
- [VideoPlayer][player]
- [Player UIKit container][container]
- [NotchReader][notch]
- [Video layout and viewport][videolayout]
- [MediaPlayerManager lifecycle][manager]
- [Custom playback toolbar][playtoolbar]
- [iOS playback controls][controls]
- [Toolbar wrapper][viewext]
- [Library grid sizing][libraryelement]
- [User-selection grid][usergrid]
- [Item detail breakpoint][itemview]
- [Content-group scrolling][contentgroup]
- [Search][search]
- [Scroll introspection][scrolloffset]
- [Player pan gestures][pangesture]
- [AVPlayer adapter and PiP][avproxy]
- [MPV adapter][mpvproxy]
- [Text subtitles][subtitles]
- [Player keyboard commands][keys]
- [Engine selection and experimental gating][enginetype]
- [Build CI][ci]
- [TestFlight workflow][testflight]
- [Project build phases and settings][project]
- [Dependency lockfile][packages]
- [Signing configuration][xcconfig]

[agents]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/AGENTS.md
[contributing]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Documentation/contributing.md
[app]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin/App/SwiftfinApp.swift#L27
[plist]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin/Resources/Info.plist#L56
[tabs]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Coordinators/Tabs/MainTabView.swift#L69
[navigation]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Coordinators/Navigation/NavigationInjectionView.swift#L43
[playroute]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Coordinators/Navigation/NavigationRoute/NavigationRoute%2BMedia.swift#L51
[player]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/VideoPlayer/VideoPlayer.swift#L41
[container]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/VideoPlayer/Container/VideoPlayer%2BContainer.swift#L600
[notch]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin/Components/NotchReader.swift#L50
[videolayout]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/VideoPlayer/Container/VideoPlayer%2BVideoLayout.swift#L90
[manager]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Objects/MediaPlayerManager/MediaPlayerManager.swift#L420
[playtoolbar]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/VideoPlayer/Components/Toolbar/VideoPlayer%2BToolbar.swift#L29
[controls]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin/Views/VideoPlayer/PlaybackControls/PlaybackControls.swift#L40
[viewext]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Extensions/ViewExtensions/ViewExtensions.swift#L411
[libraryelement]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Components/LibraryElement.swift#L69
[usergrid]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/SelectUserView/Components/SelectUserView%2BGridView.swift#L66
[itemview]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/ItemView/ItemView.swift#L50
[contentgroup]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/ContentGroupView.swift#L45
[search]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/SearchView.swift#L100
[scrolloffset]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Extensions/ViewExtensions/Modifiers/ScrollViewOffsetModifier.swift#L22
[pangesture]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin/Views/VideoPlayer/Gestures/VideoPlayerContainerView%2BPanGesture.swift#L237
[avproxy]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Objects/MediaPlayerManager/MediaPlayerProxy/MediaPlayerProxy%2BAVPlayer.swift#L200
[mpvproxy]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Objects/MediaPlayerManager/MediaPlayerProxy/MediaPlayerProxy%2BMPV.swift#L100
[subtitles]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Views/VideoPlayer/Components/TextSubtitleOverlay.swift#L19
[keys]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin/Views/VideoPlayer/VideoPlayer%2BKeyCommands.swift#L35
[enginetype]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Shared/Objects/VideoPlayerType/VideoPlayerType.swift#L12
[ci]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/.github/workflows/ci.yml#L35
[testflight]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/.github/workflows/testflight.yml#L86
[project]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin.xcodeproj/project.pbxproj#L751
[packages]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/Swiftfin.xcodeproj/project.xcworkspace/xcshareddata/swiftpm/Package.resolved
[xcconfig]: https://github.com/tvan10/Swiftfin/blob/b0cb3127b1cef57b6508581be1cfe9addd5539a2/XcodeConfig/Shared.xcconfig#L9
