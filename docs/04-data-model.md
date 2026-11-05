# 04 · 数据模型

插件返回的所有实体都用这套结构,宿主界面只认它们。字段名 `snake_case`,JSON 编码。

## 4.1 约定

| 约定 | 说明 |
|---|---|
| **ID** | 字符串,1–256 字节,对宿主**不透明**。即使来源的 ID 是数字也要写成字符串 `"123"` |
| **全局引用** | 宿主用 `插件id:类型:ID` 唯一标识实体,如 `netease:track:123`。用于历史、队列、路由。插件不用关心 |
| **类型标签** | 每个实体带 `"type"`:`track` `album` `artist` `playlist` `user` `podcast` `post`。列表里混装多种实体时靠它区分 |
| **时间** | Unix 毫秒,`int64`。时长用 `duration_ms` |
| **计数** | `int64`;未知就省略,**不要用 0 表示未知** |
| **文本** | 纯文本,UTF-8,允许 `\n`。**不要放 HTML 或 Markdown**,宿主不会解析 |
| **可选字段** | 省略即未知。双方都忽略未知字段 |
| **图片** | 见 §4.2 |
| **分页** | 见 §4.3 |
| **`extra`** | 任意 JSON 对象,≤ 2 KiB,宿主原样存储,并在后续调用该实体时**原样带回**。放来源私有的东西(如播客节目 ID)。不要放凭据 |
| **本地化** | 插件按 `ctx.locale` 尽量返回对应语言的文本 |

## 4.2 图片

图片字段是一个 URL 字符串,**可以含占位符 `{size}`**。宿主按显示尺寸(像素边长,方形)替换。没有占位符就原样使用。

```
"https://p1.music.126.net/abc.jpg?param={size}y{size}"   →   ...?param=300y300
```

图片由**宿主**获取(不带 cookie),主机必须在 `media.hosts` 内。只允许 `https`(或被显式允许的 `http`)。

## 4.3 分页

所有可翻页的列表,请求带 `cursor`(首页省略)与可选 `limit`,响应带 `next`:

```json
{ "items": [ ... ], "next": "opaque-token" }
```

`next` 为空或省略表示没有更多。`cursor` 对宿主**不透明**,插件可以在里面编码偏移量或时间戳;宿主绝不解析、不拼接。`limit` 只是建议,插件可以返回更少或更多(≤ 200)。

## 4.4 实体

标 ✓ 的是必填。

### Track(曲目)

```json
{
  "type": "track",
  "id": "186016",
  "title": "晴天",
  "artists": [{ "id": "6452", "name": "周杰伦" }],
  "album": { "id": "18905", "name": "叶惠美", "cover": "https://.../a.jpg?param={size}y{size}" },
  "duration_ms": 269000,
  "playable": true,
  "comment_thread": "R_SO_4_186016"
}
```

| 字段 | | 说明 |
|---|---|---|
| `id` | ✓ | 传给 `stream.resolve` 的标识 |
| `title` | ✓ | |
| `artists` | ✓ | `[{ id?, name }]`,可为空数组(未知艺人) |
| `album` | | `{ id?, name, cover? }` |
| `duration_ms` | | |
| `cover` | | 单曲封面;省略则用 `album.cover` |
| `playable` | | 默认 `true`。`false` 时宿主把它显示为灰色并跳过 |
| `unplayable_reason` | | 一句话原因,如「需要会员」「已下架」 |
| `aliases` | | 副标题,如别名、「Live」「伴奏」 |
| `reason` | | 推荐理由(每日推荐用) |
| `kind` | | `song`(默认)或 `episode` |
| `episode` | | `kind: "episode"` 时:`{ id, podcast: { id, title }, description?, published_ms? }`。节目 ID 与音频 ID 不同,`id` 仍是音频,`episode.id` 是节目 |
| `comment_thread` | | 评论串标识(不透明字符串)。有它且来源声明了 `comments` 才显示评论页签 |
| `url` | | 曲目在来源网站上的页面,用于「复制链接」 |
| `extra` | | 见 §4.1 |

`kind: "episode"` 的曲目照常播放;宿主据此额外提供「本机转写字幕」(见 [09](09-host-behavior.md))。

### Album(专辑)

`type`✓ `id`✓ `title`✓ · `artists[]` · `cover` · `released_ms` · `track_count` · `description` · `subtitle`(如「EP」「单曲」) · `extra`

### Artist(艺人)

`type`✓ `id`✓ `name`✓ · `avatar`(方形) · `aliases[]` · `description` · `followed`(当前用户已关注) · `track_count` · `album_count` · `extra`

### Playlist(歌单)

| 字段 | | 说明 |
|---|---|---|
| `type` `id` `title` | ✓ | |
| `cover` `description` | | |
| `track_count` `play_count` | | |
| `creator` | | `{ id?, name, avatar? }` |
| `update_text` | | 如「每周四更新」 |
| `role` | | `normal`(默认)或 `liked`(用户的「我喜欢的音乐」) |
| `private` | | 只有创建者可见 |
| `owned` | | 当前用户创建的 |
| `subscribed` | | 当前用户已收藏 |
| `preview` | | `[{ title, subtitle }]`,榜单卡片上展示的前几名 |
| `extra` | | |

### User(用户)

`type`✓ `id`✓ `name`✓ · `avatar` · `description` · `followed` · `extra`

### Podcast(播客)

`type`✓ `id`✓ `title`✓ · `cover` · `description` · `category` · `host`(`{ id?, name, avatar? }`) · `episode_count` · `subscriber_count` · `play_count` · `subscribed` · `tagline`(列表里的一句推荐语) · `latest_episode`(最新一期标题) · `extra`

