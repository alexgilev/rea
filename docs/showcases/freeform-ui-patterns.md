# Freeform UI patterns: panels, canvas, popovers and tools

A design study of Apple Freeform on macOS, made with REA and standard macOS
tooling. It records measurements and behaviour so that similar patterns can be
re-implemented from scratch. It contains no Apple code, images, icons or sounds.
[DISCLAIMER.md](../../DISCLAIMER.md) applies.

**Status: Light appearance complete, Dark pixels open.** Static evidence, live
accessibility measurements and a pixel and motion pass in Light appearance are
done. Pixel values in Dark appearance, trackpad gestures (pinch, momentum,
rubber-banding) and a few secondary flows are not measured. Every such value is
marked **Unknown** below. See [What is not measured](#what-is-not-measured).

## Target and method

| Item           | Value                                                                                                                                                          |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Target         | `/System/Applications/Freeform.app`, version 4.5, `com.apple.freeform`                                                                                         |
| Binary         | `Contents/MacOS/Freeform`, arm64e slice, SHA-256 of the file `e567ee43…`                                                                                       |
| Host           | macOS 26.6.2 (25G83), 1512×982 pt display at 2x, 120 Hz                                                                                                        |
| Appearance     | Accessibility layout pass in Dark (2026-11-16). Pixel and motion pass in Light (2026-11-17). Reduce Transparency, Reduce Motion and Increase Contrast were off |
| REA            | `rea-agents` 6.9.0 CLI, run on Node.js 24.22.3                                                                                                                 |
| Analysis dates | 2026-11-16 and 2026-11-17                                                                                                                                      |
| Native engines | None installed (no Hopper, Ghidra or IDA)                                                                                                                      |

Every value carries one of these labels.

| Label        | Meaning                                                                                                                               |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Measured** | Read from the running app: an accessibility frame (`AXUIElement`), a pixel in a window capture, or a frame time in a screen recording |
| **Derived**  | Read from the shipped bundle: a nib field, an asset record or a constant in code                                                      |
| **Inferred** | A conclusion drawn from names or from documented AppKit behaviour, not observed                                                       |
| **Unknown**  | Not established. Do not treat a guess from another app as a Freeform measurement                                                      |

Rules for **Measured** pixel and motion values:

- Lengths are in points. Captures are 2x, so the resolution is 0.5 pt.
- Colours are sRGB hex values after converting the capture from the display
  profile. All of them were sampled in Light appearance.
- A colour sampled on a translucent surface depends on what is behind it. Such
  values name their backdrop and are samples, not tokens.
- Recordings deliver 60 frames per second, so a time is the span from the first
  changed frame to the first settled frame, ±17 ms.
- A shadow's blur, offset and opacity and the fit of an easing curve are
  **Inferred** from the measured profile.

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
  window.
- `screencapture -l <window id>` and `screencapture -R` for stills, and
  `screencapture -v` for recordings, limited to the Freeform window. `sips`
  converted stills to sRGB, `ffmpeg` and `ffprobe` produced raw pixels and
  frame times, and short Python scripts read them.
- Accessibility presses for every control that accepts them, and synthetic
  pointer events (`CGEventPost`) for hover, selection, drags and scrolling. All
  interaction happened on a new scratch board that holds two default shapes.
  No capture is stored in this repository.

Two limits apply to every **Derived** code constant:

- Only the main executable was disassembled. The five embedded frameworks
  (`TSUtility`, `TSMediaLibrary`, `TSImageGeneration`, `TSAccessibility` and
  `AppsGen`) were not.
- The binary is stripped. An Objective-C method has an exact name. A Swift
  function has none, so it is named after the nearest preceding Objective-C
  method. Where this document ties a constant to a class through a Swift
  function, the constant is **Derived** and the class name is **Inferred**.
  This applies to the popover table in section 3 (every row without a named
  Objective-C method), the inspector popover behaviour, the status HUD and
  the 0.125 s shape-popover animation.

Tools that did not work on this host:

- `rea inspect-macho` and `rea observe-native-ui` compile a Swift helper. The
  Command Line Tools Swift compiler fails here with
  `redefinition of module 'SwiftBridging'`.
- Xcode is not installed, so Accessibility Inspector and Quartz Debug were not
  available.
- Pointer events sent straight to the process (`CGEventPostToPid`) are ignored
  by Freeform's canvas and popovers. Real pointer events were needed.
- Synthetic events cannot produce trackpad pinch or scroll momentum.

## 1. Floating panels

Freeform draws almost all chrome with system components. The main binary
imports `NSToolbar`, `NSToolbarItemGroup`, `NSMenuToolbarItem`,
`NSSearchToolbarItem`, `NSTrackingSeparatorToolbarItem`, `NSPopover`, `NSPanel`,
`NSSplitViewController`, `NSScrollView`, `NSVisualEffectView` and `NSGlassView`
(**Derived**, `nm -u`). The asset catalog holds 33 named colours, 25 of which
belong to the app icon (**Derived**). It also imports SwiftUI's
`View.glassEffect(_:in:)`, `Glass.regular` and `DefaultGlassEffectShape`
(**Derived**). Panel materials, tints, hairlines and shadows therefore come
from AppKit and SwiftUI, and the clean way to reproduce them is to use the
matching public API. The pixel values below describe what that system material
looks like in Light appearance over Freeform's white canvas.

Every floating surface measured has the same build (**Measured**): a capsule
whose end caps are half circles, a near-white fill, a 1 pt white rim, and a
soft shadow that is darker below than above.

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
which matches `NSToolbarItemGroup` (**Inferred**). The class names are **Derived** and their mapping to the visible items is **Inferred**.

Appearance (**Measured**, Light). `NSToolbar` on macOS 26 draws all of this, so
treat the values as a description of the system component:

| Property         | Value                                                                                           |
| ---------------- | ----------------------------------------------------------------------------------------------- |
| Bar background   | None. The canvas and its dot grid continue under the toolbar                                    |
| Scroll-edge wash | `#FCFCFC` across the 52 pt bar, then `#FDFDFD` at 65 pt, `#FEFEFE` at 80 pt, white by 100 pt    |
| Item containers  | Glass capsules 36 pt tall, 8 pt above and below, 8 pt between neighbours                        |
| Capsule widths   | Sidebar 39.5, Go Back 36 (a circle), Content Hub 36 (a circle), insert group 224, trailing 70.5 |
| Trailing inset   | The trailing capsule ends 9.5 pt from the window's right edge                                   |
| Capsule fill     | `#FEFEFE` over the `#FCFCFC` wash                                                               |
| Capsule rim      | 1 pt, `#FFFFFF`                                                                                 |
| Capsule shadow   | `#F7F7F7` at the top edge, `#F1F1F1` at the bottom edge, gone about 25 pt below                 |
| Icon colour      | `#484848`                                                                                       |
| Hover            | A pill about 33×28 pt behind the icon, `#F1F1F1`                                                |
| Pressed          | The same pill, `#E5E5E5`                                                                        |

The accessibility frames in the first table are hit areas. The visible capsules
are smaller.

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

Appearance (**Measured**, Light, over the white canvas):

| Property                     | Value                                                                                                                                    |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Container                    | One glass capsule per cluster, 36 pt tall                                                                                                |
| Insets                       | 17 pt from the window's side edge and 17 pt from its bottom edge                                                                         |
| Widths                       | Left cluster 169 pt, right cluster 78 pt                                                                                                 |
| End caps                     | Half circles, radius 18 pt. Thirteen rays from the cap centre found the rim at 17.5 to 18.1 pt                                           |
| Padding                      | 4 pt above and below the 28 pt buttons. 5 pt before the first button of the left cluster, 7.5 pt on both sides of the right cluster      |
| Fill                         | `#FDFDFD`. Grid dots behind it show at about 1% contrast, so it is close to opaque here                                                  |
| Rim                          | 1 pt, `#FFFFFF`                                                                                                                          |
| Divider                      | 1 pt wide, 20 pt tall, `#E4E4E4`, vertically centred                                                                                     |
| Glyph and label              | Glyphs `#484848`, the percentage label `#272727`                                                                                         |
| Shadow samples               | Canvas darkened by 2.0% at the top edge, 2.7% at the sides, 4.3% at the bottom edge, 0 about 22 pt above                                 |
| Shadow fit                   | Black at about 6% opacity, 8 pt down, Gaussian sigma about 16 pt (**Inferred**)                                                          |
| Hover                        | No visible change                                                                                                                        |
| Pressed                      | A square-cornered block about 27×32 pt behind the glyph, `#F0F0F0`                                                                       |
| Toggle on (Grid, Connectors) | Only the glyph changes. Connectors swaps outlined nodes for filled ones, and Grid darkens its inner dots. The background does not change |

The Zoom pop-up opens a standard `NSMenu`, 164×309 pt (**Measured**). Its left
edge is 3 pt right of the button's left edge and its bottom edge overlaps the
button's top by 1 pt. It lists the ten zoom steps with a check mark on the
current one, then a separator, Zoom to Selection and Zoom to Fit Content. It
appears within one frame and fades out over about 0.2 s, which is the system
menu's behaviour.

Scene Navigator is a drawer inside the left capsule, not a popover
(**Measured**). Pressing the button widens the capsule by 119 pt and reveals
Previous Scene, Show Scene List and Next Scene (28×28 at x=154, 188 and 222,
6 pt apart) and a second divider. The toggle moves from x=142 to x=261. Section
3 has the motion.

How to rebuild:

```css
:root {
  --float-inset: 17px;
  --float-height: 36px;
  --float-btn: 28px;
  --float-gap: 7px;
  --float-pad-y: 4px;
  --float-pad-x: 5px;
  --float-fill: rgb(253 253 253 / 0.96);
  --float-rim: #fff;
  --float-shadow: 0 8px 32px rgb(0 0 0 / 0.06);
  --float-pressed: rgb(0 0 0 / 0.05);
}
.canvas-controls {
  position: absolute;
  bottom: var(--float-inset);
  display: flex;
  align-items: center;
  gap: var(--float-gap);
  height: var(--float-height);
  padding: var(--float-pad-y) var(--float-pad-x);
  border-radius: 999px;
  background: var(--float-fill);
  backdrop-filter: blur(20px);
  box-shadow:
    inset 0 0 0 1px var(--float-rim),
    var(--float-shadow);
}
.canvas-controls.leading {
  left: var(--float-inset);
}
.canvas-controls.trailing {
  right: var(--float-inset);
}
.canvas-controls button {
  width: var(--float-btn);
  height: var(--float-btn);
}
.canvas-controls button:active {
  background: var(--float-pressed);
}
```

The fill opacity, the backdrop blur radius and the shadow are **Inferred**
stand-ins for the system material. The sizes are **Measured**.

```swift
canvas
  .overlay(alignment: .bottomLeading) {
    ZoomControl()
      .padding(.vertical, 4).padding(.horizontal, 5)
      .glassEffect(.regular, in: .capsule)
      .padding([.leading, .bottom], 17)
  }
  .overlay(alignment: .bottomTrailing) {
    CanvasToggles()
      .padding(.vertical, 4).padding(.horizontal, 7.5)
      .glassEffect(.regular, in: .capsule)
      .padding([.trailing, .bottom], 17)
  }
```

`glassEffect(_:in:)` is the modifier Freeform's binary imports. In AppKit the
public class is `NSGlassEffectView`. The system then supplies the fill, rim and
shadow, so do not copy the pixel values when you use it.

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

`CRLMacInspectorPresenter` presents them in an `NSPopover`. Two `setBehavior:` calls with value 2 (semitransient) sit at `0x1011dc294` and `0x1011dc850`, in Swift functions next to that presenter's methods. The constant is **Derived** and the link to the presenter is **Inferred**.

Interaction rules found in code (**Derived**):

- A pinch that starts over the mini formatter does not zoom the canvas
  (`isMiniFormatterHitWithEvent:` in
  `-[CRLMacCanvasViewController magnifyWithEvent:]`).
- The formatter is told when a zoom begins and ends (log strings
  `RepTrackingViewHelper is beginning zoom operation` and `… ending zoom
  operation`), so it can hide or reposition during the zoom (**Inferred**).

Geometry and behaviour (**Measured**, Light):

| Property      | Value                                                                                                                                                                                                                            |
| ------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| When it shows | After a click on an object. It did not show for an object that was only inserted, and it hides during a drag                                                                                                                     |
| Appear timing | Starts about 0.11 s after the selection outline draws, then fades in over about 0.08 s                                                                                                                                           |
| Shape         | Glass capsule, 32 pt tall, end-cap radius 16 pt                                                                                                                                                                                  |
| Width         | 24 pt per item plus 5 pt at each end. 82 pt for 3 items, 154 pt for 6 items                                                                                                                                                      |
| Items         | 24×24 pt buttons, 4 pt above and below                                                                                                                                                                                           |
| Position      | Centred horizontally on the selection's bounding box, 14 pt above its top edge                                                                                                                                                   |
| Shape variant | Fill (a 14 pt dot in the current fill colour), Stroke, Text                                                                                                                                                                      |
| Text variant  | Bold, alignment, list, font size, text colour, more. Shown while a shape's text is being edited                                                                                                                                  |
| Material      | Glass that takes on the backdrop. `#FDFDFD` with a `#FFFFFF` rim over the white canvas, `#B6FDFF` over a `#59C4F6` shape. Where the capsule straddled the shape's edge, the blue tint bled a few points into the part over white |

The Fill button opens a popover with its arrow pointing down at the button
(**Measured**): window 224×153 pt including the arrow and shadow margin,
twelve colour swatches in a 6×2 grid at a
30 pt pitch with 32 pt hit frames, a 137×26 pt No Fill button and a 22×26 pt
colour picker well. Section 3 has its motion.

How to rebuild: keep one small bar anchored 14 px above the selection's
bounding box, size it from its item count, hide it while the pointer drags,
and give each control its own popover. In SwiftUI use
`.popover(isPresented:attachmentAnchor:arrowEdge:)`. On the web use the Popover
API with `popover="auto"` plus CSS anchor positioning.

```css
.mini-formatter {
  --item: 24px;
  position: absolute;
  bottom: calc(100% + 14px);
  left: 50%;
  translate: -50% 0;
  display: flex;
  height: 32px;
  padding: 4px 5px;
  border-radius: 999px;
  animation: fade-in 80ms 110ms both;
}
.mini-formatter > button {
  width: var(--item);
  height: var(--item);
}
@keyframes fade-in {
  from {
    opacity: 0;
  }
}
```

### 1.4 HUDs

| HUD             | Evidence (**Derived**)                                                                                                                                                                                                                                                                                               |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Canvas text HUD | `CRLMacCanvasHUDWindow` (`ignoresMouseEvents`, `showAtPoint:`, `setHUDText:`) with a custom-drawn `CRLMacCanvasHUDView`. It is click-through                                                                                                                                                                         |
| Image mask HUD  | `CRLMacImageHUDPanel` (an `NSPanel`) with a slider and buttons. Its controller owns a `glassView` property of type `NSGlassView`                                                                                                                                                                                     |
| Path edit HUD   | `CRLMacPathHUDWindow.nib`                                                                                                                                                                                                                                                                                            |
| Status HUD      | Swift `HUDView`. Slides in on `transform.translation.x` from 100 pt (mirrored for right-to-left) over 0.35 s, with a second 0.35 s animation whose key path was not resolved. Height animates to 56 pt in 0.15 s, ease-out. That these constants belong to `HUDView`, and that it is the status HUD, is **Inferred** |

The position HUD that follows a drag (**Measured**, Light): a 126×26 pt box
for the text `x: 565 pt  y: 247 pt`, white text of about 13 pt, corner radius
about 4 pt. It read `#212121` over the white canvas and `#0D1920` over a
`#59C4F6` shape, which solves to black at 87% opacity. Its left edge sat about
16 pt right of the pointer and its bottom edge about 20 pt above it. Which
class in the table draws it is **Inferred** to be the canvas text HUD.

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

Canvas surface (**Measured**, Light, 100% zoom):

| Property         | Value                                                                            |
| ---------------- | -------------------------------------------------------------------------------- |
| Background       | `#FFFFFF`                                                                        |
| Dot grid         | Round dots 2 pt across, `#BFBFBF`, 25 pt apart on both axes                      |
| Grid off         | The dots disappear within one frame. There is no fade                            |
| Default shape    | 150×150 pt, fill `#59C4F6`, placed at the centre of the view and selected        |
| Next insert      | Offset 50 pt right and 50 pt down from the previous one                          |
| Insert animation | The new object first appears at about 96% scale and reaches 100% in about 0.16 s |

How to rebuild the grid:

```css
.canvas {
  background:
    radial-gradient(circle, #bfbfbf 1px, transparent 1.2px) 0 0 / 25px 25px,
    #fff;
}
```

### 2.2 Pan

| Input                         | Behaviour                                                                                                                                                                 | Label        |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------ |
| Two-finger scroll             | Handled by `NSScrollView`, so momentum and rubber-banding are the system's                                                                                                | **Inferred** |
| Space, then drag              | `CRLMacPanDraggingMouseEventHandler`. The drag scrolls by the event delta divided by the view scale, rounded                                                              | **Derived**  |
| Momentum after space-drag     | The handler makes no animation call, so the pan stops when the drag stops                                                                                                 | **Inferred** |
| Wheel scroll, pixel units     | A synthetic pixel-unit scroll event panned the canvas within one frame. There is no smoothing animation. The distance was not checked against the event delta             | **Measured** |
| Wheel scroll with ⌘ or ⌥ flag | A synthetic scroll event carrying the ⌘ or the ⌥ flag, with no key physically held, panned like a plain one and left the zoom at 100%. A real modifier key was not tested | **Measured** |

Cursor rules in that handler (**Derived**): key down sets the open-hand or the
closed-hand cursor, mouse down sets `NSCursor.closedHand`, and key up restores
`NSCursor.arrow`. That key down picks the closed hand only while the button is
down is **Inferred**. Panning also ends a collaboration follow session.

Momentum curve and rubber-band distance: **Unknown**. They are `NSScrollView`
behaviour, and synthetic events cannot produce trackpad momentum.

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

The step animation was also recorded (**Measured**). One press of Increase zoom
at 100% scaled a centred shape from 150 pt to 187.5 pt wide, symmetrically
about the centre of the view. The first changed frame and the settled frame
were 0.175 s apart going in and 0.19 s apart going out. Progress per frame was
3, 5, 11, 24, 37, 50, 66, 76, 87, 97 and 100%, which is an ease-in-ease-out shape
and agrees with the 0.2 s constant.

Spring parameters of the pinch settle and the zoom HUD appearance: **Unknown**.
Pinch was not exercised.

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

| Pattern                 | Value                                                                                                                                                                                            | Label                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------ |
| Marquee                 | `CRLMacRectangularMarqueeEventHandler` with `CRLRectangularMarqueeTracker`                                                                                                                       | **Derived**                                                                                            |
| Marquee start animation | Bounds-size animation, 0.3 s, `cubic-bezier(0.25, 0.1, 0.25, 1)` (Core Animation's default curve, the same as CSS `ease`), starting from an 80 pt size                                           | **Derived** `p_beginMarqueePressAnimation` at `0x1001b7ff0`. The meaning of 80 is **Inferred**         |
| Handles appear and hide | Opacity fade, 0.15 s                                                                                                                                                                             | **Derived** `-[CRLCanvasRep fadeKnobsIn]`, `fadeKnobsOut`                                              |
| Rotate snapping         | The rotate handler reads the modifier keys and sets a snap threshold of 22.5°                                                                                                                    | **Derived** constant in `CRLMacRotateKnobDragEventHandler`. That Shift gives 45° steps is **Inferred** |
| Selection outline       | 1 pt line, `#2C70FF`, on the object's bounding box, centred on its edge                                                                                                                          | **Measured**, Light                                                                                    |
| Resize knobs            | Eight circles, at the corners and edge midpoints. 8 pt fill `#31ADE6`, 0.5 pt white ring, a faint dark halo about 1 pt wide                                                                      | **Measured**, Light                                                                                    |
| Shape control knob      | Same size, fill `#74E326`. On a rounded rectangle it sits on the top edge and sets the corner radius                                                                                             | **Measured** colour and size. Its function is **Inferred**                                             |
| Select and deselect     | A click draws or removes the outline and knobs within one frame. The 0.15 s fade was not seen on a click                                                                                         | **Measured**                                                                                           |
| Marquee look            | Fill `#DCDCDC` over the white canvas, with the dots visible through it. 1 pt border, `#F1F1F1`. Square corners                                                                                   | **Measured**, Light                                                                                    |
| Marquee timing          | First drawn about 0.1 s after the pointer starts to move with the button down, at pointer speeds of 40 and 440 pt/s. It then tracks the pointer exactly and vanishes within one frame on release | **Measured**. The 0.3 s press animation above was not seen with a mouse drag                           |
| During a drag           | Knobs and the mini formatter are hidden, and a position HUD follows the pointer (section 1.4)                                                                                                    | **Measured**                                                                                           |
| Text editing            | A double-click on a shape puts a caret at its centre, hides the knobs and switches the mini formatter to its text variant                                                                        | **Measured**                                                                                           |

Cursors (**Derived**, `NSCursor` selectors referenced by the binary): arrow,
I-beam, crosshair, open hand, closed hand, pointing hand, contextual menu, drag
copy, drag link, operation not allowed, disappearing item, and the six
directional resize cursors. Freeform also builds its own cursors for
`cursorForResizeType:`, `cursorForRotateType:`, moving, Bézier nodes and
background selection. The cursor is chosen per point and per handle
(`cursorAtPoint:forKnob:withCursorPlatformObject:`).

Cursors seen live (**Measured** through `NSCursor.currentSystemCursor`):

| Pointer is over                                        | Cursor                             |
| ------------------------------------------------------ | ---------------------------------- |
| Empty canvas, an object, a moving drag, a marquee drag | Arrow                              |
| Corner knob                                            | Diagonal two-headed resize arrow   |
| Top or bottom knob                                     | Vertical two-headed resize arrow   |
| Left or right knob                                     | Horizontal two-headed resize arrow |
| Text being edited                                      | I-beam                             |

A knob under the pointer has the same pixels as a knob at rest, so knobs have
no hover highlight and only the cursor changes (**Measured**). Hover over an
unselected object was not captured: **Unknown**.

How to rebuild: map each handle to a standard cursor (`nwse-resize`,
`ns-resize`, `ew-resize`), keep the default arrow on objects, show and hide
handles without a transition on click, and round the rotation angle to 45°
while Shift is held. In AppKit use `NSCursor.frameResize(position:directions:)`.

```css
:root {
  --select-line: #2c70ff;
  --knob-fill: #31ade6;
  --marquee-fill: rgb(0 0 0 / 0.137);
  --marquee-line: rgb(0 0 0 / 0.055);
}
.selection {
  outline: 1px solid var(--select-line);
  outline-offset: -0.5px;
}
.knob {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  background: var(--knob-fill);
  box-shadow:
    0 0 0 0.5px #fff,
    0 0 1.5px 0.5px rgb(0 0 0 / 0.25);
}
.marquee {
  background: var(--marquee-fill);
  outline: 1px solid var(--marquee-line);
}
```

The two marquee tokens reproduce the sampled colours over white. Whether
Freeform uses black with alpha or a grey is **Unknown**. The knob halo values
are **Inferred**. Replace the blues with your own accent tokens.

### 2.5 Snapping and alignment guides

Menu switches (**Derived** from the nib, View menu): Show Grid, Snap to Grid,
Show Connectors, and under Guides: Center Guides, Edge Guides, Relative Sizing,
Relative Spacing and Use Haptic Feedback.

| Property               | Value                                                                                                                                                                        | Label                                                        |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| Guide kinds            | Edge alignment, centre alignment, equal spacing, equal sizing                                                                                                                | **Derived** from `CRLCanvasGuideController` method names     |
| Snap distance          | 7 when grid snapping is off. With grid snapping on, the grid supplies the distance                                                                                           | **Derived** `snapDistanceToUseForGuides` at `0x1003b8e0c`    |
| Snap distance vs zoom  | Scaled by a `tanh` of the view scale, so it does not grow linearly with zoom                                                                                                 | **Derived** `p_maxSnapForSnapDistance:`                      |
| Guide hide animation   | A 0.4 s opacity animation exists in the code that hides a guide                                                                                                              | **Derived** `endAlignmentOperation`, `p_hideGuideRenderable` |
| Haptics                | `NSHapticFeedbackManager` on snap                                                                                                                                            | **Inferred** from the import and the menu item               |
| Unit of the 7          | Screen points                                                                                                                                                                | **Inferred**                                                 |
| Guide colour and width | `#EECA02`, 1 pt                                                                                                                                                              | **Measured**, Light                                          |
| Guide length           | Each guide ran the full 150 pt extent of the two objects, which had identical frames once snapped. That a guide spans the union of differently sized objects is **Inferred** | **Measured** for this one case                               |
| Snap seen live         | A pointer 3 pt short of centre alignment on both axes left the object exactly centred, with one vertical and one horizontal centre guide                                     | **Measured**                                                 |
| Guide timing           | Drawn in the frame the object snaps. Removed within one frame when the alignment breaks and on mouse-up                                                                      | **Measured**. The 0.4 s fade above was not seen              |

How to rebuild: collect candidate edges and centres of visible objects, snap
when the dragged edge is within 7 px in screen space, draw a 1 px guide across
the aligned objects (the union is an assumption, see the table), and remove it as soon as the alignment breaks or the
pointer is released. On Apple platforms use
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
there is no room.

What the live app showed (**Measured**, Light):

| Fly-out                   | Kind                         | Size and placement                                                                  | Open                                    | Close                                           |
| ------------------------- | ---------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------- | ----------------------------------------------- |
| Shape library             | `NSPopover` (`AXPopover`)    | Content 300×505 pt. Below the toolbar button, centred on it, arrow pointing up      | Complete within one frame. No animation | Escape. Gone within one frame                   |
| Fill inspector            | `NSPopover`                  | Window 224×153 pt. Above the mini formatter, arrow pointing down at the Fill button | Grows out of the anchor in about 0.2 s  | Escape. Shrinks into the anchor in about 0.16 s |
| Zoom pop-up, Insert Media | `NSMenu`                     | 164×309 pt and 194×366 pt                                                           | Within one frame                        | Escape. Fades out over about 0.2 s              |
| Scene Navigator           | Inline drawer in the capsule | Widens the left floating capsule by 119 pt                                          | About 0.29 s, spring without overshoot  | Not timed                                       |

The shape library's window (326×531 pt) is 13 pt larger than its content on
every side, which leaves room for the arrow and the shadow. If the Fill
inspector has the same margin, its content is 198×127 pt (**Inferred**). The shape library opened without
animation both from an accessibility press and from a real click.

Shape library content (**Measured**, Light): body `#FAFAF9` over the white
canvas, a three-column grid of 85 pt tiles filled `#F2F2F7` with about 12 pt
between them, a 10 pt side inset and a 14 pt top inset, then two full-width
280 pt buttons (36 and 38 pt tall, 10 pt apart) for Draw with Pen and Browse
Content Hub. Twelve tiles are visible. Each tile is an accessibility button
named after its shape.

Fill inspector motion, from the changed-pixel bounding box per frame
(**Measured**):

| Time after first frame | 0 ms | 17  | 33  | 50  | 67  | 83  | 100 | 117 | 133 | 158 | 192   | 280 | 400 |
| ---------------------- | ---- | --- | --- | --- | --- | --- | --- | --- | --- | --- | ----- | --- | --- |
| Width, % of final      | 17   | 34  | 51  | 63  | 73  | 81  | 88  | 93  | 96  | 100 | 101.5 | 101 | 100 |

The popover scales from its anchor point, overshoots by about 1.5% and settles
by about 0.4 s. On close the same box shrinks to the anchor in about 0.16 s
with no overshoot.

Scene Navigator motion, from the position of the toggle glyph per frame
(**Measured**):

| Time after first frame | 8 ms | 25  | 50  | 67  | 83  | 100 | 117 | 133 | 150 | 167 | 183 | 200 | 233  | 292 |
| ---------------------- | ---- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | ---- | --- |
| Travel, % of 119 pt    | 3    | 10  | 26  | 39  | 51  | 61  | 70  | 77  | 82  | 87  | 91  | 93  | 97.5 | 100 |

A critically damped spring with a 0.3 s response predicts 10, 28, 52, 82 and
92% at 25, 50, 83, 150 and 200 ms, so that is the nearest standard curve
(**Inferred**). The start time is known to ±8 ms.

Dismissal and focus (**Inferred** from documented `NSPopover` behaviour, not
observed here):

- Transient: the system closes the popover when the user interacts with an
  element outside it.
- Semitransient: the system closes the popover when the user interacts with
  elements in the window that holds the popover's anchor view. Interaction
  elsewhere does not close it.
- AppKit does not document the exact set of closing interactions. Escape
  closed the shape library, the Fill inspector and both menus (**Measured**).
  App switching and the stacking of one popover on another were not tested.
- Freeform listens for `NSPopoverWillCloseNotification` (**Derived**), so it
  cleans up state when the system closes a popover.

Motion: the measured table above shows two cases. The toolbar's shape library
does not animate. The Fill inspector uses a scale from the anchor, which is
consistent with the system `NSPopover` animation on macOS 26 (**Inferred**).
A 0.125 s animation sits in a Swift function next to the shape popover's content controller (constant **Derived**, owner **Inferred**, purpose **Unknown**).

A reusable entrance animation exists in the binary
(**Derived**, `-[CALayer crl_addSpringScaleInAnimationWithDelay:]` at
`0x1003d7ed0`). Which views use it is **Unknown**. The measured insert
animation in section 2.1 (96% to 100% in about 0.16 s) may be its first part
(**Inferred**).

| Part                  | Value                                                                                                |
| --------------------- | ---------------------------------------------------------------------------------------------------- |
| First animation       | `CABasicAnimation`, 0.2 s, timing `cubic-bezier(0.33, 0, 0.83, 0.83)`. The key path was not resolved |
| Scale                 | `CASpringAnimation`, mass 1, stiffness 200, damping 35, duration = settling duration                 |
| Spring in other terms | Natural frequency 14.14 rad/s, damping ratio 1.24 (overdamped, no overshoot), response 0.44 s        |

How to rebuild:

```css
.popover {
  transform-origin: var(--anchor-x) var(--anchor-y);
  animation: pop-in 200ms cubic-bezier(0.2, 0.5, 0.4, 1) both;
}
.popover.closing {
  animation: pop-in 160ms ease-in reverse both;
}
@keyframes pop-in {
  from {
    opacity: 0;
    transform: scale(0.15);
  }
}
.drawer {
  /* Scene Navigator style: width change, no overshoot. */
  transition: width 290ms cubic-bezier(0.25, 0.4, 0.35, 1);
}
```

The durations and the scale-from-anchor origin are **Measured**. The two
`cubic-bezier` curves and the `0.15` start scale are **Inferred** fits to the
tables above, and the popover curve leaves out the 1.5% overshoot. In SwiftUI
use `.spring(response: 0.3, dampingFraction: 1)` for
the drawer, and leave popover motion to the system. Use the web Popover API
(`popover="auto"`) for transient behaviour (light dismiss and Escape) and
`popover="manual"` with your own outside-click rule for the semitransient kind.

## 4. Tool switching

Freeform on macOS shows no persistent tool mode in its toolbar:

- The board toolbar exposes four plain buttons and one menu button. None of them is a radio button, and none reports a selected value (**Measured**). Insert Shape opens the shape library, and choosing a shape there inserts it (**Measured**). That the other four each insert an object directly is **Inferred** from the action selectors, such as `mainMenuActionToInsertStickyNoteWithSender:`. A press sent through the Accessibility API while Freeform was in the background showed no visible result.
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

Live behaviour (**Measured**):

- Choosing Rounded Rectangle or Circle in the shape library closed the popover
  and placed the shape at the centre of the view, already selected. The pointer
  stayed the arrow and no tool stayed armed.
- Toolbar buttons show a hover pill (`#F1F1F1`) and a darker pressed pill
  (`#E5E5E5`). Floating buttons show no hover and a square pressed block
  (`#F0F0F0`). Sections 1.1 and 1.2 have the sizes.
- Grid and Connectors show their state in the glyph alone: filled nodes for
  Connectors on, dark inner dots for Grid on. Neither gets a tinted or
  selected background.
- Turning the grid off removes the dots within one frame.
- A press that is dragged off the button before release is cancelled. This
  held for a floating zoom button and for a toolbar insert button.

How to rebuild: treat inserts as one-shot commands that return to the default
pointer, pick the drag behaviour from the hit target, and keep real modes
(connectors, grid) as explicit toggles with `aria-pressed` that change the
icon, not the background. In SwiftUI use
`Button` for inserts and `Toggle(isOn:)` with `.toggleStyle(.button)` for the
modes.

## What is not measured

- Dark appearance pixels: every colour, fill, rim and shadow sample above is
  from Light. Layout frames were read in Dark and matched the Light captures.
- Trackpad gestures: pinch zoom, its snapping and settle, scroll momentum and
  rubber-banding. Synthetic events cannot produce them.
- Space-drag pan, rotation, resizing by a knob, and the cursors for those
  drags.
- Edge, spacing and sizing guides. Only centre guides were triggered.
- Popover stacking (a popover opened from another popover), dismissal by
  clicking outside and by app switching, and keyboard focus inside a popover.
- Stroke and Text inspectors, Content Hub, Share, sticky notes, tables, text
  boxes, connectors, the find bar and the image, path and status HUDs.
- Scene Navigator collapse timing and the scene list.
- The blur radius and tint of the glass material. The samples only show its
  result over two backdrops.
- Behaviour with Reduce Motion, Reduce Transparency or Increase Contrast on.

To add the Dark samples: switch the system to Dark, bring Freeform to the
front, and repeat the still captures. No pointer control is needed for the
toolbar, the floating clusters, the menus or the shape library, because
accessibility presses open them.

## Reproduce the evidence

Static:

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

Live, with Accessibility and Screen Recording granted to the terminal and
Freeform frontmost on a scratch board:

```sh
# Still of the Freeform window only, then sRGB pixels as raw RGBA.
screencapture -o -x -l "$WINDOW_ID" still.png
sips -m "/System/Library/ColorSync/Profiles/sRGB Profile.icc" still.png \
  --out still.srgb.png
ffmpeg -i still.srgb.png -f rawvideo -pix_fmt rgba still.rgba

# Recording of the window rectangle, then one timestamp per frame.
screencapture -x -v -V 5 -R0,33,1512,890 motion.mov
ffprobe -select_streams v:0 -show_entries frame=pts_time -of csv=p=0 motion.mov
```

Read lengths as runs of equal pixels along a row or column of the raw file,
and divide by the 2x scale. For motion, compare each frame with the first one
inside a fixed rectangle and note the first changed and first settled frames.
Keep the captures local and delete them afterwards. They show Apple's
interface and are not part of this document.
