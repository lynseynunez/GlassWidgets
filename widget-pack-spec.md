# Custom Widget Pack — Build Spec

A glassmorphism-styled Android widget pack. Every widget shares one visual
language (frosted glass card + floating gradient orbs) and one theming
system, so adding a new widget is a matter of building its data/interaction
layer, not restyling from scratch.

---

## 1. Core design system

### Visual language
- **Background**: `radial-gradient` dark base per widget category (each
  category gets its own two-color gradient, see per-widget notes below)
- **Floating orbs**: 2 soft, blurred, semi-transparent circles positioned
  off-canvas top-right and bottom-left, purely decorative, give depth
- **Glass card**: one `rgba(255,255,255,0.09)` panel, `backdrop-filter:
  blur(16px)`, `0.5px` translucent white border, `18px` border-radius,
  sits on top of the gradient+orbs
- **Sections within a card**: divided by `0.5px` hairlines, never separate
  floating cards — this is the "connected panel" pattern (confirmed after
  the music widget iteration)
- **Corners**: `18-22px` on outer containers, `8-12px` on inner tiles/rows

### Theming system — three axes, independent of each other

**1. Color mode** (how the accent color is chosen):
- `custom` — user manually picks one or a few accent colors, applied
  everywhere
- `per-app` — accent pulled from what the widget represents (e.g. music
  widget pulls from the actual album art via Android's **Palette API**;
  connectivity widgets could use brand colors)
- `wallpaper` — palette auto-extracted from the current wallpaper,
  re-sampled daily since the wallpaper rotates

**2. Style preset** (full look swap, confirmed via the clock widget):
- `classic` — navy/purple gradients, sans-serif, the default look used
  throughout this spec
- `fancy` — blush pink / lavender gradients, serif touches, softer glass
  (higher opacity, more blur)
- `tech` — teal/dark gradients, monospace font, same glass structure
  (NOT a full HUD/scanline redesign — the glass+orb language must survive
  the preset swap)

**3. Animation mode** — global toggle, per the utilities discussion:
- `static` — renders once, updates data silently, no motion
- `live` — subtle baked-in motion (twinkle, shimmer, marquee ticker) +
  faster refresh where the OS allows it

  > **Real constraint**: Android widgets (RemoteViews) have no native
  > animation loop and the OS throttles background refresh (~15–30 min
  > typical unless you run a foreground service). "Live" mostly means
  > baked-in AnimatedVectorDrawable loops, not a true real-time simulation.

### Shared component
Build one `GlassPanel` composable/view that takes: background gradient
colors, orb colors, and children. Every widget wraps its content in this.
Style presets swap the token set the panel reads (colors, font family),
never the panel's structure.

### Suggested stack
- **Kotlin + Jetpack Glance** (Android 12+) for the widget layer where
  possible — better suited to this than raw RemoteViews for anything with
  moderate interactivity
- Fall back to **RemoteViews + RemoteViewsService** for anything needing
  scrollable lists (multi-note swiping, long event feeds) or classic
  widget compatibility
- Games (tic-tac-toe, memory match, dice, coin flip) need real tap
  interactivity mid-widget — prototype these as Glance widgets with
  `actionRunCallback`; if that proves too limited, they may need to open
  as a lightweight transparent Activity instead of living purely inside
  the widget surface
- `RenderEffect`/`BlurMaskFilter` (API 31+) for the backdrop blur — decide
  your min-SDK based on this
- Palette API for per-app/album-art color extraction

---

## 2. Approved widget catalog

### Music (connected panel)
Album art fills the whole panel as the background (not a separate icon
tile). Track title/artist, source app badge (Spotify/YouTube Music/etc.
via `MediaSessionManager`), play/pause toggle, draggable seek bar with
elapsed/remaining time. Below a hairline: connected earbuds module —
device name (e.g. "Buds3 Pro"), L/R battery percentages.
- **APIs**: `MediaSessionManager` for playback + source app;
  Palette API for art→color extraction
- **Caveat**: per-bud battery for most brands (e.g. Samsung Buds) is not
  exposed via a public API — requires reverse-engineering the proprietary
  Bluetooth protocol (SAP for Samsung) or treating as a stretch goal

