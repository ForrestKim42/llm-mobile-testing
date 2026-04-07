# Mobile App Exploration

A pattern for systematically exploring any Android app using an LLM agent with physical device access — producing a complete map of every screen, every interaction, and every user flow.

This is an idea file. Share it with your LLM agent and explore together. The specifics will depend on your app, your device, and your goals.

## The core idea

Most app research is manual. A person taps through an app, takes screenshots when something looks interesting, and writes notes afterward. This produces partial, biased coverage — you see what catches your eye, miss what doesn't, and have no way to know what you missed.

The idea here is different. An LLM agent connected to a phone via ADB can see the screen (screenshot), understand what's on it (UI dump), and interact with it (tap, type, scroll, navigate). This means the agent can treat the app as a **graph** — each screen is a node, each tappable element is an edge — and perform a depth-first search. Systematically. Every screen, every dropdown, every toggle, every scroll position. Nothing skipped, nothing assumed.

The output is not a set of scattered screenshots. It's a **route map** — a structured, complete catalog of every screen in the app, with evidence (screenshots) and context (descriptions). Like a wiki for the app's UI. Once you have this, user flows, competitive analysis, gap analysis, and UX audits become trivial — you're working from complete data instead of memory and impressions.

This works for any app. We've used it on crypto wallets, but the pattern applies equally to banking apps, e-commerce, social media, productivity tools — anything with a UI.

## Philosophy

Three phases. Strictly separated.

**Explore** without organizing. **Organize** without analyzing. **Analyze** only on complete data.

Exploring while analyzing means you see what you want to see. Organizing while analyzing means your structure bends toward your conclusion. Analysis on incomplete data means confident but wrong. Do them in order. Each phase has its own output.

## Architecture

Three layers, same as any good knowledge system:

**Screenshots** — the raw evidence. Numbered PNGs captured during exploration. Immutable. The agent captures them; no one edits them. This is ground truth.

**ROUTES.md** — the route map. A flat table mapping every screenshot to its screen name, key elements, and one-line description. Plus a checklist of explored vs. unexplored areas, and a notes section for context (app version, device quirks, test account credentials, app state). The agent builds this during exploration and keeps it current. It's the index, the log, and the status tracker in one file.

**User flows** — organized paths through the route map. "A user wants to complete a purchase" becomes a sequence of screenshots with step numbers, actions, and tap counts. Created after exploration is complete, by reading ROUTES.md and connecting the dots. One file per flow.

The screenshots are captured by the agent. ROUTES.md is maintained by the agent. User flows are created by the agent. You direct; the agent does the work.

## Operations

**Capture.** At each screen: take a screenshot, read the UI dump (structured elements with coordinates), record what you see. One row in ROUTES.md per screenshot. Number sequentially. Name descriptively. Move on.

**Navigate.** Treat the app as a graph. DFS: for each tappable element on the current screen, tap it. If it's a new screen, recurse. If it's a modal or dropdown, capture and dismiss. Press BACK to return. When all elements are explored, the node is done. Maintain a checklist — mark what's explored, note what isn't.

**Document.** After exploration, read ROUTES.md end to end. Group screenshots into user flows by task. One flow = one user goal. Include step count and tap count — these are comparable across apps.

### Depth

Not every exploration needs the same depth. Decide upfront.

**L1 — Screens.** Visit every screen, capture it. You get a complete screen map.

**L2 — Interactions.** Open every dropdown, toggle every switch, scroll to the bottom of every list. You get every option, every state.

**L3 — Real actions.** Execute actual tasks — place an order, complete a payment, submit a form, go through a full onboarding. You get hidden screens (confirmation, success, error, edge cases), actual fee/pricing behavior, and the gap between what the UI promises and what actually happens. This may cost real money or create real data.

## The route map

ROUTES.md is the single most important artifact. Everything else derives from it.

