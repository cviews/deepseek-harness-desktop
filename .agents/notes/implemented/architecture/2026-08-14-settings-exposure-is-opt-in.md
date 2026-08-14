# Agent Note: Settings exposure is an explicit per-namespace opt-in

Status: implemented

English | [中文](2026-08-14-settings-exposure-is-opt-in.zh.md)

## Problem

The api-proxy served the Web configuration plane from a hardcoded allowlist — `WEB_SETTINGS_NAMESPACES` for the Web-preference and host-plane plugin sections, `PRODUCT_SETTINGS_NAMESPACES` for the product-owned sections — united with the model-provider namespaces. A plugin that registered a settings namespace outside that list stayed invisible to the configuration client (`settings-not-exposed`), so surfacing a plugin's own configuration required a source change in `packages/host/apiproxy`. That made the exposure boundary a per-plugin edit in a package plugins cannot touch.

## Decision

Exposure is an explicit per-namespace opt-in on `settings.register`. `SettingsRegisterOptions` gains `expose` (default `false`), reflected as `exposed` on every `SettingsDescriptor`, and `installSettingsSection` forwards it through. The api-proxy serves exactly the namespaces whose owner registered with `expose: true`, so a plugin distributed outside this repository surfaces its own configuration by opting in — with no change to `packages/host/apiproxy`. `WEB_SETTINGS_NAMESPACES`, `PRODUCT_SETTINGS_NAMESPACES`, and the `modelProviderNamespaces()` derivation are all gone; model-provider namespaces opt in like every other section. A namespace that was never registered, or that registered without the opt-in, answers `settings-not-exposed` — the same answer for both, so no caller can enumerate the registry by probing. The built-in Web-configurable sections (`locale`, `permission`, `ui-conversation`, `ui-theme`, `ui-onboarding`, `agent-presets`, `agent-loop`, `shell`, `web-search-deepseek`, `llm-deepseek`, `llm-pi-ai`) each pass `expose: true`; the host-only `agent-default-model` section does not. This supersedes the allowlist the [web-plugin-configuration note](../feature/2026-08-10-web-plugin-configuration.md) recorded, whose deferred "registration-time exposure declaration" alternative is what shipped here.

## Alternatives considered

- **Expose every registered namespace** — removes the allowlist with the smallest change, but a registration is not a safety declaration, so a plugin that buries a secret in a schema the redaction walker cannot reach would silently leak it to every client.
- **Keep the hardcoded allowlist** — preserves enumeration protection and a least-privilege surface, but every new plugin configuration section keeps requiring a source change in `packages/host/apiproxy`, the exact friction this change removes.

## Consequences

- A plugin distributed outside this repository can surface its own configuration on the Web settings page by registering with `expose: true`, with no change to `packages/host/apiproxy`; the plugin author evaluates its own section's safety and declares it.
- The registry-enumeration protection is preserved: `settings.describe` returns only opted-in namespaces, and a registered-but-unexposed namespace answers `settings-not-exposed` exactly like an unregistered one.
- Secret-role values stay redacted on the wire (`redactSecrets`), but the documented `redactSecrets` limitation applies unchanged: the walker follows only `object`/`dict`/`array`, so an owner must not opt in a namespace whose secret-role fields are reachable only through `union`/`intersection`/`transform`.