### Device info
Storage bar, battery health % (estimated, not from a public API — must be
derived from tracked charge cycles over time), uptime, Android version,
RAM. One connected panel, hairline-divided rows.

### Weather — two sizes
- **Small**: current temp, condition, feels-like, horizontal scroll strip
  of day cards (icon + high/low) for the week
- **Large**: current temp/condition/feels-like/high-low up top, wind +
  precipitation row, then a **vertical list** of all 7 days with
  condition icon, low, a proportional high/low range bar, and high
- **APIs**: any weather provider (OpenWeatherMap, etc.) + location
  permission
- Orb colors shift with condition (warm for sunny, cool for
  night/rain) — implementation detail, not yet fully specced

### Environment
UV index and air quality side by side, location at bottom.
- **APIs**: OpenWeatherMap Air Pollution API or Google Air Quality API +
  location permission

### Quote of the day
Single rotating quote, manual refresh icon. Local curated pool, no
network dependency required.

### Calculator — two sizes
- **Large**: full functional keypad (circular buttons), result display,
  operators tinted, equals as the one accent-filled button
- **Small**: icon + label launcher tile — tapping opens the **system**
  calculator app via an intent (no custom keypad logic needed here)

### Alarm clock
Shows the next enabled alarm, active weekdays highlighted, a toggle to
disable it without opening another app.
- **Constraint**: no public API to read/control the system Clock app's
  alarms — this widget must set/manage its own alarms via `AlarmManager`
  rather than reading another app's data

### World clock
Local time up top, user-configured list of other cities/zones below with
day-offset markers (`+1d`) when they cross midnight.

### Date / Calendar — three sizes, all sharing one header pattern
Header = big date number + weekday + that day's event feed (scrollable,
vertically centers when short, fades at the bottom edge when it overflows).
- **Small**: date number only, no events, small dot-strip showing
  position in the week
- **Medium**: header + a **week strip** below (day letters + numbers,
  tap any day to update the header)
- **Large**: header + a **full month grid** below instead of the week
  strip (same tap-to-update behavior). **Grid alignment bug to avoid**:
  the weekday header row and the date grid must share the exact same
  `grid-template-columns` — don't mix flexbox `space-between` for the
  header with CSS grid for the dates, they distribute space differently
  and misalign.
- **APIs**: Android Calendar Provider (read calendar permission) for the
  event feed; scrollable list needs `RemoteViewsService`

### Search
Compact pill: search field + mic + camera icons launching voice/image
search directly. No large-size variant needed (inherently a thin bar).

