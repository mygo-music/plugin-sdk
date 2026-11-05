# 05 · 能力与方法

方法由宿主调用(见 [ABI](03-abi.md) 的请求信封)。下面只写 `params` 与 `result`;错误统一用 ABI 的错误码。

通用规则:

- **读方法没有副作用**,可被宿主重试、缓存。
- 需要登录的方法,在未登录时返回 `unauthenticated`。
- 参数里的 `track` 是宿主保存的完整 Track(含 `extra`),不只是 ID。
- 方法返回前插件必须完成工作;**不要留后台任务**,实例随时会被丢弃。
- 实体类型见 [04](04-data-model.md);`Page` 指 `{ items, next }`。

## 5.1 生命周期

| 方法 | params | result |
|---|---|---|
| `plugin.init` | `{ host, locale, settings }` | `{}` |

## 5.2 `search`

| 方法 | params | result |
|---|---|---|
| `search.query` | `{ query, type, cursor?, limit? }` | `{ items: [Entity], next?, total? }` |
| `search.suggest`(特性) | `{ query }` | `{ groups: [{ type, items: [Entity] }] }` |

- `type` 是 manifest `search_types` 之一;`items` 里的实体类型与之对应(`track`→Track,`album`→Album……)。
- `search.suggest` 的 `groups` 顺序就是展示顺序,`type` 取 `track` `artist` `album` `playlist`。宿主自己保存搜索历史,插件不管。

## 5.3 `stream`

| 方法 | params | result |
|---|---|---|
| `stream.resolve` | `{ track, quality }` | Stream |

- 返回的 `quality` 可低于请求的。完全无法播放 → `unavailable`(附 `message` 说明原因)。
- 流地址的主机必须在 `media.hosts` 内。
- 该调用在**每次开始播放或地址过期时**发生,要快(默认超时 15 秒)。

## 5.4 实体详情:`album` `artist` `playlist` `user`

| 方法 | params | result |
|---|---|---|
| `album.get` | `{ id, cursor? }` | `{ album, tracks: [Track], next? }` |
| `playlist.get` | `{ id, cursor? }` | `{ playlist, tracks: [Track], next? }` |
| `artist.get` | `{ id }` | `{ artist, tabs: [Tab], first?: { tab, items, next? } }` |
| `user.get` | `{ id }` | `{ user, tabs: [Tab], first?: { tab, items, next? } }` |
| `tab.load` | `{ entity: { type, id, extra? }, tab, cursor? }` | `{ items, next? }` |

- `first` 是第一个页签的内容,省去一次往返。省略则宿主随后调用 `tab.load`。
- 声明了 `artist` / `user` 能力,但返回 `tabs: []` 是合法的(只有头部信息)。
- `tab.load` 的 `items` 实体类型必须与 `Tab.type` 一致。
- `entity.extra` 是 `artist.get` / `user.get` 返回的实体里的 `extra`,宿主原样带回。页签需要的来源私有数据(如艺人「动态」所属的账号)放在那里。

## 5.5 `lyrics`

| 方法 | params | result |
|---|---|---|
| `lyrics.get` | `{ track }` | Lyrics |

没有歌词 → `not_found`(宿主显示「暂无歌词」)。

## 5.6 `feed`(发现页)

| 方法 | params | result |
|---|---|---|
| `feed.home` | `{}` | `{ sections: [Section] }` |
| `feed.section` | `{ id, cursor? }` | `{ section: Section }` |

- `feed.home` 返回发现页的全部分区(含「每日推荐」「推荐歌单」「排行榜」之类)。需要登录的分区(如个性化推荐)在未登录时**省略**,不要报错。
- `Section.more = true` 时宿主显示「查看全部」,点开调用 `feed.section`,返回同一个分区的完整内容,可分页。
- 分区之间的失败互不影响:拿不到的分区直接省略。

## 5.7 `library`(依赖 `account`)

| 方法 | params | result |
|---|---|---|
| `library.playlists` | `{}` | `{ owned: [Playlist], saved: [Playlist] }` |

「我喜欢的音乐」是 `role: "liked"` 的歌单,放在 `owned` 里。

特性:

