# 03 · ABI

宿主与插件之间只有一条窄通道:**一个导出函数、一个导入函数,两边都传 JSON。** 新增方法或字段不改变 ABI,只在 JSON 里加。

> 这份文档规定的是 wasm 层。写 Go 插件的人用 `plugin-sdk/go`,看不到这些细节;写其他语言插件或实现运行时的人才需要读。

## 3.1 模块要求

- **WASI preview 1**(`wasip1`)模块,**reactor** 形态:没有 `_start`,有 `_initialize`。
- Go:`GOOS=wasip1 GOARCH=wasm go build -buildmode=c-shared`(需要 Go ≥ 1.24 的 `//go:wasmexport`)。
- 宿主只提供 WASI 的**时钟与随机数**。没有文件系统预打开目录、没有环境变量、没有命令行参数、没有套接字(`sock_*` 一律返回 `ENOSYS`)。
- 模块只能导入 `wasi_snapshot_preview1` 与 `mygo`,导入其他模块 → 拒绝加载。
- 内存上限 256 MiB(4096 页),超限时该次调用失败并丢弃实例。

## 3.2 导出(插件 → 宿主可见)

| 导出 | 签名 | 说明 |
|---|---|---|
| `_initialize` | `() → ()` | WASI reactor 入口。宿主实例化后**先调用一次**,再调用其他导出 |
| `mygo_abi` | `() → i32` | 返回插件构建时的 `api`,必须与 manifest 一致,否则拒绝加载 |
| `mygo_alloc` | `(size: i32) → i32` | 在插件内存里分配 `size` 字节,返回指针;失败返回 0 |
| `mygo_free` | `(ptr: i32) → ()` | 释放 `mygo_alloc` 或 `mygo_call` 返回的缓冲区 |
| `mygo_call` | `(ptr: i32, len: i32) → i64` | 处理一次方法调用,见 §3.3 |

## 3.3 调用约定:宿主 → 插件

1. 宿主调用 `mygo_alloc(n)`,把请求 JSON 写进返回的缓冲区。
2. 宿主调用 `mygo_call(ptr, n)`。**请求缓冲区的所有权转给插件**,插件解析完负责 `free`。
3. 插件返回一个 `i64`:高 32 位是响应缓冲区的指针,低 32 位是长度。失败返回 0。
4. 宿主读出响应,然后调用 `mygo_free(ptr)`。

**请求信封:**

```json
{
  "v": 1,
  "method": "search.query",
  "ctx": { "locale": "zh-CN", "settings": {}, "deadline_ms": 1760000000000 },
  "params": { "query": "周杰伦", "type": "track", "limit": 30 }
}
```

**响应信封**(二选一):

```json
{ "result": { "items": [], "next": "" } }
```
```json
{ "error": { "code": "unauthenticated", "message": "需要登录", "retry_after_ms": 0 } }
```

- `ctx.locale`:界面语言,BCP 47。插件据此返回对应语言的文本(如榜单名)。
- `ctx.settings`:用户设置(manifest 的 `settings`,不含 `password`)。
- `ctx.deadline_ms`:本次调用的绝对截止时间(Unix 毫秒)。过了它宿主会丢弃实例。
- 一次 `mygo_call` 只处理一个方法。**宿主保证同一实例不会并发调用**,也不会在一次调用未返回时再次调用(无重入)。

### 错误码

| `code` | 含义 | 宿主的反应 |
|---|---|---|
| `unauthenticated` | 需要登录或会话失效 | 提示登录;失效时清除登录状态 |
| `unavailable` | 内容不可用(下架、无版权、需会员) | 该曲目显示为不可播放,自动跳到下一首 |
| `not_found` | 实体不存在 | 显示「找不到」 |
| `rate_limited` | 被限流,带 `retry_after_ms` | 退避重试一次,仍失败则提示 |
| `network` | 网络错误或对端 5xx | 显示可重试的错误 |
| `invalid_argument` | 参数不合法 | 视为宿主或插件的 bug,记日志 |
| `unsupported` | 未实现该方法 | 宿主不应调用未声明的方法;收到则隐藏相应入口 |
| `internal` | 其他内部错误 | 显示通用错误,记日志 |

