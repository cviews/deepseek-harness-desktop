# Agent Note: Settings 暴露是按命名空间的显式 opt-in

Status: implemented

[English](2026-08-14-settings-exposure-is-opt-in.md) | 中文

## 问题

api-proxy 曾以一份硬编码白名单服务 Web 配置面——`WEB_SETTINGS_NAMESPACES` 覆盖 Web 偏好与宿主平面插件分节，`PRODUCT_SETTINGS_NAMESPACES` 覆盖产品持有分节——再与模型提供方命名空间取并集。注册了名单之外 settings 命名空间的插件，对配置客户端不可见（`settings-not-exposed`），因此要让插件自己的配置出现，就得改动 `packages/host/apiproxy` 的源码。这让暴露边界变成了对每个插件的一次、且只能在插件碰不到的包里进行的源码改动。

## 决策

暴露是 `settings.register` 上的按命名空间显式 opt-in。`SettingsRegisterOptions` 新增 `expose`（缺省 `false`），并在每个 `SettingsDescriptor` 上反映为 `exposed`，`installSettingsSection` 亦将其透传。api-proxy 只服务那些 owner 以 `expose: true` 注册的命名空间，因此在本仓库之外分发的插件只需选择暴露，即可让自己的配置出现——无需改动 `packages/host/apiproxy`。`WEB_SETTINGS_NAMESPACES`、`PRODUCT_SETTINGS_NAMESPACES` 以及 `modelProviderNamespaces()` 派生全部移除；模型提供方命名空间与其他分节一样选择暴露。从未注册、或注册但未选暴露的命名空间，都会得到 `settings-not-exposed`——两者同一个答复，因此没有调用方能靠逐个探测把注册表枚举出来。内置的可 Web 配置分节（`locale`、`permission`、`ui-conversation`、`ui-theme`、`ui-onboarding`、`agent-presets`、`agent-loop`、`shell`、`web-search-deepseek`、`llm-deepseek`、`llm-pi-ai`）均传入 `expose: true`；仅 host 侧使用的 `agent-default-model` 分节不传。这取代了 [web-plugin-configuration 笔记](../feature/2026-08-10-web-plugin-configuration.md)所记录的白名单；该笔记中暂缓的「注册期暴露声明」备选正是本笔记落地的方案。

## 曾考虑的替代方案

- **暴露所有已注册命名空间**——以最小改动移除白名单，但注册本身并不是安全声明，因此把 secret 埋进脱敏遍历器无法到达的 schema 里的插件，会把它静默泄露给每个客户端。
- **保留硬编码白名单**——保留枚举保护与最小权限面，但每个新的插件配置分节仍需改动 `packages/host/apiproxy` 源码，正是本次改动要消除的摩擦。

## 后果

- 在本仓库之外分发的插件只需以 `expose: true` 注册，就能让配置出现在 Web 设置页，无需改动 `packages/host/apiproxy`；插件作者自行评估自己分节的安全性并作出声明。
- 注册表枚举保护得以保留：`settings.describe` 只返回已选暴露的命名空间，已注册但未暴露的命名空间与未注册的命名空间得到的是同一个 `settings-not-exposed`。
- secret 角色值在线上仍会脱敏（`redactSecrets`），但已文档化的 `redactSecrets` 限制不变：walker 只跟随 `object`/`dict`/`array`，因此 owner 不得选择暴露那些 secret 角色字段只能经由 `union`/`intersection`/`transform` 触及的命名空间。
