# tpick

> fuzzy 终端会话 & 项目目录选择器 —— 一个零依赖的 Python 脚本

把「在 tmux / zellij 之间切会话」和「进某个项目目录开个新会话」这两件高频小事，汇成一张可搜索的表。
默认进去就是搜索框，输入即过滤 **会话名 / 目录名 / 工作目录**，回车即进入。

- 🧩 **零依赖**：只用 Python 3 标准库（tmux / zellij 可选，缺哪个跳过哪个）
- 🔍 **fzf 风格**：实时过滤、上下选择、Enter 直达
- 🗂️ **自动发现项目目录**：从 Claude Code、Codex 的使用历史里挖出你最近工作过的目录
- ⚡ **选中目录即建会话**：自动以文件夹名创建 zellij 会话，并以该目录为工作目录
- 📺 **铺满屏幕**：列表随终端高度自适应，超长自动滚动跟随光标

## 它解决什么

日常终端里：

- 列出会话要分别敲 `tmux ls` / `zellij list-sessions`；
- 想进某个项目开新会话，要先 `cd` 再 `zellij --session xxx`；
- 会话一多，名字根本记不住。

tpick 把这些汇成一张可搜索的表，键盘搞定。

## 安装

```sh
git clone https://github.com/makerneo-com/tpick.git
cd tpick
chmod +x tpick
# 放到 PATH，例如：
cp tpick ~/.local/bin/      # 或 ~/bin、/usr/local/bin
```

需要 Python 3.8+。macOS 自带的 `python3` 即可。

## 用法

```sh
tpick          # 交互式搜索
tpick -l       # 非交互列表（适合管道 / 脚本）
```

### 按键

| 按键 | 作用 |
|---|---|
| 输入字符 | 实时过滤（匹配 名称 / 工作目录） |
| `↑`  `↓`  或  `Ctrl-J`  `Ctrl-K` | 上下移动选中行 |
| `Enter` | 进入：会话 → attach；目录 → `zellij --session <文件夹名>` 并 `cd` 进去 |
| `Backspace` | 删除一个字符 |
| `Esc` | 清空输入；输入已空时再按 `Esc` 退出 |
| `Ctrl-C` / `Ctrl-D` | 立即退出 |

> 选中「目录」时，tpick 会先 `cd` 到该目录再启动 zellij，会话以**文件夹名**命名；若同名会话已存在则直接附加。

## 数据来源

tpick 的列表分两个区：

### SESSIONS —— 正在运行的终端会话

- **tmux**：`tmux list-sessions`，工作目录取 active pane 的 `#{pane_current_path}`。
- **zellij**：`zellij list-sessions`。zellij 本身**不暴露**会话的工作目录，tpick 通过每个 `zellij --server` 进程的 `lsof -d cwd` 读到它的启动目录 —— 这是拿到 zellij 会话工作目录的可靠办法。

### DIRECTORIES —— 你最近工作过的项目目录

- **Claude Code**：扫描 `~/.claude/projects/`，目录名即项目路径（`-Users-foo-bar` → `/Users/foo/bar`）。
- **Codex**：扫描 `~/.codex/sessions/**/*.jsonl` 里的 `cwd` 字段。
- 已不存在（被删 / 路径迁移）的目录会被自动过滤；每条标注来源 `[claude]` / `[codex]` / `[claude+codex]`，按最近使用排序。
- **本地缓存加速**：目录列表缓存在 `~/.cache/tpick/dirs.json`，交互启动时先用缓存秒开首帧，后台再增量重扫刷新（未变化的 Codex 文件直接跳过，几百 MB 的扫描从秒级降到毫秒级）。数据始终以磁盘为准，缓存只用于加速；`tpick -l`（非交互）永远全量实扫、不读缓存。

## 平台

- macOS、Linux（依赖 `lsof`、`ps`、POSIX `termios`）
- Windows 暂未适配（终端原始模式 API 不同）

## 自定义

打开 `tpick`，顶部常量都好改：

```python
KEYS = list("asdfghjkl;qwertyuiopzxcvbnm,./1234567890")   # ts -l 列表的快捷键
CLAUDE_PROJECTS = os.path.join(HOME, ".claude", "projects")
CODEX_SESSIONS  = os.path.join(HOME, ".codex", "sessions")
```

## 作为 dotfiles 的一部分

如果你用 dotfiles 管理，把它放进 `$PATH` 目录即可，例如 `~/.bin/tpick`。

## License

[MIT](./LICENSE) © MakerNeo
