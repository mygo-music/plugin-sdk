# 06 · 登录

登录的**界面由宿主画**,插件只回答「现在是哪一步、需要什么」。这样所有来源的登录体验一致,插件也无法伪造登录框来钓取密码——密码框总是宿主的原生控件。

`account` 能力的全部方法:

| 方法 | params | result |
|---|---|---|
| `auth.status` | `{}` | `{ logged_in, user? }` |
| `auth.methods` | `{}` | `{ methods: [{ id, label }] }` |
| `auth.begin` | `{ method }` | Step |
| `auth.poll` | `{ token }` | Step |
| `auth.submit` | `{ token, action?, values }` | Step |
| `auth.cancel` | `{ token }` | `{}` |
| `auth.logout` | `{}` | `{}` |

`user` 是 `{ id?, name, avatar? }`。

## 6.1 状态机

```
        auth.begin(method)
              │
              ▼
   ┌────────  Step  ◀──────────────────┐
   │   qr / form / done                │
   │                                   │
   │ qr:   宿主按 poll_ms 调 auth.poll ─┤ (waiting / scanned 仍是 qr 步)
   │ form: 用户提交 → auth.submit ──────┘ (返回下一步,或带 error 的同一步)
   ▼
  done  → 会话已由插件写入 cookie 罐 / secret;宿主调 auth.status 取用户信息
```

`token` 由插件在 `auth.begin` 里生成,标识这次登录尝试。**因为实例无状态,token 对应的中间状态(如二维码 key)要存在 kv 里,或编码进 token 本身**,不能放在内存。

## 6.2 Step

三种 `type`:

### `qr`

```json
{
  "type": "qr",
  "token": "t1",
  "content": "https://music.163.com/login?codekey=...",
  "state": "waiting",
  "hint": "请使用网易云音乐 App 扫码",
  "poll_ms": 2000,
  "expires_ms": 1760000120000
}
```

| 字段 | 说明 |
|---|---|
| `content` | 二维码内容,宿主负责渲染。必须是 `https` URL(或来源 App 认得的文本) |
| `state` | `waiting`(等待扫码)、`scanned`(已扫码,等待在手机上确认)、`expired`(已过期) |
| `hint` | 展示给用户的一句话;`scanned` 时可以放「请在手机上确认」 |
| `user` | `scanned` 时可选:`{ name, avatar? }`,宿主显示「xxx 已扫码」 |
| `poll_ms` | 宿主调 `auth.poll` 的间隔(≥ 1000) |
| `expires_ms` | 过期时间,宿主据此倒计时 |

`expired` 时宿主显示「二维码已过期」和「刷新」按钮,点击后重新 `auth.begin`。

### `form`

```json
{
  "type": "form",
  "token": "t2",
  "title": "手机号登录",
  "message": "验证码已发送到 138****0000",
  "error": "验证码不正确",
  "fields": [
    { "key": "phone", "type": "phone", "label": "手机号", "required": true },
    { "key": "code",  "type": "code",  "label": "验证码", "required": true }
  ],
  "actions": [ { "id": "send_code", "label": "发送验证码" } ],
  "submit_label": "登录"
}
```

| 字段类型 | 控件 |
|---|---|
| `text` | 单行文本 |
| `password` | 密码框(掩码,不进输入法联想) |
| `phone` | 手机号 |
| `code` | 数字验证码 |
| `select` | 下拉,带 `options: [{ value, label }]` |

- 用户点「登录」→ `auth.submit { token, values }`;点 `actions` 里的按钮 → `auth.submit { token, action: "send_code", values }`(此时 `values` 里已填的内容一并带上)。
- 返回的 Step 可以是**同一个表单**,带 `message`(提示)或 `error`(错误)。密码错误不是协议错误,不要用 `error` 错误码,返回带 `error` 文案的表单步骤。
- `values` 里 `password` 字段的值**只在这一次调用中出现**,宿主不保存。

### `done`

```json
{ "type": "done", "user": { "id": "1", "name": "某人", "avatar": "https://..." } }
```

登录完成。**会话已经被插件保存好**(cookie 罐或 secret),宿主随后调用 `auth.status` 做一次确认,并刷新「我的」「发现」等需要登录的界面。

## 6.3 会话存哪里

| 内容 | 位置 |
|---|---|
| Web 类会话(登录 cookie) | **cookie 罐**:`http.do` 默认读写,自动持久化到钥匙串。网易云的 `MUSIC_U` 就在这里 |
| 令牌、API key | `secret.set` |
| 设备标识、非敏感状态 | `kv.set` |

重启后,宿主从钥匙串恢复 cookie 罐与 secret;插件在 `plugin.init` 里读取自己需要的内存状态。

## 6.4 `auth.status`

宿主在**应用启动后、插件首次使用时**调用,之后在任何方法返回 `unauthenticated` 时再调用一次。

- 允许直接依据本地会话回答(省网络),但若会话明显过期应返回 `logged_in: false`。
- 返回 `logged_in: false` 时,宿主隐藏需要登录的入口(「我的歌单」等),并显示「登录」。

## 6.5 `auth.logout`

插件撤销服务端会话(能做就做,失败也不阻塞),并调用 `cookie.clear` 与自己的 `secret.delete`。宿主随后也会清空该插件的 cookie 罐作为兜底。

## 6.6 未登录时播放

插件可以对未登录用户提供受限服务(例如只能试听或低音质)。`stream.resolve` 在需要登录时返回 `unauthenticated`,宿主弹出登录页,**记住这首歌**,登录成功后接着播放。(这是现有「登录后接着播」的行为,由宿主实现,插件无需配合。)

## 6.7 多账号

v1 每个来源**只有一个账号**。切换账号 = 退出再登录。多账号留给以后(`api` 内可加:`auth.status` 返回账号列表、调用的 `ctx` 带 `account`)。
