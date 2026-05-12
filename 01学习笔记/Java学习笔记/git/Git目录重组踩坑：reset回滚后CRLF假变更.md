# Git 目录重组踩坑：reset 回滚后 CRLF 假变更

## 背景

对 Obsidian 笔记仓库的文件夹结构进行规范化重组，涉及：
- 目录重命名（如 `基础概念/` → `01基础概念/`）
- 目录扁平化（如 `DDD/01理论/01DDD基础概念/` → `DDD/01基础概念/`）
- 文件跨目录移动

操作前用户已用 `git commit` 做好备份。

---

## 事故链：六个连环问题

### 问题 1：`git reset --hard HEAD` 不清理新建的空目录

```bash
mkdir -p "01基础概念"
mv "01理论/01DDD基础概念"/* "01基础概念/"
```

`mv` + `mkdir` 创建了 git 不跟踪的新目录。回滚时：

```bash
git reset --hard HEAD      # 只恢复 git 跟踪的文件
# → 新建目录残留为 ?? (untracked)

git clean -fd              # 必须手动清理
```

**教训**：`git reset --hard` 只管跟踪文件，不管新建的空目录和未跟踪文件。

### 问题 2：回滚到了错误的 commit

`git reset --hard HEAD` 回退到的是**当前 HEAD**（已包含用户自己做的目录调整），不是原始状态。

```bash
git reset --hard ca6dacb  # 应回退到父 commit
```

**教训**：回滚前先用 `git log --oneline -3` 确认目标 commit。

### 问题 3：`git add --renormalize .` 制造了 index 与磁盘的换行符错位 ⚠️

**这是本次事故的核心根因。**

回滚后 Source Control 出现 CRLF 警告，我执行了：

```bash
git add --renormalize .
```

这个命令做了什么：

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  git 仓库   │     │  git index  │     │  磁盘文件   │
│   (LF)      │     │             │     │             │
├─────────────┤     ├─────────────┤     ├─────────────┤
│  操作前:    │ ←→ │  LF/CRLF ✓  │ ←→ │  CRLF       │  ← 一致，没问题
│             │     │             │     │             │
│  执行       │     │             │     │             │
│  renormalize│ ←→ │  LF  ✓      │ ←→ │  CRLF ✗     │  ← 不一致！
│  后:        │     │ (被重写)    │     │ (未改动)    │
└─────────────┘     └─────────────┘     └─────────────┘
                                             ↑
                                   VS Code 检测到差异
```

**`git add --renormalize .` 只更新了 index（清洗成 LF），但没有重写磁盘文件（磁盘还是 CRLF）。**

为什么 `git status` 不报？因为 `core.autocrlf=true` 告诉 git **忽略 CRLF 差异**。但 VS Code Source Control 插件用 libgit2 做自己的比对，它看到的是**实实在在的字节差异**（`\r\n` vs `\n`），于是标记为 changed。

**教训**：`git add --renormalize .` 不是万能修复命令。在 `core.autocrlf=true` 环境下使用它，会制造出 index 与磁盘的换行符不一致，而 `git status` 因为 autocrlf 的宽容性不会报告，但 GUI 工具可能检测到。

### 问题 4：多次 git 操作后 file stat 时间戳不同步

```bash
git ls-files --debug <file>
# ctime: 1778616232:421619100
# mtime: 1778616232:421619100  
# dev: 0    ino: 0              ← Windows 无 inode
```

`git reset --hard`、`git clean -fd`、`git checkout` 等操作反复改写了文件的修改时间戳。VS Code 的文件监视器检测到 `mtime` 变化，即使 diff 无差异仍将文件列入候选变更列表。

**修复**：`git update-index --refresh` 刷新 stat 缓存。

### 问题 5：`git reset --hard` 多次执行后出现重复文件

在不同 commit 间多次 reset，导致某些文件同时出现在旧路径和新路径下（git 跟踪版本和 mv 残留版本并存）。

**教训**：硬回滚后务必执行 `git clean -fd`。

### 问题 6：CRLF 假变更反复出现

即使执行了 `git update-index --refresh` 和 `git checkout HEAD -- .`，打开文件后 Source Control 仍显示变更。

这是因为 **`core.autocrlf=true` 的「双向转换」机制本身就是不稳定的**：

```
          checkout: LF → CRLF ✓
