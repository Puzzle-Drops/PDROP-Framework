# PDROP Arrow

A multiplayer top-down arena game built on the PDROP Framework. Right-click to move, Q to shoot arrows, one hit kills. Last player/team standing wins.

## Project Structure

```
pdrop-arrow/
├── CLAUDE.md                        ← You are here
├── package.json
├── server.js                        ← Entry point: wires framework server to game logic
│
├── server/
│   └── framework.js                 ← FRAMEWORK: lobby, slots, chat routing, ping routing, tick loop
│
├── game/
│   └── arrow.js                     ← GAME: server-side arrow logic (movement, arrows, collisions, win)
│
├── public/
│   ├── index.html                   ← SHARED: framework structure + game CSS/markup
│   ├── framework.js                 ← FRAMEWORK: scaling, camera, chat UI, pings, scoreboard, tooltips, lobby UI
│   └── game.js                      ← GAME: client-side arrow code (GameDef, rendering, input, HUD)
│
└── docs/
    ├── pdrop-framework.md           ← Framework spec
    └── pdrop-arrow-design.md        ← Game design doc
```

### What's Framework vs Game

| File | Role | Changes between games? |
|------|------|----------------------|
| `server.js` | Entry point — requires framework + game, starts Express + WS | One line: which game file to require |
| `server/framework.js` | Lobby, slots, chat, ping routing, tick loop shell | **Never** |
| `game/arrow.js` | Arrow-specific server logic | **Delete and replace** |
| `public/index.html` | Framework structure + game CSS/markup (separated by comments) | **Partially** — keep framework sections, replace game sections |
| `public/framework.js` | Scaling, camera, chat UI, pings, scoreboard, tooltips, lobby screens | **Never** |
| `public/game.js` | Arrow-specific client code (GameDef, rendering, input, HUD) | **Delete and replace** |

**To start a new game:**

1. Replace `game/arrow.js` with new server game logic
2. Replace `public/game.js` with new `window.GameDef` (only define hooks you need — all are optional)
3. In `index.html`, clear the game CSS/markup sections (marked by comments), add your own
4. Change the require path in `server.js`

Framework files never touched: `server/framework.js`, `public/framework.js`, `package.json`.

## Read These First

Before writing any code, read both design documents in `/docs/`:

1. **`docs/pdrop-framework.md`** — The framework spec. Defines the 1920×1080 scaling system, CSS zoom overlay, camera (Y-lock / Space-jump), chat (Enter / Shift+Enter), ping system (Alt+Click wheel), scoreboard & mute (Escape with full-screen backdrop that consumes clicks), tooltip system, player colors, lobby slot system, and the `GameDef` hook interface. All of this is standard infrastructure in the framework files.

2. **`docs/pdrop-arrow-design.md`** — The game-specific design doc. Defines the rules (right-click move, Q shoot, one-hit kill), player characters, arrow behavior, arena layout, networking protocol (20 tick/sec server-authoritative), lobby configuration, HUD elements, and visual style. All of this goes in the game files only.

## Tech Stack

- **Client:** `public/index.html` + `public/framework.js` + `public/game.js` — plain JS, HTML5 Canvas, no bundler
- **Server:** `server.js` + `server/framework.js` + `game/arrow.js` — Node.js + Express + `ws`
- **Deployment:** GitHub → Railway (auto-deploy on push)

## Key Conventions

- **No build tools.** No webpack, no vite, no typescript, no JSX. Plain `.html` and `.js` files. Client uses `<script>` tags. Server uses `require()`.
- **Server authoritative.** The server owns all game state. Clients send inputs, receive state, and render. Clients never compute game logic.
- **Fixed virtual resolution.** Everything is authored at 1920×1080. The scaling system (CSS media queries + zoom) handles fitting to any viewport. Never use `window.innerWidth` for layout — use the 1920×1080 coordinate space.
- **CSS `zoom` for DOM UI.** The `#ui` overlay uses `zoom` (not `transform: scale`) so text stays crisp. See §3 of the framework doc.
- **Single font.** Rajdhani from Google Fonts. 400/500/600/700 weights. Used in both DOM and canvas (`ctx.font`).
- **Player colors.** Red is slot 0 / Team 1. Blue is slot 1 / Team 2. Full palette in §10 of framework doc.
- **Port from environment.** Server must use `process.env.PORT || 3000`. Railway assigns the port.
- **WebSocket on same server.** WS upgrades happen on the Express HTTP server. Client connects to `location.host` — no hardcoded URLs.
- **Scoreboard consumes clicks.** When Escape scoreboard is open, a full-screen backdrop with `pointer-events: auto` blocks all canvas interaction. See §8 of framework doc.
- **Camera zoom.** Mouse wheel zooms 0.5×–2×. All screen-to-world conversions account for `camera.zoom`. Canvas uses `setTransform()` for zoom; HUD renders in screen space after transform reset.
- **Session persistence.** Player name saved in `localStorage`. Session (player ID + lobby ID) saved in `sessionStorage`. On reconnect, client sends `rejoin` to restore into the same slot/game.
- **No text selection.** `user-select: none` on all elements except input fields.

