# Git 目录重组踩坑：reset 回滚后 CRLF 假变更

## 背景

对 Obsidian 笔记仓库的文件夹结构进行规范化重组，涉及：
- 目录重命名（如 `基础概念/` → `01基础概念/`）
- 目录扁平化（如 `DDD/01理论/01DDD基础概念/` → `DDD/01基础概念/`）
- 文件跨目录移动

操作前用户已用 `git commit` 做好备份。

---

## 操作过程

### 第一步：执行目录重组

使用 `mv` 命令批量移动目录和文件：

```bash
mkdir -p "01基础概念"
mv "01理论/01DDD基础概念"/* "01基础概念/"
rm -r "01理论"
```

涉及 6 个目录的 10+ 次文件移动操作，全部为**未提交的本地变更**。

### 第二步：用户反馈文件"找不到了"

用户在 Obsidian 中按原路径找不到文件（实际文件已被移动到新路径，但用户不知道），于是要求回滚。

### 第三步：执行回滚

```bash
git reset --hard HEAD
```

**问题 1**：`git reset --hard HEAD` 只恢复 git 跟踪的文件，但不会删除 `mv` + `mkdir` 新建的空目录。这些目录以 `??`（未跟踪）状态留在工作区中。

```bash
# 查看残留
git status --short
# => ?? .../DDD领域驱动设计/01基础概念/
# => ?? .../Java基础/并发编程/
# => ?? .../数据库/mysql/01基础概念/
# => ...（共 9 个空目录）
```

### 第四步：清理残留目录

```bash
git clean -fd -- <目录路径列表>
```

### 第五步：发现 HEAD 位置不对

用户期望回到所有变更之前的原始状态，但 `git reset --hard HEAD` 只回到当前 commit（用户自己的调整已被提交），未回到用户做任何操作前的状态。

**问题 2**：HEAD 停留在 `8d9bcf7`（包含用户自调内容），应回退到父 commit `ca6dacb`。

```bash
git reset --hard ca6dacb
```

### 第六步：CRLF 警告爆发

回滚后，Source Control 插件（VS Code）中大量 md 文件出现在变更列表，但查看 diff 全部显示 `unchanged`。

---

## 问题原因分析

| 症状 | 根因 |
|------|------|
| 文件出现在 changelist，diff 全为 unchanged | **换行符元数据不一致** |
| 只有警告无实际内容变更 | Windows 上 git 的 `core.autocrlf` 设置导致 |
| 多文件同时出现 | `git clean -fd` 和 `git reset` 重写了文件，改变了换行符检出方式 |

### 换行符机制

Windows 上 git 的默认行为：

```
core.autocrlf = true  # 检出时转 CRLF，提交时转 LF
```

当 `git reset --hard` 和 `git clean -fd` 将文件恢复到工作区时，git 会根据配置转换行尾。如果文件在仓库中是 LF，检出到 Windows 工作区会变成 CRLF。但这种"转换"会在 git 的 index 中留下不一致的记录，导致文件被标记为"已修改"但内容无差异。

### 残留问题：编辑器打开文件后再次出现假变更

即使 `git status` 完全干净，打开文件后 VS Code Source Control 仍可能再次显示假变更。原因排查：

```
git ls-files --debug <file>
# ctime: 1778616232:421619100
# mtime: 1778616232:421619100
# dev: 0    ino: 0
```

git 索引中的文件 stat 信息与操作系统不同步（Windows 不支持 inode，dev/ino 为 0）。VS Code 的文件监视器检测到文件被 `git clean -fd` / `git checkout` 等操作"触碰"（修改时间变化），就将其列入候选变更列表。虽然 `git diff` 确认无差异，但 Source Control 插件的缓存机制仍会短暂显示。

解决方案：

```bash
# 刷新 git 索引的 stat 信息
git update-index --refresh

# 如果仍然出现，重新 checkout 全部文件
git checkout HEAD -- .
```

---

## 解决方案

### 修复换行符假变更

```bash
# 让 git 重新规范化所有文件的换行符设置
git add --renormalize .
```