磁盘 ←───────────────────── git 仓库
          commit:  CRLF → LF ✓

但 index 在中间，三者的对齐依赖 smudge/clean filter
任何一次 reset / clean / checkout 都可能打乱对齐
```

---

## 根本解决方案

### 改 `core.autocrlf=true` 为 `input`

```bash
git config core.autocrlf input
```

| 模式 | checkout | commit | 磁盘文件 |
|------|----------|--------|----------|
| `true` | LF → CRLF | CRLF → LF | CRLF（转换过的） |
| `input` | **不转换** | CRLF → LF | LF（与仓库一致） |

`input` 的核心优势：**磁盘文件和 git 仓库使用相同的 LF，没有转换环节，index/磁盘/仓库天然一致。**

### 添加 `.gitattributes`

```bash
printf '* text=auto\n*.md text\n' > .gitattributes
```

### 重新 checkout 所有文件

```bash
git rm --cached -r .
git reset --hard HEAD
```

---

## 验证

```bash
# git 仓库中
git show HEAD:<file> | head -1 | od -c   # → \n (LF)

# 磁盘上
head -1 <file> | od -c                      # → \n (LF)

# → 完全一致，不存在任何差异 ✅
```

---

## 经验教训总结

### 教训 1：`git reset --hard` 不是"一键恢复"

它只恢复 git 跟踪的文件，不管：
- 新建的空目录（需 `git clean -fd`）
- 新建的未跟踪文件
- `.gitignore` 忽略的文件

### 教训 2：回滚前先确认目标 commit

```bash
# ❌ 错误
git reset --hard HEAD

# ✅ 正确
git log --oneline -3                # 确认在哪里
git diff --name-only HEAD~1 HEAD   # 看改了啥
```

### 教训 3：目录操作分批提交，不要批量 mv

```bash
git mv 基础概念/ 01基础概念/
git commit -m "refactor: mysql 目录加编号前缀"
# 下一步
git mv 实践/ 02实践/
git commit -m "refactor: mysql 实践目录编号"
```

每步可独立 revert，出错范围可控。

### 教训 4：用 `git mv` 而不是 `mv`

`mv` 后 git 可能不识别为 rename，用 `git mv` 明确告知 git。

### 教训 5：`core.autocrlf=true` 在 Windows 上有隐患

`true` 的 LF↔CRLF 双向转换依赖 smudge/clean filter。一旦 index 和磁盘之间出现不一致（`renormalize`、多次 reset 等操作触发），差异会被 `status` 隐藏但被 GUI 工具暴露。

**推荐**：用 `input` + `.gitattributes` 替代 `true`。

### 教训 6：`git add --renormalize .` 要谨慎使用

只更新 index，不改磁盘。在 `core.autocrlf=true` 下会导致静默不一致。

---

## 完整复盘：正确操作流程

```bash
# 1. 操作前备份
git add -A && git commit -m "backup before restructuring"

# 2. 分步操作
git mv 基础概念/ 01基础概念/
git commit -m "refactor: mysql 目录加编号前缀"

# 3. 回滚单步（安全，保留历史）
git revert HEAD

# 4. 如需完全回退到某个状态
git reflog                          # 看操作记录
git reset --hard <target-commit>
git clean -fd                       # 清理残留
```

---

## 关键命令速查

| 场景 | 命令 |
|------|------|
| 看最近提交记录 | `git log --oneline -5` |
| 看 HEAD 位置 | `git branch -vv` |
| 看两次提交的差异文件 | `git diff --name-only HEAD~1 HEAD` |
| 安全回滚（保留历史） | `git revert HEAD` |
| 硬回滚 | `git reset --hard <commit>` |
| 清理未跟踪文件/目录 | `git clean -fd` |
| 刷新索引 stat 缓存 | `git update-index --refresh` |
| 查看操作历史 | `git reflog` |
| 重新 checkout 所有文件 | `git rm --cached -r . && git reset --hard HEAD` |
| 设为 LF-only 模式 | `git config core.autocrlf input` |
