# iOS + iPadOS simulator playbook

Web apps deserve real Mobile Safari testing, not just Playwright viewport
emulation. iPhone and iPad simulators are the cheapest way to surface
WebKit-specific behavior — `-webkit-fill-available`, viewport-fit notch,
soft-keyboard rect, IndexedDB quotas, PWA add-to-home heuristics, and
the dozen other things Chrome doesn't replicate.

This playbook covers two things:

1. **The universal core** — `xcrun simctl` for boot / openurl / screenshot
   / video. Works on any Mac with Xcode.
2. **A curated companion-skill ladder** — when you need real input
   (taps, swipes, gestures), point the agent at one of the excellent
   community skills the iOS folks have built. We do not duplicate their
   work; we orchestrate around them.

## Pre-flight

- macOS (Apple Silicon recommended).
- Xcode 15+ for the universal core. Xcode 26 for iOS 26 simulators.
- Detection snippet for the agent's first iOS-related action:
  ```bash
  if [[ "$(uname -s)" != "Darwin" ]]; then
    echo "iOS simulator path is macOS-only; falling back to browser playbook." >&2
    exit 1
  fi
  command -v xcrun >/dev/null || { echo "Xcode command-line tools missing." >&2; exit 1; }
  command -v axe       >/dev/null && HAS_AXE=1
  command -v baguette  >/dev/null && HAS_BAGUETTE=1
  command -v idb       >/dev/null && HAS_IDB=1
  ```

## The universal core — `xcrun simctl`

Bundled with Xcode. Covers the web-app QA happy path on iPhone or iPad
Mobile Safari without any additional install.

```bash
# Discover devices
xcrun simctl list devices
xcrun simctl list devices "iPhone 17 Pro"
xcrun simctl list devices "iPad Pro 13-inch"

# Boot (no-op if already booted)
xcrun simctl boot "iPhone 17 Pro"

# Open a URL in Mobile Safari
xcrun simctl openurl "iPhone 17 Pro" "http://localhost:3000/sign-up"

# Capture
xcrun simctl io "iPhone 17 Pro" screenshot ./shot.png
xcrun simctl io "iPhone 17 Pro" recordVideo --type=mp4 ./flow.mp4   # Ctrl-C to stop

# Reset state (for fresh-user scenarios)
xcrun simctl erase "iPhone 17 Pro"

# Shutdown when done
xcrun simctl shutdown "iPhone 17 Pro"
```

**Limitations.** `simctl` doesn't expose reliable input (taps, swipes).
For any interactive scenario, hand off to a sibling skill from the
ladder below.

## Companion-skill ladder

Curated list of community work. Our skill orchestrates the QA pass; these
projects drive the simulator. Each row links to the upstream README.

> **Verified at v0.2 release date.** Check each project's README and
> CHANGELOG for current flags before relying on a specific command.

