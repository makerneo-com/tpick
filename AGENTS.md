# AGENTS.md

> 给 AI 编码助手的项目指令。tpick 是一个**单文件、零依赖**的 Python CLI。

## 项目概述

**tpick** 是一个 fuzzy 风格的终端会话 & 项目目录选择器：

- 列出正在运行的 **tmux + zellij** 会话；
- 自动发现 **Claude Code / Codex** 最近用过的项目目录；
- 默认进入搜索模式，输入即过滤（会话名 / 目录名 / 工作目录），回车进入；
- 选中会话 → attach；选中目录 → 以文件夹名创建 zellij 会话并 `cd` 进去。

整个项目就是**一个可执行 Python 脚本** `tpick`（~450 行），加上 README / LICENSE。没有包结构、没有构建步骤。

## 设置命令

- 运行环境：Python 3.8+，macOS 或 Linux。
- **无依赖、无构建**。开发就是直接编辑 `tpick`：
  - 冒烟测试（非交互）：`python3 tpick -l < /dev/null`
  - 本地交互运行：`./tpick`（必须在交互式终端里）
- 安装到 PATH：`chmod +x tpick && cp tpick ~/.local/bin/`

## 代码风格

- **零依赖是核心约束**：只用 Python 标准库。不要引入第三方包（`rich` / `textual` / `prompt_toolkit` / `fzf` 绑定等）。
- 单文件：所有逻辑在 `tpick` 内，按 section 组织（`data` / `collectors` / `formatting` / `input` / `search` / `attach / launch` / `render` / `main`）。
- 命名直白、少语法糖、可读性优先于性能。
- **代码文案与注释用英文**；对话和 commit 用中文。
- `Item` dataclass 是统一数据模型——会话和目录都是 `Item`，用 `kind`（`session`/`dir`）和 `section`（`SESSIONS`/`DIRECTORIES`）区分。

## 测试说明

项目没有测试框架。改动后照下面的方式验证：

1. **非交互回归**：`python3 tpick -l < /dev/null` —— 应列出 `SESSIONS` + `DIRECTORIES` 两个区，数据正确。
2. **逻辑单测**：用 `importlib` 加载脚本，monkeypatch `os.execvp` / `os.read` / `select.select`，断言：
   - `attach()` 对 dir / tmux / zellij 构造的 argv 与 `chdir`；
   - `read_action()` 的按键映射（Enter=`\r`、Backspace=`\x7f`、方向键 `\x1b[A/B`、Ctrl-C=`\x03`、Ctrl-J=`\x0a`→down 等）；
   - `matches()` 覆盖 name + workdir、大小写不敏感。
3. **交互集成**：`pty.fork()` 起伪终端，发按键序列，按 `\x1b[H`（home cursor）切帧取**最后一帧**断言。注意 `collect_dirs` 要扫 Codex session 文件，首帧渲染较慢，drain 需 ≥ 2s。

## 提交规范

- 提交消息用**中文 + gitmoji**，如 `✨ feat: 支持直接创建 zellij 会话`。
- 约定式类型：`feat` / `fix` / `docs` / `refactor` / `chore`。
- 单文件项目，改完跑一遍冒烟 + 单测再提交。

## 架构 & 关键技术点（改代码前必读）

数据流：
- 非交互（`-l` / stdin 非 tty）：`collect_all()` → `print_list()`，每次全量实扫，**不读缓存、不起后台线程**（脚本和管道要真值，不能拿可能过期的缓存）。
- 交互：`load_cache()` 取缓存首帧秒开 → `search_loop()` 渲染 → 后台 daemon 增量 `collect_dirs()` 重算后经 `RefreshSlot` 刷新列表 → `attach()`。

**几个非显然、容易踩坑的点：**

