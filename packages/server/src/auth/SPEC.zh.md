# auth —— provider 状态 + 应用内登录

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

关于**模型 provider 凭据**的一切：Welcome 条渲染的读侧（`provider.status`），以及在应用内配置它们的
写侧 —— OAuth 登录、交互式 API key 输入（两者走**同一个登录通道**，issue #97）和登出。全部经由共享
的 `ModelRuntime`（pi 的模型/认证门面）；本模块从不自行解析 `auth.json` / `models.json`，也从不让任何
凭据值出现在线上。

## 边界（Boundary）

- **拥有：**
  - `providerStatus` — `getProviderStatus()` → 线上的 `ProviderStatusReport`：逐 provider 的
    `configured`（pi 的 `hasAuth` 一族事实，因此环境变量认证也算）+ 认证 `kind`（oauth / api-key / env
    / other）+ 显示名 + 应用内登录能力标志 **`canOAuth`/`canApiKey`**，已配置者优先。它**在每次读取时
    经由 `agent` 的当前 runtime 门面重新校验**：本地配置/认证可用性在关闭网络的情况下刷新，因此一次
    外部 PI 登录会变得可见。Central 不由模型 URL 推断：状态把 `shared/jbcentral` 的可执行文件/版本/
    产物后置条件与封闭的 auth/proxy 观测，同同步器最新的期望/已应用代次结合起来。Watcher 漂移会安排
    一次重建；状态在新的候选应用之前是 `configuring`，无法应用时是 `load-failed`。跨模块的有界退出
    义务与 drain 已知的饥饿风险由 [[central-integration]]（不变量）拥有。

    **auth/proxy 观测是缓存的、在读路径之外刷新，且从不轮询。** 一次已结算的 `supported` 读数立即
    返回缓存结果，并在超过 `JBCENTRAL_STATUS_TTL_MS` 之后启动一次后台 `central status` 探测；任一
    判定发生变化时它发布通常的 provider 失效通知，因此一张打开的卡片无需任何客户端定时器即可收敛。
    只有被正面观测到的否定项才设置线上标志：`signed-out` 设置 `signedOut`，被停止的 proxy 仅在已
    配置状态下设置 `proxyStopped`；unknown 从不设置。一次被拒绝的 `central add pi`、一次已启动的
    `central login` 以及一次 Start proxy 尝试都会丢弃共享缓存，因为每一者都可能让观测变陈旧。TTL
    窗口内的带外变更被刻意以陈旧值服务，直到越过该窗口的下一次读取。探测从不在动作进行中或重建
    未完成时运行，因此它无法延迟一次 Connect 或一次候选切换。

    组装是一个纯的 `buildProviderReport(sources)`，作用于一个很窄的 sources 切片，用夹具数据做单元
    测试；kind/detail 映射本身是 `agent` 的 `describeProviderAuth`，与目录的逐模型 `WireModel.auth`
    投影共享，因此 provider 行与选择器行永不矛盾。它的 runtime 读取被限制在该代次的 provider id 允许
    清单内，该清单在不透明的 Central 扩展加载之前（在不变量的宿主注册之后）捕获：Central *引入*的
    provider 永不成为行，也不上线。Central *替换*的内建 provider（真实产物中的 `anthropic`、
    `openai`、`google-vertex`）仍是一行 —— 用户必须能看见哪些 provider 是经 JetBrains AI 到达他们
    的 —— 并报告为 `kind: "central"` 且没有 `detail`，纯由该 id 是否属于该代次的 `opaqueProviderIds`
    决定；它的 `name` 是该代次扩展之前的显示名（pi 组合 `extension.name ?? base.name`，因此一次
    实时读取会把一个由产物选定的字符串交给客户端），而 `configured` 与 `canLogout` 遵循通常读取，
    Central 的 `baseUrl`/`apiKey` 从不被读取。
    - **支持 OAuth 的 id 是一等行。** id 全集取并集：模型目录的 provider、已存凭据的 provider
      （`listCredentials()`），**以及** `Provider.auth.oauth` 存在的 provider —— 一个 OAuth id 可以
      不同于任何模型 provider id（`openai-codex` ≠ `openai`），而已存凭据可以比它的模型活得更久。
      `canOAuth` = 该行的 provider 携带 OAuth 认证（因此 `provider.loginStart(row.id)` 用的是 pi 实际
      会存下的那个凭据 id）；它的行名优先取 `auth.oauth.name`（对 oauth-only 行更具体）。`canApiKey` =
      **`Provider.auth.apiKey.login` 存在** —— pi 公开的 api-key 登录事实，*别无其他*（issue #97：
      交互式登录通道会把 provider 提出的每个提示都停放起来，因此多云提示凭据 —— bedrock/vertex/azure
      —— 以及 OAuth+key 的 provider —— github-copilot —— 直接就可用；手工维护的排除集已经消失；
      `openai-codex` 报告 `false` 是因为 pi 的 provider 没有 key 认证，而不是因为本模块这么说）。
      `canLogout` = 该 id 有一份已存的 **auth.json** 凭据（`credentialProviders`）—— 宿主唯一能移除
      的认证；env / models.json 键控的认证报告 `false`（Sign-out 会变成空操作，因此条带隐藏它）。
      Central 自身的生命周期只是专用的封闭 `jbcentral` 状态，永不是一行；被替换的内建 provider 上的
      `central` 种类是一个身份事实，永不从模型 URL 或配置推断。
  - `detectAccounts` — `provider.detect` 的入驻提示：哪些**其他本地工具**已经登录到某个 pi provider
    也能登录的订阅，外加一个套餐标签（当该工具记录了它时）。各来源及其已登录谓词（每个都对工具自身
    代码验证过；重定位类环境变量被尊重）：
    Claude Code → `anthropic`（`CLAUDE_CODE_OAUTH_TOKEN`；否则是带 refresh token 和 `user:inference`
    scope 的 `.credentials.json` `claudeAiOauth`；再否则，对 macOS Keychain 登录，是全局配置里带订阅
    `billingType` 的 `oauthAccount` —— 仅一个 `oauthAccount` 也会由 Console API-key 登录写入，而
    `/logout` 会移除它；一个过期或被吊销、从未登出的 Keychain token 仍读作已登录，这是唯一被接受的
    误报，因为 Keychain 本身不可触碰）；Codex → `openai`，回退到旧版 `openai-codex`（解析出的
    `auth_mode` 为 `chatgpt` + 一个 refresh token —— 字段缺失时即 Codex 自身的旧版模式解析；keyring
    存储不留下文件，故不可见）；Copilot 编辑器插件与 Copilot CLI → `github-copilot`（在绝对
    `XDG_CONFIG_HOME` 下的 `github-copilot/{apps,hosts}.json`，否则 Windows 上绝对 `LOCALAPPDATA`
    （再否则 `~/AppData/Local`），再否则 `~/.config`；CLI 的带注释 `config.json` 的 `loggedInUsers`）；
    Kimi CLI → `kimi-coding`（带 refresh token 的 `credentials/kimi-code.json`）；Grok Build → `xai`
    （`auth.json` 中一个非 API-key 的 scope，带 refresh token 或未过期的会话）。过期的 access token
    只要配着有效的 refresh token 仍算数。Gemini CLI 与 Meta Muse 排除在外，因为 pi 没有匹配的 OAuth
    provider，或没有经验证的文件。每个信号点名候选 provider；报告当前 runtime 能 OAuth 登录的第一
    个，且一个 provider 只被报告一次（首个来源胜出）。套餐标签只在它是已知套餐名时才报告（`plus`、
    `pro`、`max`、`team`、`business`、`enterprise`、`edu`，小写化）—— 其他一切，包括 `free`，只丢掉
    徽标而不丢掉登录。文件只在它是常规文件且小于 1 MiB（Claude Code 的全局配置为 32 MiB，因为它
    携带逐项目历史）时读取，并容忍 BOM；每个来源获得 1.5 s，因此一个挂死的挂载点读作未登录，而不是
    把首次运行卡住。上限只停止等待，所以 I/O 本身是被另行限界的：无参（宿主）调用是单飞的，某个
    路径上仍在进行的读取会被复用直到它结算，因此在挂死的挂载点上重复检测永不堆叠文件系统操作、
    以致饿死宿主共享的 fs 线程池；任何不可读的东西都算作未登录。永不读 pi 的
    `auth.json`（pi 已把它报告为已配置），也永不读操作系统 keychain —— 一次 Keychain 读取会弹出系统
    访问提示。检测**只建议，从不借用**：用户仍通过 pi 自己的流程登录，因为复用另一个工具的轮换
    refresh token 会把两者之一登出。没有任何凭据值离开该函数；仅 API-key 的设置不算一次登录。
  - `providerLogin` — 应用内的凭据**写入**，无会话（登录在 Welcome 屏上运行，早于任何会话存在），
    因此是 `agent/webUiContext` 的一个以 `loginId` 为键的兄弟：
    - `startLogin(providerId, type = "oauth")` → `{ loginId }`，**同步**返回；
      `runtime.login(id, type, interaction, piLoginOptions)` **脱离地**运行（一个流程可能要几分钟 ——
      await 它会超出客户端请求超时并堵住 WS 泵）。**一个桥，两种认证类型**（issue #97）：`"oauth"`
      与 `"api_key"`（provider 自有的交互式 key 输入 —— 多数 provider 只有一个密钥提示，azure/vertex
      风格的凭据有多个提示）。`piLoginOptions`（agent barrel）从全局设置提供 pi 稳定的安装 `deviceId`，
      pi 的 openai「Sign in with ChatGPT」流程需要它（没有它会抛错）；与 pi 的 CLI 一样，它在首次
      使用时创建，从不给其他流程。pi 的 `AuthInteraction` 被接到 `provider.login` 通道上的
      `LoginFrame` 推送：`notify` 的 `auth_url`→`authUrl`、`device_code`→`deviceCode`、
      `progress`/`info`→`progress`（info 链接作为纯 URL 追加）；`prompt` 的 `select`→一个停放的
      `select` 帧，`text`/`secret`/`manual_code`→一个等待回复的停放 `prompt` 帧（`secret` 提示在帧上
      被标记，好让对话框遮蔽输入）。提示自身的 `signal` 中止（pi 取消它在浏览器 vs 粘贴竞速中的
      败者）会结算停放的输入 —— 带身份守卫，因此一次迟到的中止无法清掉一个更新的停放提示。pi 在
      **`login()` 内部**持久化凭据并刷新可用性，因此成功只需推 `success`；抛错时推 `error`。
    - `resolveLogin({ loginId, value })` — 浏览器的回复解析停放的交互。
    - `cancelLogin(loginId)` — 中止 signal **并**用 `undefined` 结算停放的输入（这会让等待中的
      `prompt` 抛错），因为仅靠 signal 不会停止 provider 的浏览器/回调服务器等待；`cancelAllLogins()`
      在宿主 `stop()` 时清扫它们。
    - **新的 `startLogin` 会取代**同一 provider 上任何未完成的登录：pi 一次只为一个 provider 运行一个
      凭据操作，因此一个停在提示处的登录（标签页关闭或重载、从未取消）否则会把之后每次尝试都堵在它
      后面的「Working…」上，直到宿主重启。被取代的登录会被取消，通过可选的
      `onSuperseded(loginId)` 回调报告（宿主借此丢弃它的分析条目），并收到一个终结 `error` 帧，因此
      仍在展示它的另一个客户端会停止等待；web store 永不会为一个它并不持有的登录从终结帧打开对话框。
    - `logoutProvider(id)` — `runtime.logout`（内部刷新），然后是 `setLogoutPublisher` 接缝（默认
      空操作），好让宿主广播这次变更。（旧的 `setProviderApiKey` —— 一个只回答单个密钥提示的罐头
      交互 —— 已随 `provider.setApiKey` 消失：对话框流程涵盖了它，并且还能服务多提示的 provider。）
    - `setLoginPublisher(fn)` — 推送接缝（默认空操作）。成功完成时还会单独提供执行该登录的代次作为
      仅宿主的上下文；它永不是 `LoginPush` 的一部分，也不被序列化到线上。
  - `jbcentral` — 应用内的 **JetBrains AI** 原生流程。它把 `shared/jbcentral` 的宿主机本地适配器与
    产物 watcher 同 `agent` 的候选代次接缝组合起来；它永不触碰管理器内部。宿主启动时启动 watcher 并
    在模型工作之前请求一次初始代次。一份经评审的已配置产物被作为不透明扩展路径传入；缺失或不支持的
    配置产生一个普通 runtime。若 Central 候选在初始启动时失败，auth 记录 `load-failed` 并向 agent
    索取一个普通 runtime，使 UI 与无关 provider 仍可用；而那个普通 runtime 失败仍会让启动失败。

    Connect 执行最低版本预检、运行 `central add pi`、校验产物存在，然后等待 watcher 所用的同一条重建
    路径。Disconnect 运行 `central remove pi`，并在产物存在时校验其缺失；产物本已缺失就是完整后置
    条件，此时直接重建普通 PI，即使 Central 本身此刻已缺失/不受支持（因此 Retry 能修复一个失败的
    普通候选）。Login 在版本预检之后启动 `central login` 并丢弃缓存的 status 观测，因为用户即将带外
    改变它；只有在子进程活过它的宽限期之后，这次启动才被报告为成功，因此无法启动的登录会以失败形式
    浮现，并给出宿主命令作为回退，而不是邀请用户在一个从未打开的浏览器里完成。
    Update 调用 `central update --install` 并复查状态。Start proxy 调用
    `central proxy start --ensure-updated`，失效共享的 status 观测，并校验一次新探测不再正面报告
    stopped；它不重建也不重新挂接 PI runtime。每个动作都使用解析后的绝对可执行文件。没有动作会编辑
    先前的模型配置、预检在线聊天模型、做补偿或回滚 Central 的全局状态。

    `getJbcentralQuota({ maxAgeMs, force })` 是独立的配额协调者。它只在当前生命周期满足卡片的
    Connected 谓词时才允许读取，然后把 `shared` 的纯数值读取包进一个进程全局、仅内存的缓存与单飞。
    一次普通读取复用比 `maxAgeMs` 更年轻的最近一次已完成尝试；force 绕过年龄但仍加入进行中的读取。
    失败时返回最后一次成功的数字并标为 `stale`，或在任何成功之前返回封闭的 `unavailable`。生命周期
    变化会同时清除新鲜度与旧的成功的回退值，因此另一个账号永不可能继承它；该边沿之后的读取会等过
    但永不加入/返回先前代次的进行中任务。值永不持久化。调用方提供由设置推导出的年龄，因此 auth 不
    导入 settings 兄弟模块。

    Watcher 事件被去抖/合并，每次重建都重新检验最新版本 + 产物后置条件。单调的请求序号阻止一个更旧
    的候选在更新的文件事件之后激活。成功时，auth 为 provider/模型读取与未来的会话激活该候选，清除
    `load-failed`，并发布模型/provider 失效通知。既有在线会话刻意保留它们创建时的 runtime —— 包括
    全局 Disconnect 之后一个以 Central 为后端的会话 —— 直到该会话被销毁或宿主重启。它们永不被 drain
    或重新挂接。

    候选失败会保留当前 runtime 并报告 `load-failed`；它永不会封死无关的模型动作。Retry、稍后的文件
    事件或 Disconnect 会请求另一次重建。一个进程范围内的单飞串行化 CLI 动作，而候选请求会合并到最新
    观测到的产物状态。状态与动作结果只使用封闭的 contracts 分类法 —— 永不用子进程输出、扩展内容、
    诊断信息、proxy 数据、原始模型或任意抛出的消息。
