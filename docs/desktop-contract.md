# Desktop 契约与验证

本插件基于 [anywhere-labs/dsh-desktop](https://github.com/anywhere-labs/dsh-desktop) 的插件规范编写，落地文档是该仓库的 [`docs/plugin-development.md`](https://github.com/anywhere-labs/dsh-desktop/blob/master/docs/plugin-development.md) 与包级 service 合同 [`dsh-plugin-desktop/docs/plugin-services.md`](https://github.com/anywhere-labs/dsh-desktop/blob/master/dsh-plugin-desktop/docs/plugin-services.md)。

## 采用的接口

| 接口 | 用途 | 说明 |
| --- | --- | --- |
| `desktopProfiles.current` | 活动 profile 的名称与绝对目录 | 一代之内不变；不从 argv、`ctx.baseUrl`、settings 或 `$DSH_HOME` 推断 |
| `desktopProfiles.list()` | 可选 profile 清单 | 只读；每项含 `exists`、`webCapable`、`bundles` 与可选 `problem` |
| `desktopProfiles.select(name)` | 持久化并请求一次有序重启 | 校验通过后才调用；结果用 `restartRequired` 表达重启边界 |
| `ctx.get('sandboxPolicy')` | 只读会话判定 | 只读会话不能切换 |
| `ctx.get('approval')` | 切换前的显式确认 | 带 agent 的调用必须 `allowed-once` |

未采用 `desktopPnpm`：桌面端的包操作归 [`dsh-desktop-suite`](https://github.com/hzxwonder-dsh-plugins/dsh-desktop-suite)，本插件不做依赖或 bundle 变更，因此不需要包服务。profile 的清单与 `node_modules` 只读。

## 规范检查表对应

| 规范要求 | 实现 |
| --- | --- |
| 变更必须来自显式用户或管理员动作 | `select` 走 approval，或由用户直接输入命令；只读会话直接拒绝 |
| 不跨代保留 service 引用 | 每次动作重新读取 `desktopProfiles`，不缓存 profile 或选择结果 |
| 声明清晰，不依赖运行时巧合 | `inject` 只声明实际使用的 `tools`、`commands`、`desktopProfiles` |
| 切换前校验而非事后补救 | 目标必须存在于 `list()`，且 `exists`、`webCapable` 为真、无 `problem` |
| 重启语义显式 | 返回值带 `restartRequired: true`，文档说明当前代结束即失效 |
| 不重复他人的组合 | 组合行仍由 `dsh-plugin-workbench` 插入，本插件只报告落位情况 |
| 不修改宿主拥有的状态 | 不写 profile 文件；持久化选择由 Desktop 完成 |

## 实机验证

在运行中的 DSH Desktop 上验证（profile patch 已启用本插件）：

1. `/desktop-workbench status`：应给出活动 profile 名称与目录，并正确报告 `dsh-plugin-workbench` 是否已声明、已安装、已进入 bundle 列表。
2. `/desktop-workbench list`：应列出全部 profile；带 `problem` 或 `webCapable: false` 的条目应显示出来且不可选择。
3. `/desktop-workbench select <name>`：切换到一个健康 profile，确认 Desktop 重启并进入该 profile；重启后 `status` 显示的应是新 profile。
4. 对不存在、缺失或带诊断问题的 profile 执行 `select`：应以 `DESKTOP_WORKBENCH_UNKNOWN_PROFILE`、`DESKTOP_WORKBENCH_PROFILE_MISSING`、`DESKTOP_WORKBENCH_PROFILE_NOT_WEB_CAPABLE` 或 `DESKTOP_WORKBENCH_PROFILE_UNHEALTHY` 拒绝，且不产生重启。
5. 在只读会话里执行 `select`：应以 `DESKTOP_WORKBENCH_READ_ONLY` 拒绝。

单元测试不覆盖真实启动器：`npm test` 用假 `desktopProfiles` 覆盖校验、门控与结果形状。
