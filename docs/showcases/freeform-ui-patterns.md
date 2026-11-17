# Freeform UI patterns: panels, canvas, popovers and tools

A design study of Apple Freeform on macOS, made with REA and standard macOS
tooling. It records measurements and behaviour so that similar patterns can be
re-implemented from scratch. It contains no Apple code, images, icons or sounds.
[DISCLAIMER.md](../../DISCLAIMER.md) applies.

**Status: partial.** Static evidence and live accessibility measurements are
complete. Pixel evidence (materials, corner radii, shadows, colours of system
chrome, animation curves from frames) is not captured, because the session had
no Screen Recording permission. Every such value is marked **Unknown** below.
See [What is not measured](#what-is-not-measured).

## Target and method

| Item           | Value                                                                    |
| -------------- | ------------------------------------------------------------------------ |
| Target         | `/System/Applications/Freeform.app`, version 4.5, `com.apple.freeform`   |
| Binary         | `Contents/MacOS/Freeform`, arm64e slice, SHA-256 of the file `e567ee43…` |
| Host           | macOS 26.6.2 (25G83), 1512×982 pt display at 2x, 120 Hz, Dark appearance |
| REA            | `rea-agents` 6.9.0 CLI, run on Node.js 24.22.3                           |
| Analysis date  | 2026-11-16                                                               |
| Native engines | None installed (no Hopper, Ghidra or IDA)                                |

Every value carries one of these labels.

| Label        | Meaning                                                                          |
| ------------ | -------------------------------------------------------------------------------- |
| **Measured** | Read from the running app through the macOS Accessibility API (`AXUIElement`)    |
| **Derived**  | Read from the shipped bundle: a nib field, an asset record or a constant in code |
| **Inferred** | A conclusion drawn from names or from documented AppKit behaviour, not observed  |
| **Unknown**  | Not established. Do not treat a guess from another app as a Freeform measurement |

Tools used and what each one produced:

- `rea decode-interface-builder`: the main menu, its key equivalents and action
  selectors, from `Base.lproj/CRLMacMainMenu.nib`.
- `rea inspect-asset-catalog`: named colours in `Assets.car` (metadata only, no
  image bytes).
- `rea inspect-desktop-application` and `rea inspect-native-dispatch-metadata`:
  bundle layout and a first slice of class metadata.
- `dyld_info -objc`, `dyld_info -function_starts`, `dyld_info -fixups` and
  `llvm-objdump -d`: Objective-C method addresses and the constants inside
  selected methods. Addresses below are virtual addresses in the arm64e slice.
- `nm -u` and `strings`: imported system classes and symbol names.
- A small Objective-C reader built on `AXUIElementCopyAttributeValue` and
  `AXUIElementPerformAction`: element frames, roles and values in the live
  window. All live interaction went through accessibility presses on a new
  empty board. No pointer or keyboard events were synthesised.

Tools that did not work on this host:

- `rea inspect-macho` and `rea observe-native-ui` compile a Swift helper. The
  Command Line Tools Swift compiler fails here with
  `redefinition of module 'SwiftBridging'`.
- Xcode is not installed, so Accessibility Inspector and Quartz Debug were not
  available.
- `screencapture` and Digital Color Meter need Screen Recording permission.

## 1. Floating panels

Freeform draws almost all chrome with system components. The main binary
imports `NSToolbar`, `NSToolbarItemGroup`, `NSMenuToolbarItem`,
`NSSearchToolbarItem`, `NSTrackingSeparatorToolbarItem`, `NSPopover`, `NSPanel`,
`NSSplitViewController`, `NSScrollView`, `NSVisualEffectView` and `NSGlassView`
(**Derived**, `nm -u`). The asset catalog holds 33 named colours, 25 of which
belong to the app icon (**Derived**). Panel materials, tints, hairlines and
shadows therefore come from AppKit, and the clean way to reproduce them is to
use the matching public API.

### 1.1 Toolbar

Window under test: 1512×891 pt. All frames are **Measured** in screen points.

| Element                     | Frame or size                         | Notes                                                    |
| --------------------------- | ------------------------------------- | -------------------------------------------------------- |
| Toolbar strip (`AXToolbar`) | 1512×52, at the top of the window     | Canvas scroll area starts directly below                 |
| Show sidebar                | 48×52 at x=100                        | Right of the window buttons                              |
| Go Back                     | 44×52 at x=148                        |                                                          |
| Board title                 | Group 79×52, text 72.5×21             | Editable title (`CRLMacTitleToolbarItem`)                |
| Content Hub                 | 44×52                                 | Sits flush against the left of the insert group          |
| Insert group                | 232×52, centred on the window         | Five items, see next row                                 |
| Insert buttons              | 38 pt tall, widths 43, 40, 39, 42, 38 | 6 pt gaps, 3 pt group side padding, 7 pt above and below |
| Share Board, New Board      | 41×52 each                            | Pinned right, 4 pt trailing margin                       |
| Board browser view switch   | Radio group 75×38, two 36×36 segments | Grid and List, 1 pt apart                                |
| Board browser search field  | 327×38                                |                                                          |

Resize rule (**Measured** at 1512 pt and 1100 pt window widths): leading items
keep their x positions, the insert group stays centred on the window (its
centre was x=756 and x=550), and the trailing items stay pinned to the right
edge with the same 4 pt margin.

Evidence for the implementation (**Derived**): toolbar item subclasses
`CRLMacToolbarItem`, `CRLMacMenuToolbarItem`, `CRLMacZoomMenuToolbarItem`,
`CRLMacSearchToolbarItem` and `CRLMacTitleToolbarItem`. The insert group is an
`AXGroup` holding four `AXButton` items and one `AXMenuButton` (Insert Shape),
which matches `NSToolbarItemGroup` (**Inferred**).

Toolbar material, corner radius, border and shadow: **Unknown**. They are drawn
by `NSToolbar`.

How to rebuild:

- SwiftUI: use `.toolbar { ToolbarItemGroup(placement: .principal) { … } }` for
  the centred group, `.navigation` for the leading items and `.primaryAction`
  for the trailing ones. The system supplies the material.
- AppKit: `NSToolbar` with `centeredItemIdentifiers` and an
  `NSToolbarItemGroup`.
- Web: a 52 px bar laid out as `grid-template-columns: 1fr auto 1fr`, so the
  middle group stays centred while the sides pin to the edges.

```css
:root {
  --bar-height: 52px;
  --tool-height: 38px;
  --tool-gap: 6px;
  --group-pad: 3px;
  --bar-trailing: 4px;
}
.toolbar {
  display: grid;
  grid-template-columns: 1fr auto 1fr;
  align-items: center;
  height: var(--bar-height);
}
.toolbar .tools {
  display: flex;
  gap: var(--tool-gap);
  padding-inline: var(--group-pad);
}
.toolbar .tools button {
  height: var(--tool-height);
  min-width: 38px;
}
.toolbar .trailing {
  justify-self: end;
  padding-right: var(--bar-trailing);
}
```

### 1.2 Floating canvas controls

Two clusters float over the canvas corners. They are views inside the window's
split view, not separate windows (**Measured**: one `AXWindow`, and the controls
are children of the split group).

| Element              | Frame (window 1512×891) | Offsets                                               |
| -------------------- | ----------------------- | ----------------------------------------------------- |
| Decrease zoom        | 28×28 at x=22           | 22 pt from the window's left edge                     |
| Zoom pop-up          | 46×28 at x=57           | 7 pt gap. Its value is the percentage, such as `100%` |
| Increase zoom        | 28×28 at x=110          | 7 pt gap                                              |
| Show Scene Navigator | 45×38 at x=142          | 4 pt gap, vertically centred on the zoom buttons      |
| Connectors toggle    | 28×28                   | Value `On` or `Off`                                   |
| Grid toggle          | 28×28, 7 pt right of it | Right edge 25 pt from the window's right edge         |

All of these have their bottom edge 21 pt above the window's bottom edge. After
resizing the window to 1100×700 the left cluster kept x=22, the right cluster
kept its 25 pt right inset, and both kept the 21 pt bottom inset
(**Measured**). The scroll bars' accessibility frames are 19 pt thick, so the
controls clear them by 3 pt at the bottom and 7 pt at the right.

Implementation (**Derived**): Swift class
`CRLMacFloatingZoomControlViewController` with lazy properties
`decrementZoomButton`, `incrementZoomButton`, `selectZoomPopUpButton`, a
`CRLMacZoomControlInternalSeparator` view and a constraint set named
`attachedToZoomControlConstraints` (bottom, trailing, width, height anchors).

Background shape, material, radius and shadow of the clusters: **Unknown**.

How to rebuild:

```css
:root {
  --float-inset-x: 22px;
  --float-inset-bottom: 21px;
  --float-btn: 28px;
  --float-gap: 7px;
}
.canvas-controls {
  position: absolute;
  bottom: var(--float-inset-bottom);
  display: flex;
  gap: var(--float-gap);
}
.canvas-controls.leading {
  left: var(--float-inset-x);
}
.canvas-controls.trailing {
  right: 25px;
}
.canvas-controls button {
  width: var(--float-btn);
  height: var(--float-btn);
}
```

```swift
canvas
  .overlay(alignment: .bottomLeading) {
    ZoomControl().padding(.leading, 22).padding(.bottom, 21)
  }
  .overlay(alignment: .bottomTrailing) {
    CanvasToggles().padding(.trailing, 25).padding(.bottom, 21)
  }
```

In SwiftUI on macOS 26, apply `.glassEffect()` to the cluster container for a
system material. On the web, use a translucent token with `backdrop-filter`.
The exact values for Freeform are **Unknown**.

### 1.3 Mini formatter and inspectors

The format bar that follows the selection is called the mini formatter
(**Derived** from class names): `CRLMacMiniFormatterViewController`,
`CRLMacRepTrackingMiniFormatterViewController` (tracks the selected object),
`CRLMacTextEditingMiniFormatterViewController`, `CRLMiniFormatterPositionHelper`
and `CRLMacMiniFormatterAnimator`. Its buttons are
`CRLMacMiniFormatterImageButton` and `CRLMacMiniFormatterTitleButton`. One data
provider per content type fills it: shape, shape fill, stroke, text, line,
table, cell, media, file, URL, USDZ, arrange and board item.

Each button opens an inspector. Inspectors are view controllers built by a
matching view builder: Arrange, Cell Fill, Connection Line, File, Image and
Movie, Line Ends, Shape Fill, Sticky Note Fill, Stroke, Table Layout, Table
Style, Text, Text Fill, URL and USDZ (`CRLMac…InspectorViewController`,
**Derived**). Palettes are separate views: `CRLMacColorPaletteView`,
`CRLMacLineEndPalette`, `CRLStrokePatternPalette` and
`CRLTableBorderStylePalette`.

`CRLMacInspectorPresenter` presents them in an `NSPopover` whose behaviour is
semitransient (**Derived**, `setBehavior:` with value 2 at `0x1011dc294` and
`0x1011dc850`).

Interaction rules found in code (**Derived**):

- A pinch that starts over the mini formatter does not zoom the canvas
  (`isMiniFormatterHitWithEvent:` in
  `-[CRLMacCanvasViewController magnifyWithEvent:]`).
- The formatter is told when a zoom begins and ends (log strings
  `RepTrackingViewHelper is beginning zoom operation` and `… ending zoom
  operation`), so it can hide or reposition during the zoom (**Inferred**).

Mini formatter size, offset from the selection, material, radius, shadow and
motion: **Unknown**. Selecting an object needs foreground pointer control, which
this session did not use.

How to rebuild: keep one small bar anchored to the selection's bounding box,
give each control its own popover, and make those popovers close when the user
clicks elsewhere in the same window but not when the app loses focus. In
SwiftUI use `.popover(isPresented:attachmentAnchor:arrowEdge:)`. On the web use
the Popover API with `popover="auto"` plus CSS anchor positioning.

### 1.4 HUDs

| HUD             | Evidence (**Derived**)                                                                                                                                                                                                                                                  |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Canvas text HUD | `CRLMacCanvasHUDWindow` (`ignoresMouseEvents`, `showAtPoint:`, `setHUDText:`) with a custom-drawn `CRLMacCanvasHUDView`. It is click-through                                                                                                                            |
| Image mask HUD  | `CRLMacImageHUDPanel` (an `NSPanel`) with a slider and buttons. Its controller owns a `glassView` property of type `NSGlassView`                                                                                                                                        |
| Path edit HUD   | `CRLMacPathHUDWindow.nib`                                                                                                                                                                                                                                               |
| Status HUD      | Swift `HUDView`. Slides in on `transform.translation.x` from 100 pt (mirrored for right-to-left) over 0.35 s, with a second 0.35 s animation whose key path was not resolved. Height animates to 56 pt in 0.15 s, ease-out. That this is the status HUD is **Inferred** |

`NSGlassView` is not in the public macOS 26.2 SDK headers. The public classes
are `NSGlassEffectView` and `NSGlassEffectContainerView`, and SwiftUI's
`glassEffect` modifier.

### 1.5 Find bar

The find bar is the only place where the main binary sets a visual-effect
material itself (**Derived**, `0x1011b22cc` and `0x1011b22d8`):
`NSVisualEffectView` with `material = .headerView` (raw value 10) and
`blendingMode = .withinWindow` (raw value 1).

How to rebuild: `NSVisualEffectView` with those two properties, or
`.background(.bar)` in SwiftUI.

### 1.6 Named colours

**Derived** from `Assets.car` with `rea inspect-asset-catalog`. Values are sRGB.

| Name                                 | Light     | Dark                  | Note                                        |
| ------------------------------------ | --------- | --------------------- | ------------------------------------------- |
| `AccentColor`                        | `#55BEF0` | `#5AC8F5`             | Tagged with system colour `systemCyanColor` |
| `gridItemLabelBackdropColor`         | `#F7F7F7` | `#404040`             | Board browser                               |
| `ContentHubCustomGradientStartColor` | `#E9ECF2` | `#30383C`             |                                             |
| `ContentHubCustomGradientEndColor`   | `#D3DCE3` | `#2F3438`             |                                             |
| `premiumFeatureTint`                 | `#9E6ED4` | `#BA84FF`             |                                             |
| `CRLShapeLibraryLineColor`           | `#000000` | `#FFFFFF`             | Shape library line art                      |
| `welcome_icon_border`                | `#C7C7C7` | `#FFFFFF` at 0% alpha |                                             |

Use your own accent token. For the cyan, use the system colour
(`NSColor.systemCyan`, `Color.cyan`) instead of the hex values.

## 2. Canvas interactions

### 2.1 Structure

The canvas is a `CRLCanvasView` (`NSView`) inside an `NSScrollView`, drawn
through `CRLCanvasLayer` with Metal (**Derived**: `-[CRLCanvasView
enclosingScrollView]`, `p_setEnclosingScrollViewZoomParameters`, `CAMetalLayer`
import). The live window shows one scroll area, 1512×839 pt, with both
scrollers, directly under the toolbar (**Measured**).

### 2.2 Pan

| Input                     | Behaviour                                                                                                    | Label        |
| ------------------------- | ------------------------------------------------------------------------------------------------------------ | ------------ |
| Two-finger scroll         | Handled by `NSScrollView`, so momentum and rubber-banding are the system's                                   | **Inferred** |
| Space, then drag          | `CRLMacPanDraggingMouseEventHandler`. The drag scrolls by the event delta divided by the view scale, rounded | **Derived**  |
| Momentum after space-drag | The handler makes no animation call, so the pan stops when the drag stops                                    | **Inferred** |

Cursor rules in that handler (**Derived**): key down sets the open-hand or the
closed-hand cursor, mouse down sets `NSCursor.closedHand`, and key up restores
`NSCursor.arrow`. That key down picks the closed hand only while the button is
down is **Inferred**. Panning also ends a collaboration follow session.

Momentum curve and rubber-band distance: **Unknown** (system behaviour, not
measured).

How to rebuild: on the web keep native scrolling or pointer-driven
`translate`, set `cursor: grab` while space is held and `cursor: grabbing`
while dragging, and divide pointer deltas by the zoom. In SwiftUI use a
`ScrollView` or `NSScrollView` so momentum stays native.

### 2.3 Zoom

| Property             | Value                                                                                                 | Label and evidence                                                                                             |
| -------------------- | ----------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| Minimum scale        | 0.1 (10%)                                                                                             | **Derived** `+[CRLInteractiveCanvasController minimumRecommendedViewScale]` at `0x10114d9f4`. **Measured** 10% |
| Maximum scale        | 4.0 (400%)                                                                                            | **Derived** `maximumRecommendedViewScale` at `0x10114da08`. **Measured** 400%                                  |
| Steps (detents)      | 10, 25, 50, 75, 100, 125, 150, 200, 300, 400%                                                         | **Derived** static array at `0x101b58b60`. **Measured** by pressing the zoom buttons through every step        |
| Step animation       | 0.2 s, ease-in-ease-out, `CABasicAnimation`                                                           | **Derived** `+[CRLCanvasLayer zoomAnimationDefaultCAAnimation]` and `zoomAnimationDefaultDuration`             |
| Overlay fade on zoom | 0.15 s out at start, 0.15 s in at end                                                                 | **Derived** `p_fadeOutForZoomOperation`, `p_fadeInForZoomOperation`                                            |
| Actual size          | Sets the scale to 1.0                                                                                 | **Derived** `-[CRLMacBoardViewController actualSize:]`                                                         |
| Zoom to fit content  | Union of all item frames, outset by 20, then animated                                                 | **Derived** `-[CRLInteractiveCanvasController zoomToFitAllContent]` (`CGRectInset` with −20)                   |
| Pinch                | `CRLCanvasZoomTracker`, anchored at the gesture location, with translation springs and a 0.2 s settle | **Derived** `magnifyWithEvent:` and `zoomWithScale:velocity:locationInView:phase:`                             |
| Pinch snapping       | Slow pinches snap to the detents. A modifier key changes this.                                        | **Inferred** from `setSlowPinchSnapsViewScale:` fed by `modifierFlags`, and `currentlySnappingViewScale`       |
| Smart magnify        | Fits the object under the pointer, clamped to the limits, 0.2 s. With no object it fits the width     | **Derived** `smartMagnifyWithEvent:`                                                                           |

Keyboard (**Derived** from the main menu nib):

| Command             | Shortcut | Action selector        |
| ------------------- | -------- | ---------------------- |
| Zoom In             | ⌘+       | `zoomIn:`              |
| Zoom Out            | ⌘−       | `zoomOut:`             |
| Actual Size         | ⌘0       | `actualSize:`          |
| Zoom to Selection   | ⇧⌘0      | `zoomToSelection:`     |
| Zoom to Fit Content | ⌥⇧⌘0     | `zoomToFitAllContent:` |

Spring parameters of the pinch settle and the zoom HUD appearance: **Unknown**.

How to rebuild:

```ts
const ZOOM_STEPS = [0.1, 0.25, 0.5, 0.75, 1, 1.25, 1.5, 2, 3, 4];
const nextStep = (z: number, up: boolean) =>
  up
    ? (ZOOM_STEPS.find((s) => s > z + 1e-6) ?? 4)
    : ([...ZOOM_STEPS].reverse().find((s) => s < z - 1e-6) ?? 0.1);
```

```css
.canvas-content {
  transform-origin: 0 0;
  transition: transform 200ms ease-in-out;
}
.canvas-overlay.zooming {
  opacity: 0;
  transition: opacity 150ms;
}
```

In SwiftUI, animate the scale with `.easeInOut(duration: 0.2)`. Freeform's
binary imports `Animation.easeInOut(duration:)` and `Animation.default`.

### 2.4 Selection, handles and rotation

| Pattern                 | Value                                                                                         | Label                                                                                                  |
| ----------------------- | --------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| Marquee                 | `CRLMacRectangularMarqueeEventHandler` with `CRLRectangularMarqueeTracker`                    | **Derived**                                                                                            |
| Marquee start animation | Bounds-size animation, 0.3 s, `cubic-bezier(0.1, 0.25, 0.25, 1)`, starting from an 80 pt size | **Derived** `p_beginMarqueePressAnimation` at `0x1001b7ff0`. The meaning of 80 is **Inferred**         |
| Handles appear and hide | Opacity fade, 0.15 s                                                                          | **Derived** `-[CRLCanvasRep fadeKnobsIn]`, `fadeKnobsOut`                                              |
| Rotate snapping         | The rotate handler reads the modifier keys and sets a snap threshold of 22.5°                 | **Derived** constant in `CRLMacRotateKnobDragEventHandler`. That Shift gives 45° steps is **Inferred** |
| Handle size and style   | **Unknown**                                                                                   |                                                                                                        |

Cursors (**Derived**, `NSCursor` selectors referenced by the binary): arrow,
I-beam, crosshair, open hand, closed hand, pointing hand, contextual menu, drag
copy, drag link, operation not allowed, disappearing item, and the six
directional resize cursors. Freeform also builds its own cursors for
`cursorForResizeType:`, `cursorForRotateType:`, moving, Bézier nodes and
background selection. The cursor is chosen per point and per handle
(`cursorAtPoint:forKnob:withCursorPlatformObject:`).

Hover visuals: **Unknown**.

How to rebuild: map each handle to a standard cursor (`nwse-resize`,
`ew-resize`, `grab`, `crosshair`), fade handles with a 150 ms opacity
transition, and round the rotation angle to 45° while Shift is held.

### 2.5 Snapping and alignment guides

Menu switches (**Derived** from the nib, View menu): Show Grid, Snap to Grid,
Show Connectors, and under Guides: Center Guides, Edge Guides, Relative Sizing,
Relative Spacing and Use Haptic Feedback.

| Property               | Value                                                                              | Label                                                        |
| ---------------------- | ---------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Guide kinds            | Edge alignment, centre alignment, equal spacing, equal sizing                      | **Derived** from `CRLCanvasGuideController` method names     |
| Snap distance          | 7 when grid snapping is off. With grid snapping on, the grid supplies the distance | **Derived** `snapDistanceToUseForGuides` at `0x1003b8e0c`    |
| Snap distance vs zoom  | Scaled by a `tanh` of the view scale, so it does not grow linearly with zoom       | **Derived** `p_maxSnapForSnapDistance:`                      |
| Guides disappear       | Opacity fade, 0.4 s, when the drag ends                                            | **Derived** `endAlignmentOperation`, `p_hideGuideRenderable` |
| Haptics                | `NSHapticFeedbackManager` on snap                                                  | **Inferred** from the import and the menu item               |
| Unit of the 7          | Screen points                                                                      | **Inferred**                                                 |
| Guide colour and width | **Unknown**                                                                        |                                                              |

How to rebuild: collect candidate edges and centres of visible objects, snap
when the dragged edge is within 7 px in screen space, draw the guide line, and
fade it out over 400 ms on pointer up. On Apple platforms use
`NSHapticFeedbackManager.defaultPerformer.perform(.alignment, …)` at the moment
of snapping.

## 3. Fly-out drawers and popovers

Every fly-out found in the binary is an `NSPopover` (**Derived**). Raw values
are `NSPopover.Behavior` (1 transient, 2 semitransient) and `NSRectEdge`
(1 minY, 2 maxX, 3 maxY).

| Popover                                    | Behaviour                                      | Preferred edge | Evidence                                                     |
| ------------------------------------------ | ---------------------------------------------- | -------------- | ------------------------------------------------------------ |
| Shape library (Insert Shape)               | Transient                                      | 2              | `CRLShapeLibraryPopoverHelper`, `0x100f552b4`, `0x100f5584c` |
| Mini shape picker                          | Transient                                      | 2              | `MiniShapePickerNSView`, `0x1006f99d8`                       |
| Image smart-crop options                   | Transient, `animates = true`                   | 3              | `0x1006a03b0`, `p_showCropSuggestionPopover`                 |
| Scenes                                     | Not resolved                                   | 1              | `CRLScenesViewController`                                    |
| Board-level popovers and the iCloud prompt | Transient                                      | 1              | `CRLMacBoardViewController`, `0x1010c07ec`                   |
| Feature hint                               | Transient                                      | Not resolved   | TipKit `TipNSPopover`, `0x1010c448c`                         |
| Inspectors and text mini formatter extras  | Semitransient                                  | Not resolved   | `CRLMacInspectorPresenter`                                   |
| Generic popover buttons                    | Semitransient, created with `animates = false` | Not resolved   | `CRLMacPopoverButtonCell`, `0x10019d160`                     |

The preferred edge is a request. AppKit moves the popover to another edge when
there is no room, and the live placement was not observed.

Dismissal and focus (**Inferred** from documented `NSPopover` behaviour, not
observed here):

- Transient: the system closes the popover when the user interacts with an
  element outside it.
- Semitransient: the system closes the popover when the user interacts with
  elements in the window that holds the popover's anchor view. Interaction
  elsewhere does not close it.
- AppKit does not document the exact set of closing interactions. Escape and
  app switching were not tested.
- Freeform listens for `NSPopoverWillCloseNotification` (**Derived**), so it
  cleans up state when the system closes a popover.

Motion: the open and close animation is the system `NSPopover` animation.
Duration, curve and origin are **Unknown**. One content transition inside the
shape popover runs for 0.125 s (**Derived**, purpose **Unknown**).

A reusable entrance animation exists in the binary
(**Derived**, `-[CALayer crl_addSpringScaleInAnimationWithDelay:]` at
`0x1003d7ed0`). Which views use it is **Unknown**.

| Part                  | Value                                                                                                                                                  |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| First animation       | `CABasicAnimation`, 0.2 s, custom timing with x control points 0.33 and 0.83. The key path (likely opacity) and the y control points were not resolved |
| Scale                 | `CASpringAnimation`, mass 1, stiffness 200, damping 35, duration = settling duration                                                                   |
| Spring in other terms | Natural frequency 14.14 rad/s, damping ratio 1.24 (overdamped, no overshoot), response 0.44 s                                                          |

How to rebuild:

```css
.popover {
  transform-origin: var(--anchor-x) var(--anchor-y);
  animation: pop-in 200ms cubic-bezier(0.33, 0, 0.83, 1) both;
}
@keyframes pop-in {
  from {
    opacity: 0;
    transform: scale(0.9);
  }
}
```

The `0.9` start scale and the two y control points are placeholders, not
Freeform values. In SwiftUI the nearest spring is
`.spring(response: 0.44, dampingFraction: 1)`. Use the web Popover API
(`popover="auto"`) for transient behaviour (light dismiss and Escape) and
`popover="manual"` with your own outside-click rule for the semitransient kind.

## 4. Tool switching

Freeform on macOS has no persistent tool mode in its toolbar
(**Measured** and **Derived**):

- The board toolbar exposes four plain buttons and one menu button. None of
  them is a radio button, and none reports a selected value. Each press inserts
  an object or opens the shape library.
- What a drag does is decided by what is under the pointer. The binary has one
  event handler class per case: default mouse, object drag, resize handle,
  rotate handle, line and connection handles, marquee, space-drag pan, path
  edit, table cell selection and mask edit (`CRLMac…EventHandler`).
- The two mode switches are the Connectors and Grid toggles in the bottom-right
  cluster. They report `On` or `Off`. The menu equivalents are View > Show
  Connectors (`enableConnectorMode:`) and View > Show Grid
  (`enableCanvasDotGrid:`).

Keyboard (**Derived**): the main menu has 73 key equivalents and none is a bare
letter for a tool. The ones that matter for a canvas app:

| Command                   | Shortcut      |
| ------------------------- | ------------- |
| Group, Ungroup            | ⌥⌘G, ⌥⇧⌘G     |
| Lock, Unlock              | ⌘L, ⌥⌘L       |
| Bring to Front, Forward   | ⇧⌘F, ⌥⇧⌘F     |
| Send to Back, Backward    | ⇧⌘B, ⌥⇧⌘B     |
| Duplicate Selection       | ⌘D            |
| Select Parent             | ⌘Return       |
| Deselect All              | ⇧⌘A           |
| Copy Style, Paste Style   | ⌥⌘C, ⌥⌘V      |
| Jump to Selection         | ⌘J            |
| Add Scene, Next, Previous | ⇧⌘S, ⌥⌘], ⌥⌘[ |
| Edit Mask                 | ⇧⌘M           |
| Show Sidebar              | ⌃⌘S           |

Insert Sticky Note, Shape, Text Box, Table, Line and Link have menu items
without shortcuts.

Button states (normal, hover, pressed, selected) and any tool transition:
**Unknown**. The buttons are standard toolbar items, so AppKit draws the states.

How to rebuild: treat inserts as one-shot commands that return to the default
pointer, pick the drag behaviour from the hit target, and keep real modes
(connectors, grid) as explicit toggles with `aria-pressed`. In SwiftUI use
`Button` for inserts and `Toggle(isOn:)` with `.toggleStyle(.button)` for the
modes.

## What is not measured

These need Screen Recording permission for the terminal and Freeform in the
foreground with pointer control:

- Material, tint, corner radius, hairline and shadow of the toolbar, the
  floating clusters, popovers, the mini formatter and the HUDs, in Light and
  Dark.
- The mini formatter's size and its offset from the selection.
- Popover open and close timing, curve and origin, and the stacking of a
  popover opened from another popover.
- Scroll momentum, rubber-banding and pinch settle curves.
- Hover, pressed and selected visuals of toolbar and floating buttons.
- Handle sizes, selection outline and guide line styling.
- Light appearance. The host was in Dark and the session did not change it.

To finish the pixel pass: grant Screen Recording to the terminal, bring
Freeform to the front on an empty board, then capture with `screencapture -l
<window id>` for stills and `screencapture -v` for motion, and step the
recording with `ffmpeg`. At 120 Hz each frame is 8.3 ms, so state timings as
±1 frame.

## Reproduce the static evidence

```sh
APP=/System/Applications/Freeform.app
rea decode-interface-builder "$APP" --format json --full-output > ib.json
rea inspect-asset-catalog "$APP" --format json --full-output > assets.json
lipo -thin arm64e "$APP/Contents/MacOS/Freeform" -output /tmp/Freeform.arm64e
dyld_info -objc /tmp/Freeform.arm64e > objc.txt
llvm-objdump -d --start-address=0x10114d9f4 --stop-address=0x10114da10 \
  /tmp/Freeform.arm64e
```

The thin copy is a local working file. Delete it afterwards and do not
redistribute it.