- **zellij 会话工作目录**：zellij 不在 `list-sessions` 暴露 cwd。每个会话是一个 `zellij --server <socket>` 进程，socket 路径末段即会话名；用 `lsof -a -p <pid> -d cwd -Fn` 读该进程 cwd = 会话启动目录。改这里时保持「socket 末段 ↔ 会话名」的映射。
- **Claude Code 项目路径解码**：`~/.claude/projects/` 的子目录名是路径编码（`-Users-foo-bar`），解码规则是首字符 `-` 当 `/`、其余 `-`→`/`。路径段本身含 `-` 会产生歧义，所以**解码后必须 `os.path.isdir()` 验证**，不存在的丢弃。
- **Codex 目录来源**：扫 `~/.codex/sessions/**/*.jsonl` 里的 `"cwd"` 字段，按文件 mtime 取最近一次。**必须二进制读**（`open(path,"rb")` + bytes 正则 `rb'"cwd"\s*:\s*"([^"]+)"'`）——text 模式 `errors="replace"` 解码后字符数 ≠ 字节数，会让读后校验 `len(data)==st.st_size` 永远失败、文件级缓存永不命中。
- **raw tty 按键**：交互用 `tty.setraw`。`Enter` 是 `\r`（**不是** `\n`——`\n` == `\x0a` == Ctrl-J，被映射成 `down`）。`ESC` 常量必须是 **bytes** `b"\x1b"`（`os.read` 返回 bytes，和 str 比较会静默失败，导致 esc / 方向键全部失效）。方向键是 `\x1b[A/B/C/D` 多字节序列，读完 `\x1b` 后用 `select` 非阻塞读后续字节。
- **颜色与对齐**：ANSI 颜色码必须包在**已经 pad 到目标宽度的可见字符串**外面（如 `paint(f"{kind:<7}", code)`）；否则 `<7` 会按"可见字符 + 转义字符数"计算，列对齐会错乱。
- **attach 用 `os.execvp`**：选中目录时先 `os.chdir(path)` 再 execvp `zellij --session <basename>`，会话继承该 cwd（zellij 没有 `--cwd` flag）。execvp 成功不返回；末尾的 `return True` 仅为测试与可读性保留。
- **双 Esc 行为**：第一个 Esc 清空 query 回全列表；query 已空时第二个 Esc 才退出（Ctrl-C / Ctrl-D 仍立即退出）。
- **缓存与异步刷新**（仅交互路径）：`$XDG_CACHE_HOME/tpick/dirs.json`（默认 `~/.cache/tpick/dirs.json`）存两级缓存——`dirs`（上次 `collect_dirs` 结果拍平，首帧秒开）+ `files`（每个 jsonl 的 `{size,mtime,cwds}`，未变则跳过整文件读取，665 文件 454MB 扫描 ~700ms→~11ms）。首帧用缓存渲染，daemon 线程增量重算后经 `RefreshSlot` 刷新列表。几条纪律：① **只有 daemon 写缓存**，主线程只读；`files_index` 仅 daemon 读写无需锁，`RefreshSlot`（跨线程 list 引用替换）必须 `Lock`。② daemon 入口 `try/except BaseException` 吞异常——raw tty 下任何 `print`/栈回溯都会污染屏幕，collectors 禁止碰 stdout/stderr。③ **`_read_token` 首字节读必须带 `poll_timeout` 超时**返回 idle 哨兵（`b""`），否则无按键时 `search_loop` 阻塞在 `os.read`、后台刷新无法触发重绘；但 ESC 续读与多字节 UTF-8 那几次 `select` **保持固定 0.02/0.05 不变**（否则破坏 Esc/方向键/UTF-8 语义）。④ `search_loop` 用 `dirty` 标志避免 idle tick 空转重绘闪烁；后台刷新后用 `(kind, path or name)` 作 key 对齐 cursor，防止选中项跳走。⑤ append-only 的 jsonl 可能边写边读，读后用 `len(data)==st.st_size` 校验，不等就不缓存、下轮重读，避免尾部 cwd 永久丢失。

## 重要文件

- `tpick` —— 全部代码，唯一入口。
- `README.md` —— 中文用户文档。
- `LICENSE` —— MIT。
- `CLAUDE.md` —— 指向 `AGENTS.md` 的软链接，保持向后兼容。

## 平台与外部依赖

- 依赖系统命令：`tmux`、`zellij`（可选，缺哪个自动跳过）、`lsof`、`ps`。
- 用 POSIX `termios` / `tty` / `select` 做原始终端 IO，**Windows 暂不支持**。
- 纯本地工具：读 `~/.claude`、`~/.codex` 与运行中进程信息，**不上传任何数据**。

## 安全

- 不提交个人数据 / 密钥。`~/.claude`、`~/.codex` 仅在本机读取。
- 新增数据源时，确保只读本地文件、不外发。
