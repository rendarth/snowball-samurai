# Snowball Samurai 🗡️❄️

> **A self-hosted, privacy-first debt payoff engine built in Rust.**

Snowball Samurai helps you aggressively slice away debt using the **Snowball**, **Avalanche**, or **Velocity Banking** strategies. Upload your financial statements, and it tells you *exactly* how much to pay, *to which account*, and *when* — with smart nudges when you have extra cash to throw at your debt.

## Key Features

- 📄 **PDF Statement Ingestion** — Drop in your credit card and bank statements; the app extracts balances, minimums, APRs, and due dates automatically.
- ❄️ **Debt Snowball** — Pay off the smallest balance first, rolling freed minimums into the next target.
- 🏔️ **Debt Avalanche** — Pay off the highest APR first to minimize total interest.
- ⚡ **Velocity Banking** — Use a Line of Credit to chunk down debt principal while parking income to reduce daily interest.
- 📊 **Side-by-Side Comparison** — See total interest paid, debt-free date, and monthly impact for all three strategies.
- 🔔 **Smart Reminders** — Get nudged when surplus cash is detected: *"You have ~$340 extra — apply it to Visa 4821."*
- 🔒 **100% Local** — Runs entirely on your network. Zero outbound connections by default. Your financial data never leaves your server.
- 🐳 **Docker-Ready** — Single-container deployment, perfect for Unraid, Synology, or any Docker host.

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Rust |
| Web Framework | Actix Web |
| Database | SQLite (rusqlite) |
| PDF Parsing | pdf-extract (pure Rust) |
| Frontend | Server-rendered HTML + HTMX + Alpine.js + Tailwind CSS |
| Notifications | Discord / Ntfy / Gotify webhooks (optional) |

## Quick Start (Docker)

```bash
docker run -d \
  --name snowball-samurai \
  -p 8080:8080 \
  -v /path/to/your/data:/data \
  snowball-samurai:latest
```

Then open `http://your-server:8080` in any browser.

## Documentation

- **[SPEC.md](./SPEC.md)** — Full product specification and requirements.
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** — Technical architecture, database schema, algorithms, and project structure.

## Status

🚧 **Under active development** — Not yet ready for use.

## License

MIT