| 特性 | 方法 | params | result |
|---|---|---|---|
| `like` | `library.liked_ids` | `{}` | `{ ids: [string] }` |
| | `track.liked_state` | `{ tracks: [Track] }` | `{ liked: { "<id>": bool } }` |
| | `track.like` | `{ track, liked }` | `{}` |
| `history` | `library.recent` | `{ limit }` | `{ plays: [{ track, played_ms }] }` |
| `follow` | `library.artists` | `{ cursor? }` | `{ items: [Artist], next? }` |
| | `artist.follow` | `{ id, followed }` | `{}` |
| `podcast.subscribe` | `library.podcasts` | `{ cursor? }` | `{ items: [Podcast], next? }` |
| | `podcast.subscribe` | `{ id, subscribed }` | `{}` |
| `playlist.subscribe` | `playlist.subscribe` | `{ id, subscribed }` | `{}` |
| `playlist.edit` | `playlist.create` | `{ title, private? }` | `{ playlist }` |
| | `playlist.rename` | `{ id, title }` | `{}` |
| | `playlist.delete` | `{ id }` | `{}` |
| | `playlist.add_tracks` | `{ id, tracks: [Track] }` | `{}`(曲目已在歌单里 → `conflict`) |
| | `playlist.remove_tracks` | `{ id, tracks: [Track] }` | `{}` |

- `library.liked_ids` 在登录后一次拉取,宿主据此给所有列表里的曲目画「喜欢」状态。曲目很多时它可能很大(≤ 8 MiB 信封内);拿不到全量的来源改用 `track.liked_state`(按页查询),二者实现一个即可,宿主两个都会尝试。
- `track.like` 同样适用于 `kind: "episode"` 的曲目,由插件按曲目的 `episode` 信息决定怎么提交。
- `library.recent` 用于「接着上次听」:宿主在登录后查看最近的播放,提示是否继续。
- 写操作成功返回 `{}`,失败返回错误;宿主在成功后自行更新界面,不再重新拉取。

## 5.8 `radio`(电台)

| 方法 | params | result |
|---|---|---|
| `radio.modes` | `{}` | `{ modes: [{ id, label }] }`,第一个是默认 |
| `radio.next` | `{ mode }` | `{ tracks: [Track] }` |
| `radio.dislike`(特性) | `{ track }` | `{}` |
| `radio.similar`(特性) | `{ seed: Track, origin?: Track, context?: { playlist_id? } }` | `{ tracks: [Track] }` |

- `radio.next` 每次返回下一批(通常 3–10 首);宿主在队列将尽时再调用。**电台的「游标」由插件自己维护**(存 kv 或会话),宿主不传。
- `radio.similar` 是「以这首为种子的相似推荐」(网易云的心动模式)。`seed` 是最近播放的一首,`origin` 是这一轮开始时的那首(省略时同 `seed`),`context.playlist_id` 是推荐所依据的歌单(用户的「我喜欢的音乐」)。

## 5.9 `comments`

| 方法 | params | result |
|---|---|---|
| `comments.list` | `{ thread, cursor?, limit? }` | `{ hot: [Comment], items: [Comment], total?, next? }` |
| `comments.like`(特性) | `{ thread, id, liked }` | `{}` |

`thread` 取自 `Track.comment_thread`。`hot` 只在首页返回。

## 5.10 `podcast`

| 方法 | params | result |
|---|---|---|
| `podcast.home` | `{}` | `{ sections: [Section] }` |
| `podcast.section` | `{ id, cursor? }` | `{ section: Section }` |
| `podcast.get` | `{ id, cursor? }` | `{ podcast, episodes: [Track], next? }` |

- `episodes` 的每一项是 `kind: "episode"` 的 Track。
- `podcast.home` 里可以有「我订阅的」(需登录,未登录时省略)。

## 5.11 `account`

方法与流程见 [06 号文档](06-auth.md)。

## 5.12 写插件时的注意

- **不要依赖实例内存**:两次调用可能落在不同实例(甚至中间实例被重建)。需要保留的东西,改变时立刻写 kv / secret / cookie 罐。
- **留意 `ctx.deadline_ms`**:剩余时间不够时尽早返回 `network`,别让宿主等到超时再杀实例。
- **失败要局部化**:发现页的一个分区拿不到,就省略它,而不是整个 `feed.home` 报错。
- **返回的实体越完整越好,但别拖慢首屏**:列表只给列表需要的字段,详情页再补全。
