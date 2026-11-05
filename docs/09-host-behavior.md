# 09 · 宿主行为与现有功能的映射

这份文档回答两个问题:**宿主拿到插件的数据后怎么呈现**,以及**现有的网易云功能在新架构里各落在哪**。后者是检查规范有没有漏项的清单,也是重构的施工图。

## 9.1 原则

- 宿主**不认识任何具体站点**。界面上出现的每一个入口都来自某个能力或特性;没有能力就没有入口。
- 宿主的界面语言、布局、动效保持现状;换的是数据来源。
- 插件缺失、被停用、崩溃时,宿主降级而不是崩溃(§9.8)。

## 9.2 多来源:界面形态(已定 D10)

| | 方案 A:来源切换器 ← **采用** | 方案 B:多来源混排 |
|---|---|---|
| 侧边栏 | 顶部一个「来源:网易云音乐 ▾」切换器;发现、搜索、我的歌单、电台、播客都显示**当前来源**的内容 | 所有来源的歌单、推荐并列 |
| 搜索 | 在当前来源内搜 | 同时搜所有来源,按来源分组 |
| 队列 | 可混合不同来源的曲目(每首曲目都带 `plugin` 字段) | 同左 |
| 复杂度 | 低,与现有界面几乎一致 | 高:结果合并、排序、去重、错误隔离 |

v1 做 A。队列本来就按曲目存储,混合来源天然成立。B 可以以后在 A 之上加「搜索所有来源」。

**首次启动**时没有任何插件:显示「还没有音乐来源」页,从市场索引里列出官方推荐(一键安装),索引拿不到时给出「从 URL 安装」入口。**不把任何插件打包进应用**,避免应用版本与插件版本绑死。

## 9.3 能力 → 界面

| 能力 / 特性 | 宿主显示 |
|---|---|
| `account` | 侧边栏账号行(头像、昵称、登录 / 退出)。未登录且有需要登录的能力时,相应入口显示「登录后可用」 |
| `search` | 搜索框;搜索类型页签 = `search_types` |
| `search.suggest` | 输入时的建议下拉 |
| `feed` | 「发现」页:`feed.home` 的分区,按 `style` 渲染 |
| `library` | 「我的歌单」「收藏的歌单」分组;「我喜欢的音乐」(`role: liked`)置顶 |
| `like` | 曲目行与正在播放页的红心 |
| `playlist.edit` / `playlist.subscribe` | 新建/重命名/删除歌单;曲目右键「加入歌单」;歌单页的「收藏」 |
| `album` `artist` `playlist` `user` | 点击对应名称/卡片进入详情页;缺少相应能力时,该名称不可点击 |
| `follow` | 艺人页的「关注」按钮;「我关注的歌手」 |
| `lyrics` | 正在播放页的歌词、逐字、翻译 |
| `comments` | 正在播放页的评论页签(曲目有 `comment_thread` 才显示) |
| `radio` | 「电台」入口;`radio.modes` 作为模式切换 |
| `radio.similar` | 「心动模式」按钮 |
| `radio.dislike` | 电台中的「不喜欢」 |
| `podcast` | 「播客」入口 |
| `history` | 登录后提示「接着上次听」 |
| `stream` | 播放。没有它,曲目的行为是**不可播放**,并提示「该来源不支持播放」 |

## 9.4 现有功能 → 归属

**读法:** 「宿主」= 与站点无关,留在宿主;「插件」= 搬进网易云插件;右列是承载它的方法。

| 现有功能 | 归属 | 承载 |
|---|---|---|
| 扫码登录;登录态存系统凭据 | 插件 | `account`(qr);会话在 cookie 罐 |
| 登录后接着播放等待中的歌 | 宿主 | `stream.resolve` 的 `unauthenticated` → 登录 → 重试 |
| 每日推荐 | 插件 | `feed.home` 的 `daily` 分区(`hero`,`play_all`) |
| 推荐歌单 | 插件 | `feed.home` 的 `grid` 分区 |
| 排行榜 | 插件 | `feed.home` 的 `chart` 分区(`Playlist.preview`) |
| 我的歌单 / 收藏的歌单 | 插件 | `library.playlists` |
| 搜索:单曲、歌手、专辑、用户、播客(歌单) | 插件 | `search.query`,`search_types` |
| 搜索建议 | 插件 | `search.suggest` |
| 搜索历史 | **宿主** | 本地保存,按来源区分 |
| 歌手页:热门、最新、专辑 | 插件 | `artist.get` + `tab.load`(`tracks` `albums`) |
| 歌手页:动态 | 插件 | `tab.load`(`posts`,`Post`) |
| 关注歌手;我关注的歌手 | 插件 | `follow`:`artist.follow`、`library.artists` |
| 专辑页、用户页 | 插件 | `album.get`、`user.get` |
| 播客:浏览、订阅、收听 | 插件 | `podcast.*`、`podcast.subscribe` |
| **本机转写播客字幕(Qwen3-ASR)** | **宿主** | 作用于 `kind: "episode"` 曲目的音频流;模型与原生库属宿主(见 §9.7) |
| 私人 FM(推荐/熟悉/探索) | 插件 | `radio.modes`、`radio.next` |
| 心动模式 | 插件 | `radio.similar` |
| FM 里「不喜欢」 | 插件 | `radio.dislike` |
| 音质:标准…Hi-Res | 宿主选档,插件映射 | `stream.resolve(quality)`、`qualities` |
| 边下边播、拖动进度、Range | 宿主 | 播放器;要求流支持 `Range` |
| 听过的缓存在本机 | 宿主 | 缓存键 = (插件 id, 曲目 id, 档位),见 §9.6 |
| 音量均衡 | 宿主 | 读 `Stream.gain_db` / `peak` |
| 暂停淡出淡入、拖动无杂音 | 宿主 | 播放器 |
| 记住队列与进度 | 宿主 | 存曲目快照(含 `plugin`、`extra`),见 §9.5 |
| 逐字歌词与翻译 | 插件产出结构,宿主渲染 | `lyrics.get` → `lines` |
| 「律动」视图 | 宿主 | 依赖歌词与封面 |
| 歌曲评论、点赞评论 | 插件 | `comments.list`、`comments.like` |
| 刘海播放器、托盘、系统媒体控制 | 宿主 | 播放器状态 |
| 喜欢歌曲;喜欢的状态 | 插件 | `like`:`track.like`、`library.liked_ids` / `track.liked_state` |
| 新建 / 重命名 / 删除歌单;加入、移出曲目 | 插件 | `playlist.edit` |
| 收藏他人歌单到侧边栏 | 插件 | `playlist.subscribe` |
| 接着上次听 | 插件产出,宿主呈现 | `history`:`library.recent` |
| 浅色 / 深色 / 跟随系统;设置;更新 | 宿主 | |

