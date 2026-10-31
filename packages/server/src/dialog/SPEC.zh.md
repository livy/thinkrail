# dialog —— 原生文件夹选择器

> 中文译文，原文见 [`SPEC.md`](./SPEC.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

## 职责（Responsibility）

宿主的原生目录选择器，使浏览器里的“Open project”能弹出真正的操作系统对话框。

## 边界（Boundary）

- **拥有：** `selectDirectory()` —— 宿主的原生文件夹选择器，按操作系统经由 `pickersFor(platform)`
  选择：macOS 用 `osascript`（`choose folder`），Linux 先 `zenity` 后 `kdialog`，Windows 用
  PowerShell 的 `FolderBrowserDialog`。`THINKRAIL_PICK_DIR` 在开发/e2e 中覆盖它，并在原生可用性
  检查之前解析。它**只**在用户取消时返回 `null`。选择器的结束状态由退出码加诊断信息分类：
  AppleScript 的 `-128` 标记，以及 Linux 无 stderr 的退出码 1，是取消；Linux 带诊断信息的退出码 1
  （包括 GTK 的 `Failed to open display`）以及所有 PowerShell 非零退出都是失败。Linux 在既无
  `DISPLAY` 也无 `WAYLAND_DISPLAY` 时，在 spawn 之前就以可操作的“无图形会话”原因失败。缺失或
  失败的候选落到下一个候选；取消则停止。若所有候选都缺失或失败，该方法抛出最有用的已观测原因，
  因为静默的 `null` 就是一个死按钮。被选中的路径按宿主平台的路径规则读取：末尾的分隔符被丢弃
  （Windows 上 `/` 与 `\` 都丢，其他地方只丢 `/`），但文件系统根（`/`、`C:\`、`\\server\share\`）
  仍是根，因为单独的 `C:` 会是驱动器相对路径，而清空后的 `/` 会被读成取消。
  **文件间接（File-indirection）：** 当 `THINKRAIL_PICK_DIR` 指向一个已存在的*文件*时，其裁剪后
  的内容**每次调用都重新读取**。普通内容即返回的路径；`error:<message>` 会在原生可用性检查之前
  抛出该确定性失败。因此一个共享的源码、二进制或桌面 e2e 宿主，可以通过重写该指针把不同的文件夹
  或失败交给不同的测试（目录值按原样返回）。
- **Windows：对话框必须以聚焦状态出现在浏览器前方。** 宿主是后台进程，而 Windows 只允许*拥有*
  前台窗口的进程调用 `SetForegroundWindow`——所以朴素的 `ShowDialog()` 会开在浏览器后面、没有
  焦点，看起来就像“按钮没反应”。一个不可见的顶层所有者窗体**本身并不够**：在 Windows 11 上用
  另一个无关应用置于前台实测，对话框 3/3 次都未聚焦出现——可见，但键盘仍归浏览器。有效的是绕过
  前台锁的官方做法：脚本 `AttachThreadInput` 到前台窗口的线程（共享其输入队列使我们的
  `SetForegroundWindow` 合法），把所有者窗体置前，再解除附着，然后显示由该现已置前的窗体**拥有**
  的对话框。在 Windows 11 上实测：3/3 次既聚焦又置顶，Enter 选中高亮文件夹，Escape 以退出码 0
  返回 `null`。整个抢占都是**尽力而为**的——外层做了包裹，因此无法编译 P/Invoke 的宿主，或拒绝
  被抢占的提权前台应用，都会降级为旧行为（对话框仍会打开，只是在后面），而不是让选择失败。
  脚本通过 `-EncodedCommand`（base64/UTF-16LE）交付，而不是 `-Command`：脚本是多行的，且包含
  P/Invoke 签名的双引号，Windows 的 argv 引号处理加上 PowerShell 自身的二次解析无法保留它们。
- **公开接口（barrel）：** `selectDirectory`（+ `pickersFor` / `Picker`，以及两个消息构造函数
  `pickerFailure` / `noPickerMessage`，为单元测试而暴露）。
- **允许的依赖：** Bun（`Bun.spawn`，带 `windowsHide`；见 `architecture.md` 的 Invariants）、
  `process.env`。
- **禁止：** `host`；同级特性；`contracts`（不需要）。
