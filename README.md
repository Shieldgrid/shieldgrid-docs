# Shieldgrid Docs

Documentation for the **Shieldgrid** platform — a single-pane-of-glass SOC operations system.

## What's here

- **Architecture** — system design, connector abstraction, data model
- **Install / Upgrade** — getting a Shieldgrid instance running
- **Connector guide** — how to write a new connector (the exact checklist to add support for a new security tool)
- **User guide** — operator/analyst workflows (alert triage, case management)

## Why a separate repo

Docs update on a different cadence than code, and this keeps the [shieldgrid-core](https://github.com/Shieldgrid/shieldgrid-core) and [shieldgrid-web](https://github.com/Shieldgrid/shieldgrid-web) READMEs short — they link here instead of duplicating install/architecture detail that would otherwise drift out of sync.

## Structure

```
docs/
├── getting-started/
│   └── install-upgrade.md
├── architecture/
│   ├── system-design.md
│   ├── data-flows.md
│   └── database-schema.md
├── integrations/
│   └── adding-a-connector.md
└── user/
    ├── operator-quickstart.md
    └── admin-quickstart.md
```

## Status

🚧 Early — architecture doc ported over from initial design phase. Install guides will follow once Phase 0 of shieldgrid-core is runnable end-to-end.

## Contributing

If something's unclear or out of date, please open an issue — docs are treated as a living system, not a one-time write-up.

## License

Documentation content is licensed [CC BY 4.0](LICENSE) unless otherwise noted. Code samples within the docs follow the AGPL-3.0 license of the main project.
