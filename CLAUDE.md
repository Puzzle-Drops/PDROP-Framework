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
│   ├── index.html                   ← HTML shell + all CSS (framework + game styles)
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
| `public/index.html` | HTML shell, canvas, `#ui` div, all CSS, `<script>` tags | Rarely — only if game adds custom CSS |
| `public/framework.js` | Scaling, camera, chat UI, pings, scoreboard, tooltips, lobby screens | **Never** |
| `public/game.js` | Arrow-specific client code (GameDef, rendering, input) | **Delete and replace** |

**To start a new game:** Delete `game/arrow.js` and `public/game.js`. Create new versions that implement the same interfaces. Change the require path in `server.js`. Done. Framework is untouched.

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

## Interface Between Framework and Game

### Server Side

`server/framework.js` exports a function that sets up the lobby/networking infrastructure and calls into the game module. The game module (`game/arrow.js`) exports an object with these hooks:

```js
module.exports = {
    id: 'pdrop-arrow',
    name: 'PDROP Arrow',
    maxPlayers: 12,
    slotsPerTeam: 3,
    supportedModes: ['teams', 'ffa'],
    defaultMode: 'ffa',
    defaultTeamCount: 2,

    // Called when host starts the game
    init(players, settings, mode, teamCount) { },

    // Called every server tick (50ms)
    tick(dt) { },

    // Called when a client sends a game-specific message
    onInput(playerId, message) { },

    // Called to get current state for broadcast
    getState() { },

    // Called to check if round is over
    checkRoundOver() { },
};
```

### Client Side

`public/framework.js` sets up all UI systems and calls into the GameDef object. `public/game.js` defines `window.GameDef` with these hooks:

```js
window.GameDef = {
    id: 'pdrop-arrow',
    name: 'PDROP Arrow',
    worldWidth: 3200,
    worldHeight: 2400,

    getCameraLockTarget(localPlayer) { },
    getCameraSnapTarget(localPlayer) { },
    getScoreboardColumns() { },
    getPlayerStats(playerId) { },
    getEntityTooltip(entity) { },

    // Called once when game state starts arriving
    onGameStart(initialState) { },

    // Called every frame
    render(ctx, camera, gameState) { },

    // Called on mouse/keyboard input (framework filters out chat/ping/scoreboard inputs first)
    onInput(inputType, data) { },

    // Called to build the post-game results screen
    getResults(finalState) { },
};
```

## Build Order

Follow this sequence. Each step should be working and testable before moving to the next.

1. **Server entry + framework skeleton** — `server.js` wires Express + WS. `server/framework.js` accepts connections, manages lobbies. `game/arrow.js` is a stub.
2. **Client shell + scaling** — `index.html` with 1920×1080 container, canvas, `#ui` overlay, CSS zoom. `framework.js` handles resize. `game.js` is a stub.
3. **Main menu + lobby** — Create/join server, slot system, team panels, ready toggle. All in `framework.js` (both sides).
4. **Chat** — Enter for team, Shift+Enter for all. Framework handles routing. Working in lobby.
5. **Game start + countdown** — Host starts, 3-2-1 countdown, spawn players. Framework manages state transition, game provides spawn logic.
6. **Camera** — Y-lock to character, Space-jump, edge scrolling. All in client `framework.js`, game provides lock/snap targets.
7. **Movement** — Right-click to move. `game.js` sends input, `arrow.js` processes it, state broadcast renders it.
8. **Arrows** — Q to shoot toward cursor. Server collision detection, elimination broadcast.
9. **Win condition + post-game** — Round over detection in `arrow.js`, results screen in `framework.js` using `getResults()`.
10. **Pings** — Alt+Click wheel. Framework handles everything, game doesn't need to do anything.
11. **Scoreboard + mute** — Escape overlay with backdrop, right-click menu. Framework handles it, game provides `getScoreboardColumns()` and `getPlayerStats()`.
12. **Tooltips + HUD** — Framework tooltip system, game provides `getEntityTooltip()`. Kill feed and alive count in `game.js`.

## Testing Locally

```bash
npm install
npm start
# Open http://localhost:3000 in multiple browser tabs to test multiplayer
```

## Deploying to Railway

Push to the connected GitHub branch. Railway auto-detects `package.json`, runs `npm install`, and starts with `npm start`. HTTPS and WSS are handled automatically.
