# Crypto Watchlist

A real-time crypto watchlist modeled on professional trading terminals. It streams live 24h ticker data for every Binance Spot pair over WebSocket and renders it in a dense, virtualized table that stays smooth with 3,000+ symbols updating every second.

Built with **React 18, TypeScript, Tailwind CSS v4 and Vite**. There are no runtime dependencies beyond React.

## Features

- **Live market data** for all Binance Spot pairs: last price, 24h change, 24h %, high, low and volume.
- **Price flash.** Each price tick flashes green (`#0ECB81`) or red (`#F6465D`), holds for 400ms, then fades back to neutral.
- **Search** by symbol (`btc`, `btc/usdt`) or asset name (`solana`). Press `/` to focus the search box and `Esc` to clear it.
- **Quote tabs** for ALL, USDT and BTC, each showing a live pair count.
- **Sorting** by Pair, Last Price, 24h % and 24h Volume, in both directions. On the ALL tab, volume is normalized to USD so BTC-, EUR- and USDT-quoted pairs rank against each other correctly.
- **Connection handling:**
  - A top banner shows a retry countdown, the attempt number, the time of the last update and a Retry button.
  - Offline detection pauses retries until the network returns.
  - A watchdog reconnects if the stream goes quiet.
- **Loading and empty states:** skeleton rows during the initial connection, an empty state with "Clear filters", and a stale-data notice when the stream gives up.
- **Responsive layout.** Secondary columns collapse on smaller screens, and the table works at phone width.

## Getting started

Requirements: Node.js 18 or later.

```bash
npm install
npm run dev
```

Then open http://localhost:5173.

| Script              | What it does                         |
| ------------------- | ------------------------------------ |
| `npm run dev`       | Start the Vite dev server            |
| `npm run build`     | Type-check and build to `dist/`      |
| `npm run preview`   | Serve the production build locally   |
| `npm run typecheck` | Run the TypeScript compiler only     |

## Project structure

```
src/
├── types/binance.ts              Binance WS/REST payload types and the normalized Ticker model
├── hooks/useBinanceStream.ts     WebSocket lifecycle, reconnect/backoff, snapshot seeding, batching
├── components/
│   ├── WatchlistTable.tsx        Toolbar (tabs + search), sorting, virtualization, loading/empty states
│   ├── TickerRow.tsx             Memoized row with price-flash logic
│   ├── ConnectionStatus.tsx      Disconnect banner and live/reconnecting indicator
│   └── columns.ts                Shared column layout and row-height constants
├── lib/
│   ├── format.ts                 Price, percent and compact-volume formatting
│   └── assets.ts                 Asset display names and USD stablecoin list
├── App.tsx                       Page layout: header, table, status bar
├── main.tsx
└── index.css                     Tailwind entry and design tokens
```

## How it works

### Data source

On every (re)connect, the hook does two things:

1. It seeds the full market from `GET https://api.binance.com/api/v3/ticker/24hr`.
2. It applies deltas from `wss://stream.binance.com:9443/ws/!miniTicker@arr`.

The REST seed is needed because array streams only push symbols that changed in the last second. Without it, the table would fill in gradually over the first minutes. Updates are applied by event time, so an older message never overwrites newer data.

> **Why not `!ticker@arr`?** Binance still documents it, but it no longer publishes data: the socket opens and then stays silent. `!miniTicker@arr` carries the same rolling 24h window, minus bid/ask and trade count, which the table doesn't show. The parser accepts both payload formats, so you can switch back by passing `url` to `useBinanceStream`.

### Connection lifecycle

The hook moves between these states:

```
idle → connecting → connected ⇄ reconnecting → error
```

- **Backoff.** Retries use exponential backoff with jitter (1s base, 30s cap, 10 attempts), so many clients don't reconnect at the same moment.
- **Retry budget.** The attempt counter resets only after a message actually arrives. A server that accepts the connection and then drops it can't cause an endless fast retry loop.
- **Stall detection.** If no data arrives for 15s, the hook treats the socket as dead and reconnects.
- **Cleanup.** On unmount, all socket handlers, timers, animation frames, in-flight fetches and window listeners are removed.

### Rendering performance

- **One publish per frame.** Incoming messages update a mutable `Map` held in a ref. The hook copies it into React state at most once per animation frame, with a timer fallback for windows where `requestAnimationFrame` is throttled.
- **Stable object identity.** A ticker whose visible fields didn't change keeps the same object, so `React.memo` rows skip re-rendering.
- **Virtualization.** Only the visible rows plus a small overscan (about 40) are mounted, out of about 3,000.
- **No layout jitter.** Rows are a fixed 36px, the table uses `table-fixed`, and numbers use `font-mono tabular-nums`, so live updates never shift the layout.
- **Responsive typing.** Search uses `useDeferredValue`, so typing stays smooth while the list re-sorts each second.

## Configuration

`useBinanceStream` accepts these options:

| Option           | Default                      | Description                                  |
| ---------------- | ---------------------------- | -------------------------------------------- |
| `url`            | `!miniTicker@arr` stream     | WebSocket endpoint                           |
| `snapshotUrl`    | `/api/v3/ticker/24hr`        | REST seed endpoint; `null` disables it       |
| `enabled`        | `true`                       | Keep the socket closed when `false`          |
| `maxRetries`     | `10`                         | Failed attempts before entering `error`      |
| `baseDelayMs`    | `1000`                       | Initial backoff delay                        |
| `maxDelayMs`     | `30000`                      | Backoff ceiling                              |
| `staleTimeoutMs` | `15000`                      | Silence threshold that forces a reconnect    |

## Notes

- Market data comes from Binance's public endpoints and needs no API key. Binance may be unreachable in some regions.
- This is not financial advice. Prices are shown as delivered by the exchange.
