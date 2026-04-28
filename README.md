[![MseeP.ai Security Assessment Badge](https://mseep.net/pr/forrestkim42-llm-app-exploration-badge.png)](https://mseep.ai/app/forrestkim42-llm-app-exploration)

# App Exploration

A pattern for systematically exploring any app — mobile or desktop — using an LLM agent with device access. Produces a complete map of every screen, every interaction, and every user flow. Then uses that map to execute tasks efficiently.

This is an idea file. Share it with your LLM agent and explore together. The specifics will depend on your app, your platform, and your goals.

## The core idea

Most app research is manual. A person taps through an app, takes screenshots when something looks interesting, and writes notes afterward. This produces partial, biased coverage — you see what catches your eye, miss what doesn't, and have no way to know what you missed.

The idea here is different. An LLM agent connected to a device can read the screen (accessibility tree), understand what's on it (structured elements), and interact with it (tap, type, scroll, navigate). This means the agent can treat the app as a **graph** — each screen is a node, each interactive element is an edge — and perform a depth-first search. Systematically. Every screen, every dropdown, every toggle, every scroll position. Nothing skipped, nothing assumed.

The output is not a set of scattered screenshots. It's a **route map** — a structured, complete catalog of every screen in the app, with evidence and context. Like a wiki for the app's UI. Once you have this, user flows, competitive analysis, gap analysis, and UX audits become trivial — you're working from complete data instead of memory and impressions.

This works for any app on any platform. Phones, tablets, desktops. Native apps, Electron apps, anything with an accessibility tree. The pattern is the same.

## Philosophy

Three phases. Strictly separated.

**Explore** without organizing. **Organize** without analyzing. **Analyze** only on complete data.

Exploring while analyzing means you see what you want to see. Organizing while analyzing means your structure bends toward your conclusion. Analysis on incomplete data means confident but wrong. Do them in order. Each phase has its own output.

A fourth phase emerges naturally: **Act.** Once you've explored an app, you know its graph. Acting on a known graph is deterministic — no guessing, no intermediate screen reads, just a sequence of actions. Exploration produces knowledge; knowledge enables efficient action.

## Architecture

Four layers:

**Screenshots** — the raw evidence. Numbered PNGs captured during exploration. Immutable. Ground truth.

**ROUTES.md** — the route map. A flat table mapping every screenshot to its screen name, key elements, and one-line description. Plus a checklist of explored vs. unexplored areas, and a notes section for context. The index, the log, and the status tracker in one file.

**TRANSITIONS.md** — the transition table. Maps `screen + action → next screen`. Built naturally during exploration — every action's response shows the new screen. Once complete, entire multi-screen flows become fully deterministic and batchable without intermediate screen reads.

**User flows** — organized paths through the route map. "A user wants to send a message" becomes a sequence of screens with actions. Created after exploration is complete, by reading ROUTES.md and TRANSITIONS.md.

All four are built by the agent. You direct; the agent does the work.

## How the agent sees

The agent doesn't look at pixels. It reads the **accessibility tree** — the structured data that every app exposes for screen readers. Every button, text field, label, and toggle has a name, a role, and a position in the tree. This is the same data that VoiceOver and TalkBack use.

Reading the accessibility tree instead of screenshots means:

- **Structured data, not pixel interpretation.** `BUTTON:Save` is unambiguous. A screenshot of a save icon requires vision inference.
- **Fast and cheap.** A tree is a few KB of text. A screenshot is tens of thousands of vision tokens.
- **Off-screen elements included.** The tree contains elements that haven't scrolled into view yet. Screenshots only show the viewport.
- **Deterministic targeting.** Tap `BUTTON:Save` and it hits the button regardless of screen resolution, DPI, or dynamic layout shifts.

Different platforms expose accessibility differently:

- **Mobile (Android):** UI dump via system tools. Some frameworks (React Native, Flutter) need fallbacks for their animation-based rendering.
- **Desktop native (macOS):** Accessibility API. Fast (~20ms), reads full UI tree of any app.
- **Desktop Electron:** The app's native accessibility exposure is incomplete. But every Electron app is Chromium under the hood — connect via Chrome DevTools Protocol and read the DOM directly. `data-qa`, `aria-label`, `textContent` — the DOM has everything the accessibility tree omits.

The platform dictates the mechanism, but the output is the same: a list of elements with types and labels that the agent can interact with.

## Element identity

Every element gets a stable ID in the format `TYPE:Label`.

```
BUTTON:Save        INPUT:Email        TEXT:Balance: $5.00
TAB:Home           LINK:Settings      CHECKBOX:Remember me
```

If duplicates exist, they get suffixed: `BUTTON:OK@1`, `BUTTON:OK@2`.

This format is intentional:
- **TYPE tells you what it does.** BUTTON is tappable. INPUT accepts text. TEXT is read-only.
- **Label tells you what it is.** Human-readable, matches what's on screen.
- **Stable across sessions.** Same app state → same IDs. No coordinates, no fragile indices.

For multi-app or desktop scenarios, an app prefix disambiguates: `Slack/BUTTON:Send`, `Arc/INPUT:URL`. This makes cross-app workflows expressible as a flat sequence of actions.

## Operations

**Capture.** At each screen: read the elements, take a screenshot, record what you see. One row in ROUTES.md per screenshot.

**Navigate.** Treat the app as a graph. DFS: for each interactive element on the current screen, tap it. If it's a new screen, record the transition and recurse. If it's a modal, capture and dismiss. Go back. Tap the next element. Batch multiple screens in one call — tap, capture, back, tap, capture, back. One call covers an entire level of the tree.

**Record transitions.** Every action that changes the screen is a transition. The response always shows the new screen — log it as `screen + action → next screen` in TRANSITIONS.md.

**Document.** After exploration, read ROUTES.md and TRANSITIONS.md end to end. Group screenshots into user flows by task. One flow = one user goal.

### Depth

Not every exploration needs the same depth. Decide upfront.

**L1 — Screens.** Visit every screen, capture it. You get a complete screen map.

**L2 — Interactions.** Open every dropdown, toggle every switch, scroll to the bottom of every list. You get every option, every state.

**L3 — Real actions.** Execute actual tasks — send a message, complete a payment, submit a form. You get hidden screens (confirmation, success, error, edge cases), actual behavior under real conditions, and the gap between what the UI promises and what happens. This may cost real money or create real data.

## The route map

ROUTES.md is the single most important artifact. Everything else derives from it.

```markdown
# [App Name] Route Map

> Version: [x.x.x] | Explored: [date]
> Platform: [device/OS] | Method: [MCP tool]
> Status: **Complete** (102 screenshots)

## Notes
- Biometric prompt blocks screenshot — documented via accessibility dump
- Test account: user@example.com

## Routes

| # | File | Screen | Key Elements |
|---|------|--------|-------------|
| 1 | 001_lock_screen.png | Lock screen | Logo, fingerprint icon, "Unlock with Password" |
| 2 | 002_home.png | Home | Balance $241, 11-tile grid, 2 dApp cards |
| 3 | 003_search.png | Search | Search bar, filters, recent history |

## Checklist
- [x] Lock & auth
- [x] Home
- [x] Swap + chain select + token select
- [ ] Settings → Custom Network
```

Every screenshot gets a row. Key Elements is factual — what you see, not what you think. The checklist is honest — unchecked items are known unknowns. This tells the next person (or the next session) exactly where to continue.

## Transition table

ROUTES.md tells you what's on each screen. A **transition table** tells you what happens when you act — `screen + action → next screen`. This is what makes fully batched, zero-intermediate-read flows possible.

```markdown
# TRANSITIONS.md

## Home
- tap BUTTON:Bridge → Bridge
- tap BUTTON:Swap → Swap
- tap BUTTON:Settings → Settings

## Bridge
- tap INPUT:Amount + type {amount} → Bridge (with quote)
- tap BUTTON:Confirm → AuthPrompt

## AuthPrompt
- tap BUTTON:Cancel → PasswordEntry
- [biometric success] → Processing
```

With this, an entire multi-screen flow can be pre-planned as a single batch — no intermediate reads needed. The stable element IDs make this work: `BUTTON:Confirm` is always `BUTTON:Confirm` regardless of session, resolution, or platform.

Build it naturally during exploration. Every action's response shows the new screen — that's a transition. Record it. After exploration, the transition table is complete and flows become deterministic.

### Parameterized flows

Transitions that differ only in data can be parameterized:

```markdown
## Batch flows

### send_message
params: [recipient, message]
1. tap BUTTON:sidebar_{recipient}
2. wait 1500
3. tap BUTTON:message_input
4. type {message}
5. tap BUTTON:send
```

One recording, infinite reuse. Change the parameters, execute the same sequence.

### Learning by doing

Transition tables don't have to be built upfront. Every time the agent performs a task — even a one-off request — the before state, actions, and after state form a transition. Record it. Over time, the table fills itself. The agent learns the app by using it.

This works well as a background process. While the agent responds to the user, a parallel process can analyze what just happened and update the transition table. The user's next request benefits from the previous one's knowledge.

## Tools

Two MCP servers implement this pattern. Both expose a single tool that reads the screen and executes actions in one call, using `TYPE:Label` element IDs.

**[mobile-mcp](https://github.com/ForrestKim42/mobile-mcp/tree/feat/llm-friendly-ui-analysis)** — Android and iOS. Real devices and simulators. Handles React Native/Flutter via automatic fallback. Single `mobile_do` tool.

**[desktop-mcp](https://github.com/ForrestKim42/desktop-mcp)** — macOS. Native apps via Accessibility API, Electron apps via Chrome DevTools Protocol (DOM-level access). Single `desktop_do` tool. Cross-app batching with `App/TYPE:Label` prefix.

Add either to your agent's MCP config and start exploring. The agent will figure out the rest.

## Platform notes

**Mobile** — phone with developer access, connected via USB. The MCP server handles accessibility tree reading and input injection.

**Desktop native** — accessibility API is always available (one-time permission grant in System Settings).

**Desktop Electron** — launch with `--remote-debugging-port=PORT` for CDP access. This gives the agent direct DOM access — `data-qa`, `aria-label`, `textContent` — far richer than the OS accessibility layer alone.

**Cross-platform** — the element ID format and action format are platform-independent. A transition table built on one platform can inform actions on another if the app's UI is similar. The pattern is the same; only the transport differs.

## Tips and tricks

**Biometric / secure screens** capture as black on mobile. The accessibility tree still works — it reads element data regardless. Cancel the biometric prompt to reach the password fallback.

**Electron apps with weak accessibility** — many Electron apps expose minimal data through the OS accessibility layer. Don't stop there. Connect via CDP and query the DOM directly. `data-qa` attributes (used by the app's own test team) are the most stable selectors. `aria-label` and `textContent` fill the rest.

**Virtual/lazy lists** only render visible items in the DOM. The accessibility tree often sees more. If an element exists in the accessibility tree but not in the DOM (or vice versa), use whichever source can see it.

**Session breaks** are inevitable — context limits, crashes, timeouts. ROUTES.md's checklist is your resume point. Read it, find the unchecked items, continue from the last screenshot number.

**Don't assume absence.** If you didn't tap it, you don't know if it exists. "Not found" and "doesn't exist" are different claims. The checklist enforces this — an unchecked item is an honest gap, not a negative finding.

**Real actions reveal hidden UX.** Confirmation flows, error states, pricing details, rate limiting, post-action screens — none of these appear until you actually execute. A single real action surfaces screens that UI exploration alone will never find.

**Cross-app workflows** are first-class. Copy from one app, paste into another. Read a notification, tap through to the source app. Each step is an action with an app prefix. The transition table handles app switches the same way it handles screen switches.

## Why this works

The bottleneck in app research was never the analysis — it was the data collection. Tapping through an app is tedious, inconsistent, and incomplete. You get tired, you skip things, you forget what you already checked. An LLM agent doesn't get tired, maintains a checklist, and can systematically cover hundreds of screens without losing track.

The bottleneck in app automation was never the actions — it was the understanding. Most automation is brittle because it targets coordinates or hardcoded selectors that break when the UI changes. An agent that reads the accessibility tree targets elements by their semantic identity. `BUTTON:Save` works regardless of where the button is on screen, what color it is, or whether the layout shifted.

Exploration produces understanding. Understanding enables reliable action. The route map and transition table are the bridge between the two.

## Note

This document is intentionally abstract. It describes a pattern, not an implementation. The exact tools, file structure, naming conventions, and depth level depend on your app, your platform, and your agent. The pattern works on any app with a UI — phones, tablets, laptops. Share this with your LLM agent and build out the specifics together.

## File structure

For reference, here's what a completed exploration looks like:

```
[app-name]/
  screenshots/
    001_lock_screen.png
    002_home.png
    ...
  ROUTES.md
  TRANSITIONS.md
  user-flows/
    01-onboarding.md
    02-send.md
    ...
```

For multi-app comparison:

```
products/
  app-a/
    screenshots/
    ROUTES.md
    TRANSITIONS.md
  app-b/
    ...
analysis/
  comparison.md
```
