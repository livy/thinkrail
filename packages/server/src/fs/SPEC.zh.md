# fs —— 工作树文件读取

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

读取工作区工作树内的目录与文件，路径受包含校验，并判定**一个资源的字节是什么** —— 宿主内每次内容
读取所共享的那一处字节级分类。

## 边界（Boundary）

- **拥有：** `readDir`/`readFile(workspaceId, path)` —— 每个路径都解析并包含到工作树根；`.git` 被隐藏；
  目录排在前面。`readDir` 在每个操作系统上都报告以 `/` 为分隔符的工作树相对路径 —— 与 git status 和
  watch 事件所用的同一形式 —— 好让浏览器能把一个列出的节点匹配到一个变更路径（Windows 的
  `relative()` 否则会产出 `src\auth`）。（`.thinkrail/` **不**被隐藏 —— 它像任何其他目录一样展示，
  因此未来宿主管理的内容在那里保持可见；它那短暂的 `context/` 被挡在 git 之外，而不是挡在树之外。）
  `readFile` 回答 **`{ content, meta }`**：文件按字节读取，且只在它是文本时才解码，因此一个仅字节的
  资源其 `content` 是 `""`，客户端改从宿主的 `/files` 路由获取那些字节，而不是收到乱码。
  **`resolveWorktreeFile(workspaceId, path)`** 返回被包含的绝对路径，供宿主经 HTTP 流式发送一个文件
  的原始字节（`/files/…` 路由服务 markdown 查看器中的相对图片，`/blob` 路由的包含检查），也供
  `changes` 模块做原子写入。词法逃逸、`.git`，以及一个已存在且其解析目标离开工作树的符号链接链都会被
  拒绝；缺失的叶子路径被允许，因此一个已删除的文件可以被恢复。`changes` 传 `{ followLeaf: false }`，
  因此包含校验经由父目录，而它自己的 `lstat` 能拒绝一个逃逸的叶子符号链接而不跟随它。本模块拥有路径
  安全；它的调用方拥有流式发送与写入。**`workspaceRelativePath(root, path)`** 是从原生或绝对路径到
  线上工作树相对 POSIX 形式的唯一转换（architecture Decision 22）：对根解析，在外为 `null`（诸如
  `..env` 的名字是在内），根自身为 `""`。`reviews` 经它规范化锚点。它保持宿主语义，因此在 POSIX 上
  反斜杠仍是文件名的一部分；持有模型写入路径的调用方自行把 `\` 变成 `/`。
  **内容分类**（`content.ts`，纯的，无依赖）：`CONTENT_SNIFF_BYTES` 点名共享的 8 KiB 有界头部大小；
  `classifyBytes(bytes)` → `{ text, mime? }` —— **mime** 是*字节*所证明的东西（png/jpeg/gif/webp/avif/
  bmp/ico/pdf/zip/gzip/woff/woff2 的魔数，加上对根元素为 `<svg>` 的文本的 `image/svg+xml`，无论直接是
  还是藏在一个 XML 序言之后 —— 仅有序言不构成图像；以及 `application/vnd.git-lfs`，对至多 1 KiB 且
  恰好是 `git lfs` 自己写出的三行指针的文本文件（依次是 `version`、`oid sha256:`、`size`，别无其他），
  因此任何仅仅引用了指针的散文都不会被误认成指针 —— 所以一个内容是指针的 `.png` 被报告为它本来的
  指针，而不是一张解码失败的图像）；**text** 是「无被识别的二进制魔数、前 8 KiB 内无 NUL 字节，且严格
  UTF-8 解码通过」（BOM 是文本；SVG 是文本，因为 `image/svg+xml` 是从文本推断出来的，永不从魔数推断）。
  魔数胜过可解码性：一个未压缩的 PDF 是合法 ASCII，然而它是 PDF 渲染器拥有的文档，而不是代码渲染器能
  认领的源码；`hashBytes(bytes)` → sha-256 十六进制，它**就是**一个资源在线上契约中的身份
  （`ResourceMeta.hash`、`ReviewAnchor.contentHash`、`change.*` 的 compare-and-swap）；
  `decodeText(bytes)` → 那唯一一次 UTF-8 解码（保留 BOM，因为 BOM 是哈希所覆盖字节的一部分）；
  `mimeFromPath(path)` 拥有任何字节检查都给不出的文件名回退（markdown/json/csv/yaml/html/xhtml/plain）；
  `resourceMeta(bytes, path)` → 线上的 **`ResourceMeta`**，加上那个回退，并把 `null` 字节当作缺失
  （`hash`/`byteLength` 为 null，`text: true` —— 没有东西需要解码，而与它配对的空字符串是合法文本）。
- **公开接口（barrel）：** `readDir`、`readFile`、`resolveWorktreeFile`、`workspaceRelativePath`、
  `classifyBytes`、`CONTENT_SNIFF_BYTES`、`mimeFromPath`、`hashBytes`、`decodeText`、`resourceMeta`。
- **允许的依赖：** `persistence`（工作区查找）；`contracts`（`FileNode`、`ResourceMeta`）；Node 的
  `fs`/`path`/`crypto`。
- **禁止：** `host`；兄弟特性。

## 必须做对（Get right）

- **一处分类，一个哈希。** 一个资源的文本性、媒体类型与 sha-256 在此决定，别无他处：`reviews` 用它们
  捕获锚点，`git` 用它们给 diff 两侧盖章，`changes` 用它们守卫自己的写入。两份实现会让一条评论的
  `contentHash` 与一次回退的 `expect` 哈希对同一批字节意见不一。
- **扩展名永不胜过字节。** 一个内容是 PNG 的 `.md` 文件报告 `image/png`；文件名只在字节什么都没说时
  才被咨询，因此一个标错的资源无法说服渲染器把它当文本解析。
