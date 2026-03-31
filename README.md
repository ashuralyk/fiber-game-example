# Phaser's Revenge: A Layer 2 Gaming Demo

Phaser's Revenge is a Space Invaders-style browser game that demonstrates real-time micro-payments over [Fiber Network](http://fiber.world/) — CKB's Layer 2 payment channel protocol, analogous to Bitcoin's Lightning Network. Every in-game event (hitting the boss, taking damage) instantly settles CKB tokens between two Fiber nodes with no on-chain transaction and no perceptible latency.

The project is intentionally kept simple so it can serve as a reference for how game developers can wire blockchain payments into an existing Phaser.js game with minimal changes to the game code itself.

## How It Works

```
Browser (Phaser game)
    │  hit / damage event
    ▼
src/fiber/index.ts          ← payment orchestration
    │  createInvoice / sendPayment
    ├──► Boss node  (node1)  ──┐
    │    localhost:8227         │  Fiber off-chain channel
    └──► Player node (node2) ◄─┘
         localhost:8237
```

- **Boss node** acts as the game host. It owns the "boss" ship.
- **Player node** acts as the connected player.
- A pre-funded payment channel must already exist between the two nodes before the game starts.
- Each point scored equals **1 CKB** (10⁸ Shannon). The rate is defined by `amountPerPoint` in `src/fiber/index.ts`.

### Payment Flow

| Event | Who pays whom | Function called |
|---|---|---|
| Player hits the boss | Boss → Player | `payPlayerPoints()` |
| Boss bullet hits player | Player → Boss | `payBossPoints()` |

Both functions create a Fiber invoice on the recipient's node, then instruct the sender's node to pay it — all in one async call.

## Project Structure

```
src/
├── fiber/
│   ├── index.ts       # Node config, prepareNodes(), payment helpers
│   └── node.ts        # FiberNode wrapper around @ckb-ccc/fiber SDK
├── gameobjects/
│   ├── Player.ts      # Player spaceship (movement, shooting)
│   ├── BlueEnemy.ts   # Boss ship (tween movement, difficulty scaling)
│   └── Bullet.ts      # Bullet physics and particle effects
├── scenes/
│   ├── Preloader.ts   # Asset preloader
│   ├── SplashScene.ts # 2-second logo intro
│   ├── MenuScene.ts   # "Click to start" title screen
│   ├── MainScene.ts   # Core gameplay loop, collision handling, payments
│   ├── HudScene.ts    # Overlay: live points + countdown timer
│   └── GameOverScene.ts # Final scores: points, CKB gained, CKB lost
└── main.ts            # Phaser game config and scene registration
vite.config.ts         # Dev-server proxy for Fiber node APIs
```

## Prerequisites

### 1. Running Fiber Nodes

You need two Fiber nodes reachable from your machine. The quickest way is to run them locally. Refer to the [Fiber Network documentation](http://fiber.world/docs) for full node setup instructions.

By default the game expects:

| Role | RPC port | P2P port |
|---|---|---|
| Boss (node 1) | 8227 | 8228 |
| Player (node 2) | 8237 | 8238 |

### 2. Open a Payment Channel

Before starting the game, open a channel between the two nodes and fund each side with enough CKB to cover the rounds you intend to play. A comfortable starting balance is **500 CKB per side**.

The game itself does not open or close channels — it only sends payments through an already-open `CHANNEL_READY` channel.

## Configuration

All node configuration lives in two places.

### `src/fiber/index.ts` — Node identities

```ts
const node1 = {
    peerId: "QmUTsrLgHnoFenNbSFQyYAmAFxi3ZPkxAVnqJ8e9hhzeSm",   // boss node peer ID
    address: "/ip4/127.0.0.1/tcp/8228/p2p/QmUTsrLgHnoFenNbSFQyYAmAFxi3ZPkxAVnqJ8e9hhzeSm",
    url: "/node1-api",   // matches the Vite proxy path below
};

const node2 = {
    peerId: "QmTUyCohdeaWyPVSeQg5uWiTG8ztZ2rwBXP7PUZPk5Qc2w",   // player node peer ID
    address: "/ip4/127.0.0.1/tcp/8238/p2p/QmTUyCohdeaWyPVSeQg5uWiTG8ztZ2rwBXP7PUZPk5Qc2w",
    url: "/node2-api",
};
```

Replace `peerId` and `address` with the actual values from your nodes. You can retrieve a node's peer ID from its RPC:

```bash
curl -s http://localhost:8227 \
  -H 'Content-Type: application/json' \
  -d '{"id": 1, "jsonrpc": "2.0", "method": "node_info", "params": []}' \
  | jq '.result.node_id'
```

The `url` field is the Vite proxy path — it must match the key you set in `vite.config.ts` (see below). Do not change it unless you also update the proxy config.

### `vite.config.ts` — Proxy targets

```ts
server: {
    proxy: {
        "/node1-api": {
            target: "http://localhost:8227",   // boss node RPC
            changeOrigin: true,
        },
        "/node2-api": {
            target: "http://localhost:8237",   // player node RPC
            changeOrigin: true,
        },
    },
},
```

If your nodes run on different hosts or ports, update the `target` URLs here. The proxy exists so the browser can call Fiber node RPC endpoints without running into CORS restrictions during development.

### Non-local nodes

If you are connecting to remote nodes (e.g. a testnet node or a cloud VM), change the `target` values to the remote RPC URLs and update the `address` field in `src/fiber/index.ts` to use the correct IP/hostname and P2P port:

```ts
// example: remote nodes
const node1 = {
    peerId: "<your-boss-peer-id>",
    address: "/ip4/<boss-host>/tcp/<boss-p2p-port>/p2p/<your-boss-peer-id>",
    url: "/node1-api",
};
```

```ts
// vite.config.ts
"/node1-api": {
    target: "http://<boss-host>:<boss-rpc-port>",
    changeOrigin: true,
},
```

## Getting Started

```bash
# 1. Install dependencies
pnpm install

# 2. Edit node config (see Configuration section above)
#    src/fiber/index.ts   — peer IDs and addresses
#    vite.config.ts       — proxy targets

# 3. Start the dev server
pnpm dev
```

Open the URL printed by Vite (default: http://localhost:5173) and click to start. Make sure both Fiber nodes are running and the payment channel is open before the game initializes, otherwise the node connection step will throw an error in the browser console (the game will still load, but payments will be skipped).

```bash
# Type-check only (no emit)
pnpm typecheck

# Production build
pnpm build

# Preview the production build locally
pnpm preview
```

## Technical Details

| Layer | Technology |
|---|---|
| Game engine | Phaser.js 3 |
| Layer 2 | Fiber Network |
| Base chain | Nervos CKB |
| Token | Native CKB (Shannon unit) |
| SDK | `@ckb-ccc/fiber` |
| Build | Vite + TypeScript |

## Limitations and Production Considerations

This demo intentionally simplifies several concerns that a real game would need to address:

- **Channel lifecycle** — channels are opened and closed manually outside the game. A production system would handle channel negotiation, opening, and cooperative closing as part of the player session.
- **Balance checks** — if a channel runs out of funds mid-game, payments fail silently. Production code should monitor channel capacity and pause the game accordingly.
- **Single-player assumption** — node1 is always the boss and node2 is always the player. A real matchmaking system would assign nodes dynamically per session.
- **On-chain settlement** — final scores are not settled on-chain when the game ends. A production version should close the channel and record the final state.

## Disclaimer

This is a demonstration project to showcase the capabilities of Fiber Network for gaming applications. The implementation is simplified for educational purposes. Production applications should implement proper security measures, error handling, and user management systems.