```markdown
# [App Name] Route Map

> Version: [x.x.x] | Explored: [date]
> Device: [model] | Method: ADB MCP
> Status: **Complete** (102 screenshots)

## Notes
- Biometric prompt blocks screenshot (FLAG_SECURE) — documented via UI dump
- Test account: user@example.com
- Device-specific: [any quirks]

## Routes

| # | File | Screen | Key Elements |
|---|------|--------|-------------|
| 1 | 001_lock_screen.png | Lock screen | Logo, fingerprint icon, "Unlock with Password" |
| 2 | 002_home.png | Home | Balance $241, 11-tile grid, 2 dApp cards |
| 3 | 003_search.png | Search | Search bar, filters, recent history, 3 categories |
| ... | ... | ... | ... |

## Checklist
- [x] Lock & auth
- [x] Home
- [x] Swap + chain select + token select
- [ ] Settings → Custom Network
```

Every screenshot gets a row. Key Elements is factual — what you see, not what you think. The checklist is honest — unchecked items are known unknowns. This is valuable. It tells the next person (or the next session) exactly where to continue.

## Setup

You need ADB (Android Debug Bridge) and an MCP server that wraps it.

**On the phone:** Developer Mode on, USB Debugging on. Connect via USB.

**On the computer:**

```bash
# Verify connection
adb devices

# MCP server — add to your agent's MCP config
{
  "mcpServers": {
    "android-debug-bridge": {
      "command": "npx",
      "args": ["-y", "android-debug-bridge-mcp"]
    }
  }
}
```

Package: [android-debug-bridge-mcp](https://npmjs.com/package/android-debug-bridge-mcp) by TiagoDanin (MIT). Works with any MCP-compatible agent — Claude Code, Cursor, Windsurf, or anything that speaks MCP.

This gives the agent: `capture_screenshot`, `capture_ui_dump`, `input_tap`, `input_text`, `input_keyevent`, `input_scroll`, `open_app`. That's all you need.

## Tips and tricks

**Foldable devices** (Galaxy Z Fold, etc.) have dual displays. `screencap` without a display ID outputs garbage. Fix: `adb shell dumpsys SurfaceFlinger --display-id` to find IDs, then patch the MCP handler to add `-d <display_id>`. Restart the MCP server after patching — Node loads code at startup.

**FLAG_SECURE screens** (biometric prompts, banking screens) capture as black. Use `capture_ui_dump` instead — it reads the accessibility tree regardless. Document what the dump reveals.

**Scroll reliability** varies by device. `input_scroll` may be too gentle. Fallback: `adb shell input swipe 540 2000 540 800 500`. Adjust coordinates for your screen resolution.

**Session breaks** are inevitable — context limits, app crashes, device timeouts. ROUTES.md's checklist is your resume point. Read it, find the unchecked items, continue from the last screenshot number.

**Don't assume absence.** If you didn't tap it, you don't know if it exists. "Not found" and "doesn't exist" are different claims. The checklist enforces this — an unchecked item is an honest gap, not a negative finding.

**Real actions reveal hidden UX.** Confirmation flows, error states, pricing details, rate limiting, post-action screens — none of these appear until you actually execute. A single real action surfaces screens that UI exploration alone will never find.

## Why this works

The bottleneck in app research was never the analysis — it was the data collection. Tapping through an app is tedious, inconsistent, and incomplete. You get tired, you skip things, you forget what you already checked. An LLM agent doesn't get tired, maintains a checklist, and can tap through 100+ screens in a session without losing track.

The human's job is to direct: which app, how deep, what to focus on. The agent's job is everything mechanical: tap, capture, record, navigate back, repeat. The result is research-grade coverage that would take a human team days, produced in hours, with every claim backed by a screenshot.

## Note

This document is intentionally abstract. It describes a pattern, not an implementation. The exact file structure, the ROUTES.md format, the naming conventions, the depth level — all of that depends on your app, your goals, and your agent. The examples above are from real explorations (100+ screenshots each), but the pattern works on any app with a UI. Share this with your LLM agent and build out the specifics together. The document's only job is to communicate the idea. Your agent can figure out the rest.

## File structure

For reference, here's what a completed exploration looks like:

```
[app-name]/
  screenshots/
    001_lock_screen.png
    002_home.png
    ...
  ROUTES.md
  user-flows/
    01-onboarding.md
    02-send.md
    03-checkout.md
    ...
```

For multi-app comparison:

```
products/
  app-a/
    screenshots/
    ROUTES.md
    user-flows/
  app-b/
    ...
analysis/
  comparison.md
report/
  index.html
```
