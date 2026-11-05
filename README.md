# MyGo 音乐 · 插件 SDK

> **状态:草案(API v1 draft)。** 规范还在评审,尚无可运行的实现。

MyGo 音乐是一个支持插件扩展的跨平台原生播放器。播放内核(解码、缓存、淡入淡出、音量均衡、歌词、字幕)在宿主里,**音乐来源**(登录、搜索、歌单、取播放流……)由插件提供。

这个仓库是插件和宿主之间的契约,以及实现契约所需的工具:

| 目录 | 内容 |
|---|---|
| [`docs/`](docs) | 规范文档,按顺序读 |
| [`schema/`](schema) | JSON Schema(manifest 等),供 CI 与编辑器校验 |
| `plugin/`(待建) | 写插件用的 Go 库(`plugin`、`plugintest`、歌词解析助手) |
| `host/`(待建) | 宿主端的 WASM 运行时(wazero),应用和 CLI 共用,保证只有一份 ABI 实现 |
| `cmd/mygo-plugin/`(待建) | 命令行:`init` / `build` / `check` / `test` / `dev` |
| `template/`(待建) | 新插件模板,含发布用的 GitHub Actions |

## 阅读顺序

1. [总览与决策](docs/01-overview.md):架构、术语、范围、已定的决策
2. [Manifest](docs/02-manifest.md):插件的自我描述与权限声明
3. [ABI](docs/03-abi.md):宿主与 WASM 模块怎样通信
4. [数据模型](docs/04-data-model.md):歌曲、专辑、歌单……的统一结构
5. [能力与方法](docs/05-capabilities.md):插件能实现什么
6. [登录](docs/06-auth.md)
7. [安全模型](docs/07-security.md)
8. [打包、SDK 与发布](docs/08-packaging.md)
9. [宿主行为与现有功能的映射](docs/09-host-behavior.md):现有网易云功能在新架构里落在哪

插件市场的收录标准见 [`mygo-music/plugins`](https://github.com/mygo-music/plugins)。

## 许可证

[Apache-2.0](LICENSE)。写插件不会被迫开源,也不必使用与宿主相同的许可证。