- **公开接口（barrel）：** `getProviderStatus`、`buildProviderReport`（+ `ProviderStatusSources`）；
  `detectAccounts`；
  `startLogin`、`resolveLogin`、`cancelLogin`、`cancelAllLogins`、`logoutProvider`、
  `setLoginPublisher`、`setLogoutPublisher`；`initializeJbcentralRuntime`、`stopJbcentralRuntime`、
  `getJbcentralStatus`、
  `getJbcentralQuota`、`connectJbcentral`、`disconnectJbcentral`、`startProxyJbcentral`、
  `updateJbcentral`、`jbcentralLogin`、成功动作 / runtime 变化的发布接缝，以及兄弟宿主测试所用的
  显式 `resetJbcentralStateForTests` 生命周期接缝。
- **允许的依赖：** `contracts`（线上类型）；`shared/jbcentral`；当前 runtime/认证门面加上候选
  prepare/activate 所需的 **`agent` barrel**；`@earendil-works/pi-ai`（仅认证交互**类型**）。
- **禁止：** 伸进 `agent` 内部（runtime 与代次变更只经由它的 barrel）；导入 `host`、`settings` 或任何
  其他兄弟模块；为 pi 的私有 provider 常量深导入它的 TUI（`modes/interactive/*`）；让凭据**值**上线。

