# Composio Setup Guide

A standalone guide for getting AI agents and their users set up with [Composio](https://composio.dev) — the platform that gives your agent access to 1000+ external apps (Gmail, Slack, GitHub, Notion, Google Calendar, HubSpot, Salesforce, etc.) through a unified API.

## What's inside

- **[composio-setup-guide.md](./composio-setup-guide.md)** — the full guide
  - How to create a Composio account
  - How to get a **Project API Key** (the modern `ak_*` format)
  - How to install the `@composio/core` SDK and create a CLI shim
  - How to link your first external account (e.g. Gmail)
  - How to execute your first tool
  - Troubleshooting common issues
  - Why this guide exists (and why it avoids the official Composio CLI)

## Why this guide is needed

The Composio CLI (`composio` command) currently ships with a header-auth bug that rejects modern `ak_*` keys — every command except `whoami` returns 401. This guide works around that by going straight to the `@composio/core` SDK, which sends the correct headers. Any AI agent can follow this guide on a fresh machine in about 5 minutes.

## Who this is for

- **Heyron users** — agents built on the [Heyron](https://heyron.ai) platform that want to integrate external services
- **Anyone running an AI agent** who wants a reliable Composio setup

## License

MIT — see [LICENSE](./LICENSE).

## Contributing

Found an error? Have an improvement? Open an issue or PR.