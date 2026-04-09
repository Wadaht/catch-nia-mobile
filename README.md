# Catch Nia! — iOS Mobile Build

A landscape-only iOS port of the original `Catch Nia! - Dylans Version.html` game,
wrapped in [Capacitor](https://capacitorjs.com) so it runs as a native iOS app
through `WKWebView`. The original Three.js game logic is **unchanged** — a thin
mobile-input adapter (added at the bottom of `www/index.html`) translates touch
events into the synthetic keyboard/mouse events the game already listens for.

## 🚀 Install on your iPhone (no Mac needed)

This repo has a GitHub Actions workflow that builds an **unsigned `.ipa`** on a
free macOS runner. Sideloadly will re-sign it with your Apple ID and install it
on your iPhone over USB.

1. Go to the **Actions** tab of this repo on GitHub.
2. Click the **"Build unsigned IPA"** workflow → **"Run workflow"** → **Run**.
   (It also runs automatically on every push.)
3. Wait ~5 minutes for the green checkmark.
4. Open the workflow run, scroll to **Artifacts**, and download
   **`CatchNia-unsigned-ipa`** (a zip).
5. Unzip it to get `CatchNia-unsigned.ipa`.
6. Open **[Sideloadly](https://sideloadly.io)** on your PC, plug in your iPhone,
   drag the `.ipa` in, enter your Apple ID, and click **Start**.
7. On your iPhone: **Settings → General → VPN & Device Management** → trust
   your Apple ID profile. The game launches in landscape.

> **Free Apple ID note:** the app expires after 7 days and you'll need to
> re-sideload it. A paid Apple Developer account ($99/yr) gives you 1-year
> signing.

## What's in here

```
CatchNiaMobile/
├── www/
│   └── index.html              ← The game + mobile touch adapter
├── package.json                ← Capacitor dependencies
├── capacitor.config.json       ← App ID, name, iOS settings
├── ios-info-plist-additions.xml← Snippet to lock landscape (paste into Info.plist)
└── README.md                   ← You are here
```

## How the mobile adapter works

The original game uses **WASD + mouse + pointer-lock**. The adapter at the bottom
of `www/index.html` does this on iOS:

| Original control | Mobile control |
|---|---|
| `W A S D` | Virtual joystick (left half of screen) |
| Mouse look | Drag right half of screen |
| Left-click shoot | Tap right half (no drag) **or** the SHOOT button |
| `Space` jump | JUMP button |
| `Shift` sprint | SPRINT button (hold) |
| `E` interact / catch | CATCH button |
| `F` pickup weapon | PICK button |
| `B` shop | SHOP button |
| Click to lock pointer | Tap the start screen |

The adapter:

1. Detects iOS / touch devices and adds `class="is-mobile"` to `<html>`.
2. Overrides `requestPointerLock` so it just dispatches a `pointerlockchange`
   event — the game's existing handler then runs `startRound()` normally.
3. Forces `document.pointerLockElement` to always look "locked", so the
   game's existing `mousemove` and `mousedown` handlers keep working.
4. Dispatches synthetic `KeyboardEvent`s from joystick/buttons so the
   game's `keydown`/`keyup` listeners pick them up unmodified.
5. Auto-skips the multiplayer lobby (mobile = solo only) by calling
   `mpPlaySolo()` on load.
6. Hides the desktop crosshair and resizes HUD panels for small screens.
7. Locks gestures: no pinch zoom, no pull-to-refresh, no long-press menu.

## Build it for iOS

> **Requirements:** macOS, Xcode 15+, Node 18+, CocoaPods. Capacitor only
> supports building iOS apps from a Mac.

From the project root:

```bash
# 1. Install Capacitor + CLI
npm install

# 2. Add the iOS native shell (creates ios/App/...)
npx cap add ios

# 3. Sync the www/ folder into the iOS project
npx cap sync ios

# 4. Open Xcode
npx cap open ios
```

### Lock landscape orientation

After step 2 (`cap add ios`), open `ios/App/App/Info.plist` and paste the keys
from `ios-info-plist-additions.xml` inside the top-level `<dict>`. This is the
only manual step — it sets:

- `UIRequiresFullScreen = true`
- `UIStatusBarHidden = true`
- `UISupportedInterfaceOrientations` = landscape only (iPhone & iPad)

Then re-run `npx cap sync ios`.

### Run in the simulator

In Xcode: pick an **iPhone Pro / Pro Max** simulator (rotated to landscape via
`Cmd+→`) and hit ▶. Or:

```bash
npx cap run ios
```

### Ship to a real device / TestFlight

1. In Xcode → `App` target → **Signing & Capabilities** → set your Apple Team.
2. Update the bundle identifier (`com.catchnia.game`) if needed.
3. **Product → Archive** → upload to App Store Connect → distribute via TestFlight.

## Customizing

- **App icon / splash:** Replace the default Capacitor icons in
  `ios/App/App/Assets.xcassets/AppIcon.appiconset/`. Generate from the original
  game art with [`@capacitor/assets`](https://github.com/ionic-team/capacitor-assets).
- **Tweak touch button positions:** edit the `.tbtn` CSS rules at the bottom of
  `www/index.html` (search for `MOBILE TOUCH OVERLAY`).
- **Joystick sensitivity / look sensitivity:** edit `LOOK_SENS` and the joystick
  `max` value in the same script block.
- **Re-enable multiplayer:** delete the `.is-mobile #mp-lobby{display:none}` rule
  and remove the `autoSkipLobby()` call. Firebase still works inside `WKWebView`.

## Updating the game source

If `Catch Nia! - Dylans Version.html` ever updates, just re-copy it to
`www/index.html` and re-apply the mobile adapter block (everything below the
`<!-- MOBILE TOUCH OVERLAY -->` comment). Or diff this file against the
original to migrate.

## Notes & known limits

- **Performance:** Three.js runs fine in `WKWebView` on A12+ (iPhone XS and
  newer). Older devices may chug — drop the renderer pixel ratio and shadow
  quality in the original `init()` if you need to.
- **Audio:** iOS requires a user gesture to start audio. The game's first tap
  on the start screen counts, so SFX should work after the round begins.
- **Multiplayer:** Disabled on mobile by default (the lobby is hidden and we
  call `mpPlaySolo()` automatically). The Firebase code is still bundled,
  so you can re-enable it later.
- **Dev panel:** The hidden dev PIN (type `5432` during a round) still works
  — but on mobile there's no number key, so you'd need to wire it up manually
  if you want it.
- **App Store review:** This is a single-player game with no IAP, no ads, and
  no user-generated content. It should pass review with no extra metadata
  beyond age rating and a privacy policy stating "no data collected". Remove
  the dev PIN before submission if you want a clean build.
