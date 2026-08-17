# MCMultiLoginCompat

> Bukkit / Spigot / Paper / Purpur 服务端的多账户正版登录兼容插件。

## 项目简介

让 bukkit 类服务器**同时支持正版玩家与多个 Yggdrasil（皮肤站）账号登录**，
而**不需要开启离线模式**（`online-mode=false`）。适用于你想让 LittleSkin /
自建 authlib-injector 玩家与正版玩家同服的场景。

本项目移植于项目[`mc-multilogin-compat-mod`](https://github.com/wifi-left/mc-multilogin-compat-mod)，意在使Bukkit系服务器能够拥有同一能力。

这是**纯版**：只包含「多账户登录兼容」核心能力。

## 特性

- **接管 `hasJoinedServer`**：正版 + 多 Yggdrasil 并存登录，无需关闭在线验证。
- **两种验证来源，任选其一（也可同时配）**：
  - **外部模式**：对接你已有的 [`MC-MultiLogin-service`](https://github.com/wifi-left/MC-MultiLogin-service)（在 `config.yml` 填 `api-url`）。
  - **自包含模式**：把验证逻辑直接内嵌进插件，靠 `config.json` 的 `method[]`
    配置，**无需再起任何外部 HTTP 服务**。
- **详细登录失败原因**：在 Netty 层拦截登录断开包，把原版笼统的
  `Authentication servers are down` 替换为认证服务返回的真实原因
  （名称重复 / 封禁 / 皮肤站不支持 / 登录过快等）。
- **名称重复自动改名重试**：认证返回 `DUPLICATE_NAME` 时，自动用返回的可用名
  重试一次，并把「原名 → 新名」记进 `renames.json`。
- **单 JAR 全版本兼容**：产物目标字节码 Java 8，`api-version: 1.13`，
  可在 1.8 ~ 最新 Purpur 上加载。

## 工作原理

```
MC 服  MCMultiLoginCompat（本插件，load: STARTUP 尽早启用）
  ├─ SessionServiceHook：动态代理包装 MinecraftSessionService，接管 hasJoinedServer
  │     → 正版 / 多 Yggdrasil 玩家并行验证，均通过才放行
  └─ LoginPipelineHook：Netty 登录通道注入
        → 拦截「登录阶段断开包」，把笼统错误替换为认证服务的真实原因
```

验证实际怎么跑（满足任一即接管登录，都不满足则安全 fail-open）：

- `config.yml` 配置了 `api-url` → 走外部 `MC-MultiLogin-service`（HTTP 请求）。
- `config.json` 有 `method[]` → 走内嵌 `VerifyService`（进程内，无需外部服务）。
- 两者都没有 → 插件只注册指令、不接管登录（避免服主误以为在验证）。

## 配置

### `config.yml`（插件主配置）

| 字段 | 说明 | 默认 |
| --- | --- | --- |
| `api-url` | 外部 `MC-MultiLogin-service` 某登录方式基础地址（即其 `method[].url`）。留空则不挂钩外部服务 | 空 |
| `auto-rename` | 认证返回 `DUPLICATE_NAME` 时，自动用返回的可用名重试一次并记入 `renames.json` | `true` |
| `request-timeout-seconds` | 每次验证请求超时（秒） | `10` |
| `shutdown-on-failure` | 会话代理安装失败时是否直接关闭服务器（宁可不开服也不裸奔） | `false` |
| `debug` | 是否输出每次被拦截登录的额外日志（调试用） | `false` |

### `config.json`（自包含验证，仅 `method[]` 模式用）

首次运行自动生成模板。为了**安全 fail-open**，默认 `method` 是**空数组**，
插件不会接管登录；需要自包含验证时，手动填入至少一个 method。

```json
{
  "apis": [
    { "id": "littleskin", "name": "LittleSkin", "root": "https://littleskin.cn/api/yggdrasil" },
    { "id": "original", "name": "Official" }
  ],
  "default": "original",
  "method": [
    {
      "url": "/login/my",
      "name": "myserver",
      "secret": "your_secret_key_here",
      "handles": ["littleskin", "original"]
    }
  ],
  "errorMessages": { "...": "..." }
}
```

- `method[]`：验证入口。空数组 = 不自包含验证。
- `handles`：该入口允许哪些皮肤站（与上方 `apis[].id` 对应）。
- `url` / `secret`：保留与原 Node 版 `MC-MultiLogin-service` 一致的字段语义，
  合并进插件后主要作 method 标识与缓存分目录用。
- 填好 `method[]` 后**重启**（或 `/multilogin reload`）即可启用自包含验证。

## 指令

`/multilogin <status|reload|renames>`（别名 `mlogin` / `mml`，权限 `mcmultilogin.admin`，默认 OP）

- `status`：查看会话代理 / Netty 注入状态与已配置验证来源。
- `reload`：热重载 `config.yml` 与 `config.json`（会话代理本身不受影响）。
- `renames`：查看「原名 → 新名」自动改名记录。

## 构建

```bash
# 需要 JDK 8+（产物目标字节码 Java 8，适配 1.8 ~ 最新 Purpur）
./gradlew build
# 产物：build/libs/mc-multilogin-compat-bukkit-<version>.jar
```

将 JAR 放入服务端 `plugins/` 重启即可。首次运行自动生成
`config.yml` / `config.json` / `renames.json`。

## 开源协议

本项目以 **GNU Affero General Public License v3.0 (AGPL-3.0)** 发布，详见
[LICENSE](./LICENSE)。