`message` 是给用户看的一句话(可本地化),不要放敏感信息。

## 3.4 调用约定:插件 → 宿主

导入函数,模块名 `mygo`:

```
mygo.call(req_ptr: i32, req_len: i32, out_ptr: i32, out_cap: i32) → i32
```

- `req_ptr/req_len`:插件内存里的请求 JSON,**所有权仍属于插件**,宿主在调用期间复制。
- `out_ptr/out_cap`:插件预先分配的输出缓冲区。
- 返回值 ≥ 0:响应长度,已写入 `out_ptr`。
- 返回值 < 0:响应比缓冲区大,所需长度为 `-返回值`。宿主**暂存**这份响应;插件分配足够大的缓冲区后调用 **`mygo.take(out_ptr, out_cap) → i32`** 取走(取走后暂存清空)。

这样宿主**不需要在插件阻塞于导入调用时回调插件的导出函数**(无重入),ABI 因此不依赖任何语言运行时对重入的处理。代价是大响应多一次 `take` 调用。时间、随机数用 WASI 自带的 `clock_time_get` 与 `random_get`,不另设宿主函数。

> 待实测 S1(见[总览](01-overview.md)):Go 的 wasip1 reactor 在「导出函数执行中阻塞于导入函数」时的行为。上面的设计已经避开重入,但这一点本身仍要在 wazero 下验证。

请求与响应用同一种信封,只是 `method` 是宿主函数名:

```json
{ "method": "http.do", "params": { "url": "https://music.163.com/..." } }
```

## 3.5 宿主函数

所有函数都受 §3.7 的资源限制。凡是涉及网络的,都按 manifest 的权限强制执行(见 [07](07-security.md))。

### `http.do`

发起一次 HTTP 请求。

```json
{
  "method": "POST",
  "url": "https://music.163.com/eapi/...",
  "headers": { "Content-Type": "application/x-www-form-urlencoded" },
  "body": "<base64>",
  "timeout_ms": 15000,
  "redirect": "follow",
  "cookies": "jar"
}
```

| 字段 | 说明 |
|---|---|
| `method` | `GET` `POST` `PUT` `DELETE` `HEAD` |
| `url` | 必须是 `https`,或 manifest 显式允许的 `http`;主机必须在 `network.hosts` 内 |
| `headers` | 宿主会剔除并重置:`Host`、`Content-Length`、`Connection`、`Cookie`(在 `cookies: "jar"` 时由宿主按 cookie 罐填)、`Proxy-*` |
| `body` | base64,可省略 |
| `timeout_ms` | 默认 15000,最大 60000 |
| `redirect` | `follow`(默认,最多 5 跳,**每一跳重新校验主机与 IP**)或 `manual` |
| `cookies` | `jar`(默认):按该插件的 cookie 罐读写 `Cookie` / `Set-Cookie`;`none`:不带也不存 |

响应:

```json
{ "status": 200, "headers": { "content-type": ["application/json"] }, "body": "<base64>", "url": "最终地址" }
```

响应体 ≤ 16 MiB,超出则返回 `network` 错误。HTTP 状态码(含 4xx/5xx)是**正常响应**,由插件自行判断;传输失败才返回错误。

### `cookie.list` / `cookie.set` / `cookie.clear`

读写**该插件自己的** cookie 罐,只能操作 `network.hosts` 覆盖的域。

```json
{ "method": "cookie.list", "params": { "url": "https://music.163.com/" } }
→ { "cookies": [{ "name": "MUSIC_U", "value": "...", "domain": "music.163.com", "path": "/", "expires_ms": 0, "secure": true, "http_only": true }] }
```

