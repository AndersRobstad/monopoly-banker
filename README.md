# Monopoly Banker

A simple Svelte 5 application for managing money in a Monopoly game.

## Features

- **Dynamic players (min 2)**: Add/remove players with unique IDs; names editable
- **Per-player $2M**: Quick button to give a single player $2M (passing GO)
- **Add/Withdraw**: Manage individual balances with formatted inputs
- **Transfers**: Move money between players with validation
- **History**: Logs all transactions (adds, withdraws, transfers, player changes)
- **Persistence**: State saved in `localStorage`
- **New game/reset**: Three-dot menu to start a new game (set start balance) or fully reset

## Getting Started

1) Install dependencies:
```bash
npm install
```

2) Start the development server:
```bash
npm run dev
```

3) Open your browser to the URL shown in the terminal (usually `http://localhost:5173`)

## Build for Production (local)

```bash
npm run build
```

The built files will be in the `dist` directory.

## Deploy to Vercel

1) Install the Vercel CLI (if not already):
```bash
npm install -g vercel
```

2) Deploy (from project root):
```bash
vercel        # first time to link project
vercel --prod # production deployment
```

Vercel uses `vercel.json`:
- `buildCommand`: `npm run build`
- `outputDirectory`: `dist`
- `framework`: `vite`

Alternatively, use the Vercel dashboard, import the repo, set `Build Command` to `npm run build`, and `Output Directory` to `dist`.

