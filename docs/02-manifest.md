# 02 · Manifest

每个插件包根目录有一个 `manifest.json`。它是插件对宿主、对市场、对用户的**唯一自我描述**,宿主不加载 wasm 就能据此展示安装对话框。机器可校验的版本见 [`schema/manifest.schema.json`](../schema/manifest.schema.json)。

## 2.1 示例

```json
{
  "$schema": "https://mygo-music.github.io/plugin-sdk/schema/manifest.schema.json",
  "api": 1,
  "id": "netease",
  "name": "网易云音乐",
  "version": "1.0.0",
  "description": "第三方网易云音乐来源:扫码登录、推荐、搜索、歌单、电台、播客。",
  "authors": [{ "name": "MyGo Music", "github": "mygo-music" }],
  "repository": "https://github.com/mygo-music/plugin-netease",
  "license": "AGPL-3.0-only",
  "icon": "icon.png",
  "wasm": "plugin.wasm",
  "min_host": "0.2.0",

  "capabilities": ["search", "stream", "album", "artist", "playlist", "user",
                   "lyrics", "account", "library", "feed", "radio", "comments", "podcast"],
  "features": ["search.suggest", "like", "history", "playlist.edit", "playlist.subscribe",
               "follow", "podcast.subscribe", "radio.similar", "radio.dislike", "comments.like"],
  "search_types": ["track", "album", "artist", "playlist", "user", "podcast"],
  "qualities": ["standard", "higher", "exhigh", "lossless", "hires"],

  "permissions": {
    "network": { "hosts": ["music.163.com", "interface.music.163.com"] },
    "media":   { "hosts": ["*.music.126.net", "http://*.music.126.net"] }
  },

  "i18n": {
    "en": { "name": "NetEase Cloud Music", "description": "Unofficial NetEase Cloud Music source." }
  }
}
```

## 2.2 字段

| 字段 | 必填 | 说明 |
|---|---|---|
| `api` | ✓ | 整数,契约主版本,本规范为 `1` |
| `id` | ✓ | 全局唯一标识,`^[a-z][a-z0-9-]{1,31}$`。同时是数据命名空间:数据目录、钥匙串条目、曲目全局引用都以它为前缀。**发布后不可更改**。保留字:`mygo`、`host`、`core`、`system`、`local`、`builtin`、`dev` |
| `name` | ✓ | 显示名,≤ 40 个字符 |
| `version` | ✓ | semver,不带 `v` 前缀。Git 标签必须是 `v` + 此值 |
| `description` | ✓ | 一句话,≤ 200 个字符,纯文本 |
| `authors` | ✓ | `[{ name, github?, url? }]`,至少一项。`github` 用于登记处核对提交者 |
| `repository` | ✓ | 源码仓库 URL。市场收录要求是 `https://github.com/<owner>/<repo>` |
| `license` | ✓ | SPDX 标识符 |
| `homepage` | | 主页 URL |
| `icon` | ✓ | 包内 PNG 路径,256×256,≤ 256 KiB |
| `wasm` | ✓ | 包内 wasm 路径,≤ 32 MiB |
| `min_host` | | 需要的最低宿主版本(semver) |
| `capabilities` | ✓ | 实现的能力,见 §2.3。至少一项 |
| `features` | | 能力内的可选方法,见 §2.3 |
| `search_types` | 搜索必填 | 声明 `search` 时必填:支持的搜索类型 |
| `qualities` | 取流必填 | 声明 `stream` 时必填:支持的音质档位,从低到高 |
| `permissions` | ✓ | 见 §2.4。不需要任何网络访问时写 `{}` |
| `settings` | | 用户可配置项,见 §2.5 |
| `i18n` | | 其他语言的 `name` 与 `description`,键为 BCP 47 语言标签 |

## 2.3 能力与特性

`capabilities` 声明「实现了哪组方法」,宿主据此决定显示哪些入口(见 [09 号文档](09-host-behavior.md))。**声明了能力就必须实现它的必选方法**,`mygo-plugin check` 与登记处 CI 会逐一探测。

| 能力 | 必选方法 | 宿主因此显示 |
|---|---|---|
| `search` | `search.query` | 搜索框与搜索页 |
| `stream` | `stream.resolve` | 播放。**没有它,该来源的歌曲不可播放** |
| `album` `artist` `playlist` `user` | `<name>.get` | 对应实体的详情页 |
| `lyrics` | `lyrics.get` | 歌词 |
| `account` | `auth.*` | 登录 / 账号 |
| `library` | `library.playlists`(依赖 `account`) | 「我的歌单」 |
| `feed` | `feed.home` | 发现页 |
| `radio` | `radio.modes`、`radio.next` | 电台入口 |
| `comments` | `comments.list` | 评论页签 |
| `podcast` | `podcast.home`、`podcast.get` | 播客入口 |