cookie 罐持久化在钥匙串(按插件隔离,见 [07](07-security.md)),重启后恢复。`cookie.clear`(不带参数)清空整个罐,用于退出登录。

### `kv.get` / `kv.set` / `kv.delete` / `kv.list`

插件私有的键值存储,存在用户数据目录。值是 JSON。总量 ≤ 1 MiB。**用于设备标识、缓存、偏好,不用于凭据。**

```json
{ "method": "kv.set", "params": { "key": "device_id", "value": "abc" } }
{ "method": "kv.get", "params": { "key": "device_id" } } → { "value": "abc" }   // 不存在时 value 为 null
```

### `secret.get` / `secret.set` / `secret.delete`

存进系统钥匙串,按插件隔离。值是字符串,单个 ≤ 4 KiB,最多 16 个键。**凭据、令牌放这里。**

### `log`

```json
{ "method": "log", "params": { "level": "info", "msg": "resolved", "fields": { "id": "123" } } }
```

`level`:`debug` `info` `warn` `error`。宿主限流(每秒 50 条),进入应用日志与开发控制台;**绝不会**被上传。不要往日志写令牌。

### `sleep`

```json
{ "method": "sleep", "params": { "ms": 500 } }
```

阻塞指定毫秒(≤ 10000)。使用它而不是忙等,宿主会计入 `deadline_ms`。

## 3.6 生命周期

```
安装 → 编译(缓存到磁盘)→ [首次使用] 实例化 → _initialize → mygo_abi → plugin.init → 服务调用 …
                                                                                  └→ 超时 / 超内存 / 空闲 5 分钟 → 丢弃实例
```

- **`plugin.init`**:每个新实例的第一个调用。参数:

  ```json
  { "host": { "name": "MyGo Music", "version": "0.2.0", "api": [1], "os": "windows", "arch": "amd64", "features": [] },
    "locale": "zh-CN", "settings": {} }
  ```

  返回 `{}`。插件在这里从 cookie 罐、kv、secret 恢复自己需要的内存状态。失败 → 实例丢弃,调用方得到 `internal`。

- **实例池。** 宿主为每个插件维护最多 4 个实例,每个实例一次一个调用。排满时请求排队。宿主根据负载自行扩缩,**插件不能假设两次调用落在同一个实例上**。
- **没有 `plugin.shutdown`。** 实例随时可能被丢弃,所以任何需要持久化的东西必须在改变时立刻写出(cookie 罐是自动持久化的)。
- **取消与超时。** 用户离开页面时,宿主可以取消一次进行中的调用,做法是关闭该实例(wazero 的 `WithCloseOnContextDone`)。这是**设计内的**行为,因此必须保证中途丢弃实例不会留下半写状态——凡是多步写入,先写 secret/kv 的新值,再切换。

## 3.7 资源限制

| 项 | 限制 |
|---|---|
| 单次调用默认超时 | 20 s(`stream.resolve` 15 s,`auth.poll` 10 s) |
| 内存 | 256 MiB |
| 请求 / 响应 JSON | 各 ≤ 8 MiB |
| `http.do` 响应体 | ≤ 16 MiB;并发 ≤ 6 |
| kv 总量 | 1 MiB |
| secret | 单个 4 KiB,最多 16 个键 |
| 日志 | 50 条/秒 |
| wasm 文件 | ≤ 32 MiB |

## 3.8 编译缓存

wasm 首次编译较慢(体积大的 Go 模块可能要数秒)。宿主在安装时就编译,结果缓存到磁盘(wazero `CompilationCache`),缓存键包含 wasm 的 sha256 与宿主的编译器版本。之后实例化是毫秒级。

## 3.9 其他语言

ABI 与语言无关:能产出满足 §3.1 的 wasm 模块即可。官方只维护 Go SDK;其他语言的 SDK 欢迎社区贡献,`mygo-plugin check` 与同一套一致性测试对它们同样适用。