该命令会重新扫描所有文件，按照当前 git 配置重新计算换行符状态，清除虚假变更。

### 修复编辑器再次触发的假变更

```bash
# 刷新索引 stat 缓存
git update-index --refresh
```

如果 VS Code 仍然显示，重启 VS Code 窗口即可（Source Control 插件缓存导致）。

### VS Code 端的预防

如果问题反复出现，检查 VS Code 设置是否自动修改文件：

```json
// 检查这些设置
"files.autoSave": "off",                // 关闭自动保存
"files.trimTrailingWhitespace": false,  // 关闭尾部空格修剪
"files.insertFinalNewline": false,      // 关闭末尾新行插入
"editor.formatOnSave": false,           // 关闭保存时格式化
```

---

## 经验教训

### 1. 回滚前先确认 HEAD 位置

```bash
# ❌ 错误做法
git reset --hard HEAD

# ✅ 正确做法
git log --oneline -3        # 先看清当前在哪个 commit
git diff --name-only HEAD~1 HEAD  # 看上次提交改了哪些文件
# 确认无误后再操作
```

### 2. 目录操作建议分批提交

```bash
# 每完成一组相关的目录操作就提交一次
git add .
git commit -m "refactor: mysql目录添加编号前缀"

# 下一组
git add .
git commit -m "refactor: MQ子目录标准化"
```

这样：
- 每步都可独立回滚
- 回滚范围精确，不会出现残留目录
- 如果用户想保留部分改动，可以逐 commit 选择

### 3. `git reset --hard` 的局限性

`git reset --hard` **只影响 git 跟踪的文件**，不会清理：
- 新建的空目录（需 `git clean -fd`）
- 新建的未跟踪文件
- `.gitignore` 中的文件

如果需要**完全恢复到某个 commit 的状态**，组合命令：

```bash
git reset --hard <target-commit>
git clean -fd          # 删除所有未跟踪文件和目录
```

### 4. 不要直接用 `mv` 操作 git 跟踪的文件

文件被 `mv` 移动后，git 会丢失跟踪关系。改用 `git mv`：

```bash
# ❌ 错误：git 可能不识别为 rename
mv file.md newdir/

# ✅ 正确：git 明确识别为 rename
git mv file.md newdir/
```

或者先 `mv` 后用 `git add -A` 让 git 自动检测 rename。

### 5. Windows CRLF 问题的预防

在仓库根目录添加 `.gitattributes` 文件，明确声明 md 文件的换行符策略：

```gitattributes
# 让 git 对所有 md 文件使用 LF，不做自动转换
*.md text eol=lf
```

这样可以避免 Windows 上的 CRLF 假变更问题。

---

## 本次事故的正确操作流程（复盘）

应该这样做：

```bash
# 1. 先做备份
git add -A && git commit -m "backup before restructuring"

# 2. 分步操作，每一步提交
git mv 基础概念/ 01基础概念/
git commit -m "refactor: mysql 目录加编号前缀"

git mv CopyOnWriteArrayList.md Java基础/并发编程/
git commit -m "refactor: 归集 Java 并发笔记"

# 3. 如果需要回滚某一步
git revert HEAD    # 安全回滚，保留历史

# 4. 如果必须硬回滚
git reflog         # 先看操作记录
git reset --hard HEAD@{n}  # 指定精确位置
git clean -fd      # 清理残留
```

---

## 关键命令速查

| 场景 | 命令 |
|------|------|
| 看最近提交记录 | `git log --oneline -5` |
| 看当前 HEAD 位置 | `git branch -vv` |
| 看哪些文件被改了 | `git diff --name-only HEAD~1 HEAD` |
| 安全回滚（保留历史） | `git revert HEAD` |
| 硬回滚到指定 commit | `git reset --hard <commit>` |
| 清理未跟踪文件/目录 | `git clean -fd` |
| 修复换行符假变更 | `git add --renormalize .` |
| 刷新索引 stat 缓存 | `git update-index --refresh` |
| 查看操作历史 | `git reflog` |
