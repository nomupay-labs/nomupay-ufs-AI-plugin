# Nomupay merchant AI integration plugin

> **Version:** 0.1.0 · **Author:** Nomupay · **Category:** Development

A multi-platform agent plugin repository for GitHub Copilot and Claude Code.
Currently ships one plugin for the Nomupay Ecommerce API, with support for C#,
JavaScript/TypeScript, and PHP.

---

## Plugins

| Plugin | Description |
| ------ | ----------- |
| [`nomupay-merchant-integration`](plugins/nomupay-merchant-integration/README.md) | Language-specific agents (C#, JS/TS, PHP) and cross-language skills for the Nomupay Ecommerce API — authentication, signing, verification, webhooks, and payment operations. |

---

## Installation

This plugin is distributed via the
[`pezaio-testing/merchant-agents`](https://github.com/pezaio-testing/merchant-agents)
GitHub repository and works across VS Code, GitHub Copilot CLI, and Claude Code.

### VS Code

Add `pezaio-testing/merchant-agents` to the `chat.plugins.marketplaces` setting
in VS Code, then search `@agentPlugins` in the Extensions view and install
`nomupay-merchant-integration`. See [Agent plugins in VS
Code](https://code.visualstudio.com/docs/agent-customization/agent-plugins#_configure-plugin-marketplaces)
for full instructions.

### GitHub Copilot CLI

Register the marketplace with `copilot plugin marketplace add
pezaio-testing/merchant-agents` (marketplace name: `nomupay-copilot-plugins`),
then install with `copilot plugin install
nomupay-merchant-integration@nomupay-copilot-plugins`. See [Finding and
installing
plugins](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-finding-installing)
for full instructions.

### Claude Code

Register the marketplace with `/plugin marketplace add
pezaio-testing/merchant-agents` (marketplace name: `nomupay-claude-plugins`),
then install with `/plugin install
nomupay-merchant-integration@nomupay-claude-plugins` and run `/reload-plugins`
to activate. See [Discover and install
plugins](https://code.claude.com/docs/en/discover-plugins) for full
instructions.

## Key resources

| Resource         | URL                                                                  |
| ---------------- | -------------------------------------------------------------------- |
| OpenAPI spec     | `https://docs.nomupay.com/openapi/online-payments.yaml`              |
| Integration docs | `https://docs.nomupay.com/payments/integration-up-api/`              |
| API Reference    | `https://docs.nomupay.com/payments/integration-up-api/api-reference` |
| Security / Auth  | `https://docs.nomupay.com/payments/integration-up-api/security`      |
| Webhooks         | `https://docs.nomupay.com/payments/integration-up-api/webhooks`      |

| Environment       | Base URL                          |
| ----------------- | --------------------------------- |
| Sandbox (default) | `https://api.sandbox.nomupay.com` |
| Live              | `https://api.nomupay.com`         |

> All API requests must be made over **HTTPS (TLS 1.2)**.

---

## License

This plugin is released under the [MIT License](LICENSE). Copyright © 2026
Nomupay.

> **AI disclaimer:** This plugin uses AI language models to generate code and
> integration guidance. Outputs are experimental and not guaranteed to be
> correct, secure, or complete. Review all AI-generated code before deploying to
> production. Nomupay accepts no liability for errors, security vulnerabilities,
> or financial losses arising from AI-generated content produced by this plugin.
