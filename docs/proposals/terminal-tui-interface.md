# Proposal: Terminal/TUI Interface for CoreScope

**Status:** Idea / Early Proposal
**Issue:** TBD

## Problem

CoreScope's web UI requires a browser. Operators managing remote mesh deployments often work over SSH — headless servers, Raspberry Pis, field laptops with spotty connectivity. They need to check mesh health, view packet flow, and diagnose issues without opening a browser.

## Vision

A terminal-based user interface (TUI) that connects to a CoreScope instance's API and renders key views directly in the terminal. Think `htop` for mesh networks.

## Core Views

### 1. Fleet Dashboard (default view)
```
┌─ CoreScope TUI ──────────────────────────────────────────┐
│ Nodes: 518 active | Observers: 35 | Packets/hr: 2,336   │
├──────────────────────────────────────────────────────────┤
│ Observer          │ NF     │ TX%  │ RX%  │ Errs │ Status │
│ GY889 Repeater    │ -112   │ 2.1  │ 8.3  │    0 │ ●      │
│ C0ffee SF         │ -108   │ 1.4  │ 12.1 │   42 │ ●      │
│ ELC-ONNIE-RPT-1   │ -95   │ 3.2  │ 6.7  │  180 │ ▲      │
│ Bar Repeater 🤷   │ -76   │ 0.1  │ 0.0  │   62 │ ▼      │
└──────────────────────────────────────────────────────────┘
```

### 2. Live Packet Feed
```
┌─ Live Feed ──────────────────────────────────────────────┐
│ 14:32:01 ADVERT   GY889 Repeater       → 3 hops  -112dB │
│ 14:32:02 GRP_TXT  #test "hello world"  → 5 hops  -98dB  │
│ 14:32:03 TXT_MSG  [encrypted]          → 2 hops  -105dB │
│ 14:32:04 CHAN     #sf "anyone on?"     → 8 hops  -91dB  │
└──────────────────────────────────────────────────────────┘
```

### 3. Node Detail
- Select a node → see its packets, paths, neighbors, health history
- Keyboard navigation (j/k up/down, Enter to select, Esc to back)

### 4. RF Health Sparklines
- ASCII sparklines for noise floor over time per observer
- `▁▂▃▅▇█` block characters for compact visualization

## Architecture

```
CoreScope Server (existing)
    ↑ REST API + WebSocket
    │
CoreScope TUI (new binary)
    - Go binary using bubbletea/lipgloss (or tview)
    - Connects to any CoreScope instance via --url flag
    - No database, no state — pure API consumer
    - Real-time updates via WebSocket
```

## Key Design Decisions

- **Separate binary** — `corescope-tui` alongside `corescope-server`. Not embedded in the server.
- **API consumer only** — uses existing REST + WebSocket APIs. No direct DB access. Works against any CoreScope instance (local or remote).
- **Keyboard-driven** — vim-like navigation (j/k/g/G), tab to switch views, / to filter, q to quit
- **No mouse required** — but mouse support for clicking rows is nice-to-have
- **Color terminal assumed** — 256-color minimum, true-color preferred. Graceful fallback to 16-color.
- **Cross-platform** — single Go binary, no dependencies. Works on Linux, macOS, Windows.

## Technology Options

| Library | Language | Pros | Cons |
|---|---|---|---|
| **bubbletea + lipgloss** | Go | Elm-architecture, composable, beautiful output, active community | Newer, less battle-tested |
| **tview** | Go | Mature, widget-rich, built-in tables/forms | Imperative style, harder to compose |
| **charm/wish** | Go | SSH server built-in — serve TUI over SSH without local install | Adds complexity |
| **textual** | Python | Rich widget library | Not Go, separate runtime |

**Recommendation:** bubbletea + lipgloss. Matches CoreScope's Go stack, Elm-architecture is clean, lipgloss styling is excellent.

## Stretch Features (not M1)
- **SSH server mode** — run `corescope-tui --serve-ssh :2222` and operators SSH into the TUI directly. No binary install needed on client side.
- **Alerting** — terminal bell / desktop notification on RF anomalies
- **Multi-instance** — connect to multiple CoreScope instances, tab between them
- **Export** — dump current view as CSV/JSON to stdout

## Open Questions

1. **Who is the target user?** Field operators? Sysadmins? Power users who prefer terminal? All of the above?
2. **Minimum API coverage for M1?** Fleet dashboard + live feed is probably enough.
3. **Should this live in the same repo or a separate one?** Same repo (`cmd/tui/`) keeps it co-located but adds build complexity.
4. **Branding?** `corescope-tui`? `cscope`? `meshmon`?