### Post(动态)

用于艺人/用户页的「动态」页签。

`type`✓ `id`✓ · `author`(`{ id?, name, avatar? }`) · `time_ms` · `text` · `images[]` · `attachment`(一个 Track/Album/Artist/Playlist 实体,动态里分享的内容) · `forwarded`(转发的原文,纯文本) · `like_count` `comment_count` `share_count` · `has_unsupported_media`(带有宿主无法播放的视频)

## 4.5 Stream(取流结果)

```json
{
  "url": "https://m701.music.126.net/xxx.flac",
  "headers": { "Referer": "https://music.163.com/" },
  "format": "flac",
  "quality": "lossless",
  "size_bytes": 31457280,
  "gain_db": -3.2,
  "peak": 0.98,
  "expires_ms": 1760000000000
}
```

| 字段 | | 说明 |
|---|---|---|
| `url` | ✓ | 主机必须在 `media.hosts` 内 |
| `format` | ✓ | `mp3` 或 `flac`(v1 宿主只解码这两种) |
| `quality` | ✓ | 实际给出的档位,可能低于请求的 |
| `headers` | | 宿主请求该 URL 时附带。不得含 `Cookie`(宿主会剔除) |
| `size_bytes` | | 已知时填,用于进度与缓存校验 |
| `gain_db` / `peak` | | 响度均衡。`gain_db` 是应调整的分贝数;`peak` 是最大采样(满幅为 1),未知省略 |
| `expires_ms` | | 地址失效时间。过期后宿主会重新 `stream.resolve`,而不是重试旧地址 |

**服务器必须支持 HTTP `Range` 请求**,否则无法拖动进度。不支持的来源暂不在 v1 范围内。

**音质档位**(从低到高,`quality` / `qualities` 使用):

| id | 约定 | 界面名 |
|---|---|---|
| `standard` | ≈ 128 kbps | 标准 |
| `higher` | ≈ 192 kbps | 较高 |
| `exhigh` | ≈ 320 kbps | 极高 |
| `lossless` | 无损(CD 级) | 无损 |
| `hires` | 高解析度无损 | Hi-Res |

用户选择的档位不在插件的 `qualities` 里时,宿主向下取最近的一档。

## 4.6 Lyrics(歌词)

两种形式二选一;都给时以 `lines` 为准。

```json
{
  "lines": [
    { "start_ms": 12000, "end_ms": 16500, "text": "故事的小黄花", "translation": "The little yellow flower of the story",
      "words": [ { "start_ms": 12000, "end_ms": 12400, "text": "故" }, { "start_ms": 12400, "end_ms": 12800, "text": "事" } ] }
  ],
  "instrumental": false
}
```

```json
{ "lrc": "[00:12.00]故事的小黄花\n...", "translation_lrc": "[00:12.00]...", "instrumental": false }
```

- `words` 有就按逐字渲染,没有按逐行。
- 宿主解析标准 LRC(含 `[mm:ss.xx]` 与增强版逐字标记 `<mm:ss.xx>`)。**来源私有的歌词格式(如网易云的 YRC)由插件自己转成 `lines`**,SDK 提供解析助手。
- `instrumental: true` 表示纯音乐,宿主显示「纯音乐」而不是「暂无歌词」。

## 4.7 Comment(评论)

```json
{ "id": "1", "user": { "id": "9", "name": "某人", "avatar": "https://..." }, "text": "好听", "time_ms": 1700000000000,
  "like_count": 120, "liked": false, "location": "上海", "reply_to": { "user_name": "另一个人", "text": "原文" } }
```

`comments.list` 返回 `{ "hot": [Comment], "items": [Comment], "total": 1234, "next": "..." }`。`hot` 只在首页给。

## 4.8 页面结构:分区与页签

插件不画界面,但需要表达「发现页由哪几块组成」「艺人页有哪几个页签」。用两个声明式结构:

### Section(分区)

```json
{
  "id": "daily",
  "title": "每日推荐",
  "subtitle": "根据你的口味,每天 6:00 更新",
  "style": "hero",
  "play_all": true,
  "items": [ { "type": "track", "id": "..." } ],
  "next": "",
  "more": true
}
```

| 字段 | 说明 |
|---|---|
| `id` `title` | ✓ |
| `style` | 展示提示:`list`(曲目列表)、`grid`(卡片网格)、`hero`(大卡片,配「播放全部」)、`chart`(榜单卡,显示 `preview`)。宿主不认识的值按 `list`/`grid` 兜底 |
| `play_all` | 是否显示「播放全部」,仅对全是曲目的分区有效 |
| `items` | 分区里的实体(可混装),首屏数量由插件定,建议 ≤ 30 |
| `more` | 是否有完整页:`true` 时宿主显示「查看全部」,点开调用 `feed.section` |
| `next` | `feed.section` 的分页游标 |

### Tab(页签)

实体详情(艺人、用户)可返回页签列表,每个页签用 `tab.load` 取内容:

```json
{ "id": "hot", "label": "热门歌曲", "type": "tracks" }
```

`type` 是页签内容的实体类型:`tracks` `albums` `artists` `playlists` `podcasts` `users` `posts`。宿主按类型选择列表或卡片布局。页签的**名称与顺序由插件决定**,「动态」之类来源特有的页签因此不需要宿主认识。

## 4.9 登录步骤

登录相关的结构(方式、表单、二维码)单独在 [06 号文档](06-auth.md)。