## Interface Between Framework and Game

### Server Side

`server/framework.js` exports a function that sets up the lobby/networking infrastructure and calls into the game module. The game module (`game/arrow.js`) exports an object. Only `id`, `name`, and `maxPlayers` are required. All hooks are optional — the framework checks before calling.

```js
module.exports = {
    // Required
    id: 'pdrop-arrow',
    name: 'PDROP Arrow',
    maxPlayers: 12,

    // Optional properties
    supportedModes: ['teams', 'ffa'],    // default: ['ffa']
    defaultMode: 'ffa',
    defaultTeamCount: 2,

    // Optional hooks (framework guards all of these)
    init(players, settings, mode, teamCount) { },
    tick(dt) { },
    onInput(playerId, message) { },
    getState() { },
    checkRoundOver() { },
};
```

### Client Side

`public/framework.js` sets up all UI systems and calls into the GameDef object. `public/game.js` defines `window.GameDef`. Only `id`, `name`, and `maxPlayers` are required. All hooks are optional — only implement what the game needs.

```js
window.GameDef = {
    // Required
    id: 'pdrop-arrow',
    name: 'PDROP Arrow',
    maxPlayers: 12,

    // Optional properties
    worldWidth: 3200,       // default: 1920 (no camera needed)
    worldHeight: 2400,      // default: 1080 (no camera needed)

    // Optional hooks (only implement what you need)
    getCameraLockTarget(localPlayer) { },
    getCameraSnapTarget(localPlayer) { },
    getScoreboardColumns() { },
    getPlayerStats(playerId) { },
    getEntityTooltip(entity) { },
    onGameStart(initialState) { },
    render(ctx, camera, gameState) { },
    onInput(inputType, data) { },
    getResults(finalState) { },
};
```

## Build Order (All Complete)

All 12 steps are implemented and working.

1. **Server entry + framework skeleton** — `server.js` wires Express + WS. `server/framework.js` manages lobbies, slots, chat, pings, tick loop, session rejoin.
2. **Client shell + scaling** — `index.html` with 1920×1080 scaling, CSS zoom overlay. `framework.js` handles resize.
3. **Main menu + lobby** — Create/join server, slot system, team panels, ready toggle, host transfer, mode switching.
4. **Chat** — Enter opens team chat, Shift+Enter opens all chat. Tab/Shift cycles channel. Framework routes team/all.
5. **Game start + countdown** — 3-2-1 countdown with arena visible. Players spawn at FFA ellipse or team corners.
6. **Camera** — Y-lock toggle, Space-snap (stays on release), edge scrolling, mouse wheel zoom (0.5×–2×).
7. **Movement** — Right-click to move. Server-authoritative with wall/obstacle collision.
8. **Arrows** — Q to shoot toward cursor. 30-tick cooldown. Server collision detection, elimination broadcast, kill feed, alive count HUD.
9. **Win condition + post-game** — Last player/team standing. Kill leaderboard on results screen. Rematch returns to lobby.
10. **Pings** — Alt+Click wheel with 5 types. Team-only visibility. Audio feedback. Mute filtering.
11. **Scoreboard + mute** — Escape overlay. Game-defined stat columns. Per-player right-click mute menu. Global mute toggles. Persisted in localStorage.
12. **Tooltips + HUD** — DOM tooltips via `data-tooltip`. Canvas entity tooltips on players (name + status) and obstacles. Kill feed + alive count.

## Testing Locally

```bash
npm install
npm start
# Open http://localhost:3000 in multiple browser tabs to test multiplayer
```

## Deploying to Railway

Push to the connected GitHub branch. Railway auto-detects `package.json`, runs `npm install`, and starts with `npm start`. HTTPS and WSS are handled automatically.
