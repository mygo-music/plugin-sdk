# 08 · 打包、SDK 与发布

## 8.1 `.mgp` 包

`.mgp`(MyGo Plugin)是一个 zip,**只有根目录的这几个文件**:

| 文件 | | 说明 |
|---|---|---|
| `manifest.json` | ✓ | 见 [02](02-manifest.md) |
| `plugin.wasm` | ✓ | 路径以 manifest 的 `wasm` 为准 |
| `icon.png` | ✓ | 路径以 manifest 的 `icon` 为准;256×256 |
| `LICENSE` | 推荐 | 插件自己的许可证文本 |
| `README.md` | | 展示在插件详情页 |

安装时拒绝:绝对路径、`..`、符号链接、子目录、未列出的文件、解压后总大小 > 48 MiB 的包。

文件名 `<id>-<version>.mgp`。发布到 GitHub Release 时,**Git 标签必须是 `v` + manifest 的 `version`**。构建建议可复现(`-trimpath`、固定 zip 时间戳),但 v1 不强制。

## 8.2 安装来源

| 来源 | 流程 | 校验 |
|---|---|---|
| **市场** | 读 `index.json`,下载该版本的 `url`,校验 sha256 | sha256 固定 + 来源证明 + 可吊销 |
| **URL** | 接受 `https://github.com/<owner>/<repo>`(取最新非预发布 Release 里的 `*.mgp`)或直接的 `.mgp` 地址 | 仅结构校验,强风险提示;若带来源证明则标注 |
| **本地文件** | 拖进窗口或「从文件安装」 | 同 URL |
| **开发目录** | 环境变量 `MYGO_PLUGIN_DEV=<dir>[;<dir>]`,每个目录是一个未打包的插件 | 不校验 sha256;wasm 改动自动重载;界面上标注「开发中」 |

无论来源,**所有插件都在同一个沙箱里,受同样的权限约束**。

## 8.3 构建

```bash
GOOS=wasip1 GOARCH=wasm go build -trimpath -buildmode=c-shared -o plugin.wasm .
```

`mygo-plugin build` 包装了这条命令,并完成打包:读 `manifest.json`,构建 wasm,校验,输出 `dist/<id>-<version>.mgp`。

体积提示:带 `net/http` 的 Go wasm 体积可观(数 MB 到十几 MB),上限 32 MiB。SDK 的 `plugin/http` 提供了不依赖 `net/http` 的轻量客户端;同时提供一个 `http.RoundTripper`,让习惯标准库的人也能用。(见总览的待实测 S2。)

## 8.4 Go SDK 草图

> 以下是**目标体验**的草图,不是已有的 API。签名以实现为准。

```go
package main

import (
	"github.com/mygo-music/plugin-sdk/plugin"
)

// Source 实现了哪些接口,就拥有哪些能力。
type Source struct{ plugin.Base }

func (s *Source) Search(ctx *plugin.Context, q plugin.SearchQuery) (*plugin.SearchResult, error) {
	var out struct{ Songs []song }
	// ctx.HTTP 是 *http.Client,经由宿主的 http.do,受权限与 cookie 罐管理。
	if err := getJSON(ctx.HTTP(), "https://music.163.com/api/search?keywords="+url.QueryEscape(q.Query), &out); err != nil {
		return nil, plugin.Network(err)
	}
	return &plugin.SearchResult{Items: toTracks(out.Songs)}, nil
}

func (s *Source) Resolve(ctx *plugin.Context, t *plugin.Track, quality plugin.Quality) (*plugin.Stream, error) {
	u, err := s.songURL(ctx, t.ID, quality)
	if err != nil {
		return nil, plugin.Unavailable("需要会员")
	}
	return &plugin.Stream{URL: u, Format: plugin.FLAC, Quality: plugin.Lossless}, nil
}

func init() { plugin.Register(&Source{}) } // wasip1 reactor:main 不会被运行
func main() {}
```

要点:

- **能力靠接口探测**:`plugin.Searcher`、`plugin.Streamer`、`plugin.Authenticator`、`plugin.Feeder`……`plugin.Base` 对所有方法返回 `unsupported`,只重写你需要的。
- **`plugin.Context`** 提供:`HTTP()`、`Cookies()`、`KV()`、`Secrets()`、`Log()`、`Locale()`、`Settings()`、`Deadline()`。
- **错误**:`plugin.Unavailable(msg)`、`plugin.Unauthenticated()`、`plugin.NotFound()`、`plugin.RateLimited(after)`、`plugin.Network(err)`。普通 `error` 会被当作 `internal`。
- **助手**:`plugin/lyrics` 提供 LRC 与常见逐字格式的解析,输出 `lines`。
- 协议层(信封、`mygo_call`、内存管理)全部由 SDK 隐藏。

## 8.5 命令行 `mygo-plugin`

| 命令 | 作用 |
|---|---|
| `init <id>` | 用模板创建新插件:目录结构、`manifest.json`、示例、测试、GitHub Actions |
| `build` | 构建 wasm 并打包 `.mgp` |
| `check [目录\|.mgp]` | 一致性检查,见下 |
| `test` | 跑插件自己的测试(用 HTTP 录制回放,不联网) |
| `dev` | 监视源码,重建并通知正在运行的应用重载(配合 `MYGO_PLUGIN_DEV`) |

`check` 做的事,**登记处的 CI 跑的是同一个**:

1. manifest 通过 JSON Schema,且与包内文件一致(图标尺寸、wasm 大小)。
2. wasm 只导入 `wasi_snapshot_preview1` 与 `mygo`;导出齐全;`mygo_abi` 与 manifest 的 `api` 一致。
3. 在参考运行时里实例化,调用 `plugin.init`。
4. 对声明的每个能力和特性,调用其必选方法(HTTP 返回固定的空响应),**期望得到合法的信封**——允许 `network` 之类的错误,不允许 `unsupported`、崩溃、非法 JSON。
5. 对**未**声明的能力,方法应返回 `unsupported`。
6. 资源:体积、`plugin.init` 的耗时。

## 8.6 测试

两层:

- **原生单元测试**(快):`plugin/plugintest` 提供一个假的宿主上下文,直接在本机 Go 里调用你的 `Source`,HTTP 用 `testdata/` 里的录制回放。适合 99% 的逻辑。
- **wasm 一致性**(`mygo-plugin check`):验证打出来的 wasm 真能被运行时加载、符合 ABI。

`plugintest` 同时用于 `plugin-netease`,把现有的 `netease` 测试迁过来。

## 8.7 发布

模板自带的工作流,在推送 `v*` 标签时:

```
test → build → check → pack → attest-build-provenance → 创建 GitHub Release 并上传 <id>-<version>.mgp
```

之后:

- 想进市场 → 去 [`mygo-music/plugins`](https://github.com/mygo-music/plugins) 提交一个 PR(只加一个文件),见那边的 README。
- 已在市场的插件 → 登记处的定时任务会发现新版本,自动更新索引(权限扩大时转人工评审)。
- 不进市场 → 把仓库地址发给用户,用「从 URL 安装」。
