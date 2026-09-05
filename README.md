# V2Hub Documentation

> Official documentation for the V2Hub developer ecosystem

V2Hub Documentation provides guides, concepts, usage examples, and API references for building applications and integrations with the V2Hub ecosystem.

### 🌐 [Part of the V2Hub Ecosystem](https://github.com/nestthub/nestthub/tree/main/ecosystems/v2hub)

This repository contains the official documentation for the developer-facing V2Hub components:

- **[V2Hub](https://github.com/nestthub/v2hub-core)** — Python client library for the V2Hub API — [PyPI](https://pypi.org/project/v2hub/)
- **[V2Hub Admin](https://github.com/nestthub/v2hub-admin)** — administration extension for user, provider, and system management — [PyPI](https://pypi.org/project/v2hub-admin/)
- **[V2Hub CLI](https://github.com/nestthub/v2hub-cli)** — command-line interface for V2Hub — [PyPI](https://pypi.org/project/v2hub-cli/)

The **[V2Hub API](https://github.com/nestthub/v2hub-api)** has its own documentation and operator resources available at [v2hub.link](https://v2hub.link).

---

## 📚 Documentation

The full documentation is available at [**v2hub.dev**](https://v2hub.dev).

It covers:

- Getting started and installation
- Client configuration and authentication
- Subscription and provider management
- CLI usage and commands
- Administration and integrations
- Guides and practical examples
- Concepts and architecture
- API and type references

---

## 🧩 V2Hub

[V2Hub](https://github.com/nestthub/v2hub-core) is a Python client library for the V2Hub API with async and sync support, typed models, retry and circuit-breaker support, provider functionality, and production-oriented features.

**PyPI:** [v2hub](https://pypi.org/project/v2hub/)

## 🔐 V2Hub Admin

[V2Hub Admin](https://github.com/nestthub/v2hub-admin) provides privileged administration capabilities for V2Hub, including user and provider management, authorization handling, usage statistics, IP bans, and whitelist management.

**PyPI:** [v2hub-admin](https://pypi.org/project/v2hub-admin/)

## 🖥️ V2Hub CLI

[V2Hub CLI](https://github.com/nestthub/v2hub-cli) provides a terminal-based interface for V2Hub, including subscription management, provider operations, shell autocompletion, and optional administrative commands.

**PyPI:** [v2hub-cli](https://pypi.org/project/v2hub-cli/)

---

## 🛠️ Development

The documentation is built with [Zensical](https://zensical.org/).

Install dependencies:

```bash
uv sync
```

Run the documentation locally:

```bash
uv run zensical serve
```

Build the documentation:

```bash
uv run zensical build --clean --strict
```

---

## 📄 License

The documentation is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/).

See [`LICENSE`](LICENSE) for the full license text.