## 9.5 持久化

宿主保存的每一首曲目是**完整快照**:插件当时返回的曲目 JSON(含 `extra`),外加宿主写入的 `plugin` 字段(来源的 `id`),与曲目平铺在同一个对象里。

| 持久化内容 | 存什么 |
|---|---|
| 播放队列与进度 | 曲目快照列表 + 当前位置 + 进度 |
| 搜索历史 | `{ plugin, query }` |
| 最近播放(本地) | 曲目快照 |

快照意味着**插件更新后旧数据仍能还原**,也意味着 `extra` 要保持向后兼容(插件自己负责)。曲目对应的插件**未安装 / 已停用**时:在队列里显示为灰色「插件未安装」,附「安装」按钮,播放时跳过。

## 9.6 缓存

- 键:`(plugin id, track.id, quality)`,文件名取其哈希。
- 流地址过期(`expires_ms`)或播放失败 → 重新 `stream.resolve`,最多一次,然后报错。
- 命中缓存且档位足够时**不调用插件**,离线也能播放已缓存的曲目(这时插件即使被停用也能放缓存)。
- 缓存按插件分目录,卸载插件时询问是否清除。

## 9.7 宿主独占的重功能:字幕转写

本机语音转写(sherpa-onnx、Qwen3-ASR、Silero VAD)与具体来源无关,**保留在宿主**:

- 触发条件:曲目 `kind == "episode"`。
- 输入:宿主已通过 `stream.resolve` 拿到的音频流。
- 原生库与模型按需下载。库的下载地址是应用自己的发布资源,与插件无关。
- 字幕缓存键:`(plugin id, track.id)`。

## 9.8 失败与降级

| 情形 | 行为 |
|---|---|
| 调用超时 / 实例崩溃 | 丢弃实例,提示可重试;同一方法连续失败 3 次后,界面显示「该来源暂时不可用」而不是反复重试 |
| `unauthenticated` | 清除登录状态,提示登录 |
| `unavailable`(取流) | 该曲目标为不可播放,自动跳到下一首 |
| 插件被吊销 | 停用,提示;缓存仍可播放 |
| 插件加载失败(损坏、`api` 不兼容) | 在插件管理页显示原因,不影响其他来源 |

## 9.9 插件管理界面

设置里的「插件」页:已安装列表(图标、名称、版本、来源层级)、启用/停用、更新、卸载;详情页显示权限与能力、该来源的账号状态、插件自己的设置(`settings`);「插件市场」页浏览索引;「从 URL / 文件安装」。

## 9.10 宿主内部结构(为重构服务)

这是第 2 阶段要做的事,先写在这里。

```
internal/source/                 与站点无关的模型和接口
  model.go                       Track、Album、Artist、Playlist…(字段与 JSON 与 04 号文档一致)
  source.go                      Source 接口 + 各能力的子接口(Searcher、Streamer、…)
internal/source/netease/         第 2 阶段:现有 internal/netease 的适配器(临时)
internal/source/wasm/            第 3 阶段:基于 plugin-sdk/host 的适配器(取代上一个)
internal/app, internal/player    只依赖 internal/source
```

- `internal/source` 里的类型与 JSON 标签直接对应规范,wasm 适配器**解码即用**,不再转换。
- UI 与播放器**只依赖 `internal/source`**。第 2 阶段结束时,`internal/app` 与 `internal/player` 里不再出现 `netease` 的导入。
- 现有测试用包级变量换掉网络(`likeSong`、`copyText` 等);重构后它们换掉的是 `Source` 的实现——测试里提供一个内存里的假 `Source`,UI 测试不需要 wasm。这与 AGENTS.md 的测试约定一致。

## 9.11 迁移计划

| 阶段 | 内容 | 验收 |
|---|---|---|
| 1 | 本规范评审通过 | 你确认 |
| 2 | 宿主内部抽象:新增 `internal/source`,现有 `netease` 包作为内置适配器;UI/播放器改依赖接口;ID 改为字符串;缓存键、会话快照、设置里的音质改用新模型 | `go test ./internal/...` 全绿;行为不变;`internal/app` `internal/player` 无 `netease` 导入 |
| 3 | `plugin-sdk`(`plugin/`、`host/`、CLI)和 `plugin-netease`;宿主接入 wazero,移除内置 `netease` 包;完成待实测 S1–S3 | 用 `.mgp` 安装网易云插件后,功能与阶段 2 一致 |
| 4 | 插件管理界面;`plugins` 登记处的 CI 与 `index.json` 生成;市场页;URL 安装 | 提一个测试插件的 PR → 合并 → 出现在市场 |