## 必须做对（Get right）

- **`loginStart` 必须不 `await` 该流程** —— 返回句柄，脱离地运行 `login()`。
- **会话前的 runtime/认证读取与写入捕获当前代次。** pi 的登录/登出在本地同步被捕获的 runtime；在线
  会话保留自己的代次。产物漂移会构建并原子地激活一个新的当前代次，既不原地改动候选，也不原地改动
  既有会话。
- **API key 只经 `login(id, "api_key", interaction)` 持久化** —— `setRuntimeApiKey` 是会话生命期的
  覆盖层，会在宿主重启时悄悄丢掉该 key。该交互是真正的对话框桥，永不是罐头自动应答（罐头应答只能
  服务单提示 provider）。
- **Cancel 结算停放的 promise**，而不只是 `abort()`。
- 帧在客户端**累积**（`authUrl` + 粘贴 `prompt` 的竞速），因此终结的 `success`/`error` 才是关闭流程
  的东西 —— `terminate()` 保证每个 `loginId` 恰好一个终结结果。

## 消费方（Consumed by）

`host`（接线所有 `provider.*` 处理器 + `provider.login` 通道发布，并停止 Central watcher）；`agent`
**不**依赖 `auth`（这条边是单向的：`auth` → `agent`，经由当前 runtime 与候选代次接缝）。