| Project | Maintainer | Adds beyond simctl | Use when |
|---------|-----------|--------------------|----------|
| **[AXe](https://github.com/cameroncooke/AXe)** | [Cameron Cooke](https://github.com/cameroncooke) | Tap by `--id` / `--label` via Apple Accessibility APIs, `describe-ui` accessibility-tree dump (cheap text observation), `batch` for multi-step flows. `brew install cameroncooke/axe/axe`. `axe init` installs AXe's own agent skill. | You need real input on Mobile Safari or a native shell. **Default recommendation for input.** |
| **[XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP)** | Cameron Cooke / Sentry | 82-tool MCP server + CLI: `boot_sim`, `screenshot`, `record_sim_video`, `snapshot_ui` (view hierarchy), `build_run_sim`, `start_sim_log_cap`. Ships its own MCP and CLI skill primers. | Your agent runs in an MCP-capable client and you want one server covering build + simulator + log workflows |
| **[baguette](https://github.com/tddworks/baguette)** | [tddworks](https://github.com/tddworks) | Headless boot, 60fps streaming (MJPEG / H.264), taps via SimulatorKit private HID (the path iOS 26 still honors). `baguette serve` exposes a multi-device farm dashboard. `brew install tddworks/tap/baguette`. | iOS 26 simulators on Apple Silicon, high-fidelity gestures, or a browser-based device farm for human inspection |
| **[ios-simulator-skill](https://github.com/conorluddy/ios-simulator-skill)** | [Conor Luddy](https://github.com/conorluddy) | 27 Python scripts wrapping simctl + idb. Semantic navigation, WCAG audit, visual diff, test recorder. 96% token reduction vs raw tools. | You want a batteries-included Python toolkit alongside the BRB skill |
| **[ios-build-verify](https://github.com/vermont42/ios-build-verify)** | [Josh Adams](https://github.com/vermont42) | Named-intent scripts wrapping xcodebuild + AXe ("verify screen loaded", "audit accessibility for screen", "screenshot a named view"). Text-before-pixels via `axe describe-ui`. | You want the verification half of the agentic loop and prefer named intents over raw CLIs |
| **[ios-idb-skill](https://github.com/haowu77/ios-idb-skill)** | [Hao Wu](https://github.com/haowu77) | E2E via Meta's [idb](https://github.com/facebook/idb) (`device_init`, `device_tap`, `device_step`, log markers). Cross-agent (agentskills.io standard). | AXe isn't available, or you're already running `idb` for native E2E |
| **[serve-sim-skill](https://github.com/malopezr7/serve-sim-skill)** | [malopezr7](https://github.com/malopezr7) | Wraps Evan Bacon's [serve-sim](https://github.com/EvanBacon/serve-sim) for taps, gestures, rotation, **camera injection** (real / synthetic feeds). | React Native / Expo flows, or you need camera input |
| **[swiftui-autotest-skill](https://github.com/yusufkaran/swiftui-autotest-skill)** | [Yusuf Karan](https://github.com/yusufkaran) | `/ios-test` and `/add-accessibility` via Claude Code computer-use. Crawls every screen, crash logs, perf analysis. | Pure-SwiftUI native apps where the visual crawl IS the QA |
| **[xcode-build-skill](https://github.com/pzep1/xcode-build-skill)** | [pzep1](https://github.com/pzep1) | Teaches the agent xcodebuild + simctl directly, no MCP needed | Minimal-dependency setups, agents that prefer raw CLIs |
| **[App-Store-Connect-CLI](https://github.com/rudrankriyam/App-Store-Connect-CLI)** + **[app-store-connect-cli-skills](https://github.com/rudrankriyam/app-store-connect-cli-skills)** | [Rudrank Riyam](https://github.com/rudrankriyam) | Go CLI + agent skills for TestFlight, builds, submissions, signing, screenshot capture (via AXe). JSON-first, no prompts. | Post-QA: pushing a build to TestFlight or refreshing App Store screenshots after BRB sign-off |

If you build something that should live on this list, follow the
extension pattern in [extending-the-skill.md](extending-the-skill.md) and
open a PR.

## Recommended stacks per surface

Skip the table reading; here's the quick pick.

| Surface | Recommended stack |
|---------|-------------------|
| Web app on iPhone Mobile Safari, screenshot only | `xcrun simctl` (boot + openurl + screenshot) |
| Web app on iPhone Mobile Safari, interactive | `xcrun simctl` + **AXe** for `axe tap --label "Sign in"` |
| Web app on iPad Mobile Safari | Same, with iPad device name |
| iOS 26 + Apple Silicon, high-fidelity gestures | **baguette** |
| Native SwiftUI app, full visual crawl | **swiftui-autotest-skill** or **ios-simulator-skill** |
| Cross-agent client, one MCP for everything | **XcodeBuildMCP** |
| Verification primitives (named intents) | **ios-build-verify** |
| React Native / Expo | **serve-sim-skill** |
| TestFlight upload after BRB sign-off | **App-Store-Connect-CLI** + Rudrank's asc skills |

## Workflow patterns

### Web app QA on iPhone Mobile Safari

1. Boot the device:
   ```bash
   xcrun simctl boot "iPhone 17 Pro"
   ```
2. Open the dev URL:
   ```bash
   xcrun simctl openurl "iPhone 17 Pro" "http://localhost:3000/sign-up?invite=ABC"
   ```
3. Screenshot the resulting page:
   ```bash
   xcrun simctl io "iPhone 17 Pro" screenshot \
     docs/qa/bug-reports/assets/BUG-NNN/ios/01-signup.png
   ```
4. For interactions, escalate to AXe (or whichever companion is
   installed):
   ```bash
   axe tap --label "Continue" --udid $(xcrun simctl list devices "iPhone 17 Pro" -j \
     | jq -r '.devices | .[] | .[] | select(.name == "iPhone 17 Pro" and .state == "Booted") | .udid')
   ```
5. Screenshot the next state; repeat. File evidence under
   `docs/qa/bug-reports/assets/BUG-NNN/ios/` so the HTML report's
   "iOS Simulator screenshots" gallery picks it up.

### Web app QA on iPad

Same flow, different device name. `iPad Pro 13-inch`, `iPad Air 13-inch`,
`iPad mini`. Layout-heavy apps often have iPad-specific scenarios
(multi-column, sidebar, keyboard shortcuts) — add a P-block for them in
the manual test plan.

### Multi-device pass

```bash
for device in "iPhone 17 Pro" "iPad Pro 13-inch"; do
  xcrun simctl boot "$device" || true
  xcrun simctl openurl "$device" "http://localhost:3000/dashboard"
  sleep 1
  xcrun simctl io "$device" screenshot \
    "docs/qa/bug-reports/assets/BUG-NNN/ios/$(echo $device | tr ' ' '_').png"
done
```

Useful for "looks broken on iPad only" reports.

### Evidence capture for bugs

Always under `docs/qa/bug-reports/assets/BUG-NNN/ios/`. The HTML report's
bug detail page has a dedicated **iOS Simulator screenshots** gallery
section that lazy-loads tiles from that path. See
[html-report-style-guide.md](html-report-style-guide.md).

## Configuration

In `docs/qa/qa-config.json`:

```jsonc
"platforms": {
  "ios": {
    "enabled": true,
    "devices": ["iPhone 17 Pro", "iPad Pro 13-inch"],
    "url": "http://localhost:3000",
    "preferred": "AXe"     // or "baguette" / "XcodeBuildMCP" / "ios-build-verify"
  }
}
```

The `preferred` value tells the agent which companion to reach for first
when a scenario requires input.

## Graceful degradation

| Situation | Behavior |
|-----------|----------|
| On Linux / non-Mac cloud agent | iOS path unavailable. The agent says so and falls back to [browser-playbook.md](browser-playbook.md). |
| macOS but no Xcode | Detection fails. Agent tells the user to install Xcode CLI tools (`xcode-select --install`). |
| simctl works, no AXe/baguette/idb | Screenshot-only mode. Document the scenario's interactive steps in plain prose; the user / a separate session drives the simulator manually. |
| iOS 26 simulator, baguette not installed | Suggest `brew install tddworks/tap/baguette`; in the meantime, fall back to AXe (works through iOS 26 but with the standard caveat that Apple may evolve the path) or simctl-only. |

## Mobile Safari quirks to watch for

Add scenarios to your manual test plan that target each of these — they
hit users in production and Chrome viewport emulation never catches them:

- `-webkit-fill-available` height vs `100vh` (iOS Safari toolbar
  collapses)
- Soft-keyboard rect vs visual viewport — does the form's submit button
  hide behind the keyboard?
- `viewport-fit=cover` and notch / Dynamic Island safe areas
- IndexedDB quotas — Safari is stingier than Chrome
- PWA add-to-home heuristics — Safari shows the "Add to Home Screen"
  prompt only after specific user gestures
- Tap-target sizing (44 × 44 px minimum per Apple HIG)
- Form auto-fill behavior (1Password, iCloud Keychain)
- Smart App Banner conflicts with custom banners
- Service worker scope (Safari is stricter than Chrome)
- 100vh vs the safe-area-inset bottom

## Credit

This playbook stands on the shoulders of the iOS community. Major thanks
to:

- **[Cameron Cooke](https://github.com/cameroncooke)** for
  [AXe](https://github.com/cameroncooke/AXe) and
  [XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP), and for
  popularizing the accessibility-tree-as-observation-primitive pattern
  that this skill's evidence taxonomy leans on.
- **[Rudrank Riyam](https://github.com/rudrankriyam)** for the
  [App Store Connect CLI](https://github.com/rudrankriyam/App-Store-Connect-CLI)
  and [companion skills](https://github.com/rudrankriyam/app-store-connect-cli-skills),
  which thread QA evidence through to TestFlight / App Store review.
- **[Conor Luddy](https://github.com/conorluddy)** for
  [ios-simulator-skill](https://github.com/conorluddy/ios-simulator-skill),
  the inspiration for layered helper-script skills.
- **[Josh Adams](https://github.com/vermont42)** for
  [ios-build-verify](https://github.com/vermont42/ios-build-verify),
  for naming intents over raw CLIs.
- **[tddworks](https://github.com/tddworks)** for
  [baguette](https://github.com/tddworks/baguette), for the iOS 26
  input path.
- **[Hao Wu](https://github.com/haowu77)**,
  **[malopezr7](https://github.com/malopezr7)**,
  **[Yusuf Karan](https://github.com/yusufkaran)**,
  **[pzep1](https://github.com/pzep1)**, and many others building in
  the open under [agentskills.io](https://agentskills.io).