`features` 是能力内的可选方法,声明了才会显示对应控件:

| 特性 | 方法 | 控件 |
|---|---|---|
| `search.suggest` | `search.suggest` | 输入时的搜索建议 |
| `like` | `track.like`,以及 `library.liked_ids` / `track.liked_state` 至少一个 | 喜欢按钮 |
| `history` | `library.recent` | 「接着上次听」 |
| `playlist.edit` | `playlist.create/rename/delete/add_tracks/remove_tracks` | 新建、重命名、删除、加入/移出歌单 |
| `playlist.subscribe` | `playlist.subscribe` | 收藏他人歌单 |
| `follow` | `artist.follow`、`library.artists` | 关注歌手与「我关注的歌手」 |
| `podcast.subscribe` | `podcast.subscribe`、`library.podcasts` | 订阅播客与「我订阅的播客」 |
| `radio.similar` | `radio.similar` | 「心动模式」类的相似推荐 |
| `radio.dislike` | `radio.dislike` | 电台里的「不喜欢」 |
| `comments.like` | `comments.like` | 评论点赞 |

方法的参数与返回见 [05 号文档](05-capabilities.md)。

## 2.4 权限

```json
"permissions": {
  "network": { "hosts": ["music.163.com"], "local": false },
  "media":   { "hosts": ["*.music.126.net"] }
}
```

| 键 | 含义 |
|---|---|
| `network.hosts` | 插件通过 `http.do` 可以访问的主机。插件的请求带 cookie,可携带会话 |
| `network.local` | `true` 时允许访问本机与局域网地址(自建服务器类来源)。**市场收录需要额外人工评审**,安装时有醒目提示 |
| `media.hosts` | **宿主**代为获取的资源主机:封面、头像、音频流。宿主发这些请求时**不带 cookie**,也不经过插件 |

主机写法:

- `example.com`:仅 HTTPS、仅该主机。
- `*.example.com`:任意子域,不含 `example.com` 自身。不允许单独的 `*`,也不允许 `*.com` 这类过宽的通配。
- `http://example.com`:显式允许明文 HTTP。**未写协议一律只允许 HTTPS。**
- 不允许端口、路径;端口一律是协议默认端口。

权限的完整含义与宿主如何强制执行见 [07 号文档](07-security.md)。**更新时权限扩大(新增主机、`local` 变为 `true`、新增能力)需要用户重新确认**,不会静默生效。

## 2.5 用户设置

对「自建服务器」类来源,服务器地址、账号这类信息只有用户知道,不可能写进 manifest。`settings` 声明一份表单,宿主在插件设置页渲染,并在每次调用的 `ctx.settings` 里带给插件。

```json
"settings": [
  { "key": "server",   "type": "url",      "label": "服务器地址", "required": true, "grant_network": true },
  { "key": "username", "type": "text",     "label": "用户名",     "required": true },
  { "key": "password", "type": "password", "label": "密码",       "required": true },
  { "key": "format",   "type": "choice",   "label": "转码格式",   "default": "mp3",
    "options": [{ "value": "mp3", "label": "MP3" }, { "value": "flac", "label": "FLAC" }] }
]
```

| `type` | 说明 |
|---|---|
| `text` `number` `bool` `choice` | 普通项,存在插件的 kv 里 |
| `url` | 同 `text`,宿主校验为 `http(s)` URL。`grant_network: true` 时,用户填入的主机会被**加入该插件的 `network.hosts`**(并在用户确认后允许局域网地址)。这是唯一一种「运行时扩大网络权限」的方式,因为主机是用户亲手输入的 |
| `password` | 存入钥匙串而不是 kv,**不会出现在 `ctx.settings`**,插件用 `secret.get` 读取 |

登录相关的表单(扫码、短信验证码)不在这里,见 [06 号文档](06-auth.md)。

## 2.6 校验

- 解析失败、未知的 `api`、缺失必填字段、`id` 或 `version` 格式不合法 → 拒绝安装。
- 未知字段**忽略**(前向兼容)。
- `capabilities` 与 wasm 实际实现不一致(声明了未实现的方法)→ `mygo-plugin check` 报错,宿主运行时调用到未实现的方法会得到 `unsupported`。