### Dice
Single die fills nearly the whole panel (minimal padding — this was
iterated multiple times, don't under-size it). Tap anywhere on the panel
to trigger a rapid face-shuffle animation (~700–800ms, cycling random
faces with a scale/rotate wobble) before landing on a result. "tap to
roll" label sits just beneath the die.

### Coin flip
Circular coin (must be visually distinct from the square dice — round
shape, not shared geometry). Flips **end-over-end** (vertical squash via
`scaleY`, not horizontal) between two vector faces:
- **Heads**: a traced head-profile silhouette (client-supplied path data,
  recolored white, kept purely as inline SVG — no image assets)
- **Tails**: a simple line-swirl
Both faces must be re-centered inside the coin circle after any path
changes — check bounding-box math, don't eyeball it.

### Tic-tac-toe
Fully playable 3x3 grid, X/O alternating turns, win/draw detection,
reset button. **AI toggle** in the header — when on, O plays automatically
after a short delay. Current AI is random-move only (not blocking/optimal)
— flagged as intentionally beatable; upgrade to minimax if a real
challenge is wanted later.

### Memory match
Icon-pair matching grid with **three difficulty modes** via pill buttons:
- Easy: 3 pairs, 3-column grid
- Medium: 8 pairs, 4-column grid
- Hard: 12 pairs, 4-column grid (6 rows — tall; decide whether the widget
  resizes per difficulty or the board scrolls within a fixed size)
Matched pairs stay revealed (dimmed), mismatches flip back after ~700ms.

### Screen time
Total time today up top, top 3 apps by usage with proportional bars.
Neutral copy only — no guilt-tripping framing.
- **Constraint**: requires `PACKAGE_USAGE_STATS`, which is **not** a
  normal runtime permission — user must manually enable it in
  Settings → Special App Access. Cannot be requested via a standard
  permission dialog. Plan onboarding accordingly.

### Clock — three style presets (classic / fancy / tech)
Large time display + date underneath. This is the reference widget for
how style presets should work: same `GlassPanel` structure and layout
across all three, only font-family and accent/gradient colors change.
- `classic`: navy/purple, sans-serif
- `fancy`: blush/lavender, serif touch on the numerals, sparkle accent
- `tech`: teal/dark, monospace — **glass+orb structure must be preserved**,
  this is not a full HUD redesign

### Compass
N/E/S/W dial with a rotating needle, heading in degrees + cardinal
abbreviation below.
- **APIs**: `Sensor.TYPE_MAGNETIC_FIELD` + accelerometer for tilt
  compensation
- **Constraint**: RemoteViews widgets can't run a continuous live sensor
  listener. A smoothly-tracking needle requires a background/foreground
  service feeding periodic updates; otherwise the compass only refreshes
  on tap/open rather than live-tracking while idle on the homescreen.

### Countdown (custom date)
Big day-count number, custom label, target date. Fully user-set, no
calendar integration. Multiple instances can coexist for tracking several
dates at once.

### Count-up (custom date)
Same visual shell as countdown, but counts up from a past date instead of
down to a future one (streaks, "days since," anniversaries). Built as a
**separate standalone widget**, not a toggle on the countdown widget.

### Notes (multi-note)
Dot indicators show position among multiple notes, swipe left/right to
navigate, `+` adds a new note, pencil opens the editor.
- **Constraint**: RemoteViews doesn't support live inline text editing.
  Tapping must launch a small transparent edit Activity/dialog; the
  widget then refreshes to show the saved text. Swiping between notes
  needs a `RemoteViewsService`-backed `StackView`/`ViewFlipper`.

---

## 3. Explicitly scrapped (do not build)
- Combo "utilities" panel (dark mode + volume + calculator together) —
  replaced by the standalone calculator; dark mode and volume toggles
  still need their own standalone widgets if wanted later
- Combo "productivity" panel (clipboard + notes) — notes was rebuilt
  standalone with multi-note support; clipboard was dropped entirely
- Combo "personalization" panel (greeting + day counter + custom note) —
  dropped entirely (day-counter concept lives on via countdown/count-up)
- Truth-or-dare — replaced by coin flip in the games lineup
- Pedometer — scrapped mid-iteration (ring-sizing kept missing the mark)
- Earbuds manager (standalone) — scrapped in favor of folding basic
  earbuds battery into the music widget instead
- The full "companion pet / tamagotchi" and "RPG stat HUD" character
  concepts from early exploration — abandoned in favor of the
  glassmorphism direction

## 4. Open ideas, not yet built
Games: rock-paper-scissors, magic 8-ball, spinner wheel, slot machine,
would-you-rather.
Utility: timer/stopwatch, unit converter, flashlight toggle, QR code
generator, currency converter.
System toggles: dark mode, volume, Wi-Fi, Bluetooth, airplane mode, DND,
brightness, battery saver.
Personal/reference: hydration tally, habit streak tracker, "been a
while" contact reminder, word of the day, trivia of the day, flashcard
deck, pomodoro timer.
Other: to-do list, shopping list, savings/budget tracker, quick-dial
contact, photo frame, news headlines, stock/crypto ticker, recipe of
the day.

---

## 5. Cross-cutting technical notes
- Min SDK should be chosen around **API 31 (Android 12)** — this is the
  floor for `RenderEffect`/blur and Jetpack Glance's more useful features
- Any widget needing scrollable content (notes, large calendar's event
  feed) needs `RemoteViewsService` + a collection widget, not a plain
  Glance/RemoteViews layout
- Several widgets depend on permissions that require manual user setup
  outside the normal runtime-permission flow (`PACKAGE_USAGE_STATS` for
  screen time) — audit all permission requirements before writing any
  onboarding flow
- No image assets required anywhere in the current spec — all icon/face
  art is inline vector (SVG path data or Tabler-style icon glyphs),
  per an explicit decision to avoid manual asset management
