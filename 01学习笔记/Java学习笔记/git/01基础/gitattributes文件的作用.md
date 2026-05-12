`.gitattributes` 文件是 Git 的一个配置文件，用于为仓库中的**特定文件或路径**定义属性，从而精细控制 Git 的行为。它的核心作用是让 Git 在不同操作系统、不同开发场景下，对文件进行统一、正确的处理。

主要作用包括以下几点：

1.  **统一文件换行符 (LF vs CRLF)**
    这是最常见的用途。不同操作系统默认换行符不同（Windows 用 `CRLF`，Linux/macOS 用 `LF`）。混用会导致代码差异混乱、脚本执行报错。通过 `.gitattributes` 可以强制统一：
    ```gitattributes
    # 文本文件，提交到仓库时强制转换为 LF，检出到工作区时按当前系统转换
    * text=auto

    # 所有文本文件，提交时和检出后都强制使用 LF（推荐跨平台项目）
    * text eol=lf

    # 特定脚本文件，强制使用 LF
    *.sh text eol=lf

    # 批处理文件，强制使用 CRLF
    *.bat text eol=crlf
    ```

2.  **识别二进制文件，避免误处理**
    告诉 Git 哪些文件是二进制（不可读）而不是文本。防止 Git 对二进制文件尝试合并或展示差异（否则会出现乱码或巨大无意义的 diff）。
    ```gitattributes
    *.png binary
    *.jpg binary
    *.zip binary
    *.pdf binary
    ```
    这相当于设置了 `-diff -merge -text` 等属性。

3.  **指定差异对比 (diff) 工具**
    对于非代码文件（如文档、图片），可以让 Git 调用专门工具来显示差异。
    ```gitattributes
    # Excel 文件，调用自定义脚本展示差异
    *.xlsx diff=excel

    # Markdown 文件，即使内容改变，也尝试按单词而非整行对比
    *.md diff=markdown

    # 二进制图片，只显示文件大小变化（不显示内容）
    *.png diff=image
    ```

4.  **控制合并策略**
    某些文件（如版本号文件、锁文件）不应自动合并，可以强制在冲突时使用某版本。
    ```gitattributes
    # 遇到 package-lock.json 冲突时，始终使用当前分支的版本
    package-lock.json merge=ours

    # 或者始终用对方分支的版本
    config.json merge=theirs

    # 不要尝试合并该文件，直接标记为冲突
    database.db merge=binary
    ```

5.  **让 `export`（如 `git archive` 生成源码包）忽略某些文件**
    当用 `git archive` 导出项目快照（如打包发布 zip）时，可以排除开发文件。
    ```gitattributes
    # 导出时忽略测试和开发配置
    /tests/ export-ignore
    .gitignore export-ignore
    .env.example export-ignore
    ```

6.  **标记文件的语言或编码**
    辅助 GitHub 等平台正确识别项目主要语言或强制文件编码。
    ```gitattributes
    # 通知 GitHub 这个项目主要语言是 C++
    *.cpp linguist-language=C++
    # 把 .h 文件识别为 C++ 头文件而非 C
    *.h linguist-language=C++

    # 指定文件编码
    *.py working-tree-encoding=UTF-8
    ```

## 优先级与工作区

-   **版本控制**：`.gitattributes` 文件本身需要提交到仓库，团队成员都会共享这些规则。
-   **优先级**：`.gitattributes` 中的规则 > 项目本地配置（`core.autocrlf` 等）> 全局配置。
-   **应用时机**：通常在 `git add` 和 `git checkout` 时生效。修改 `.gitattributes` 后，可能需要执行 `git add --renormalize .` 来重新应用规则。

## 一个常用的通用示例

```gitattributes
# 自动检测文本文件，统一换行符
* text=auto

# 明确指定文本文件，统一用 LF
*.c text eol=lf
*.js text eol=lf
*.py text eol=lf
*.md text eol=lf
*.json text eol=lf

# 批处理文件必须用 CRLF
*.bat text eol=crlf
*.cmd text eol=crlf

# 二进制文件
*.png binary
*.jpg binary
*.ico binary
*.pdf binary
*.zip binary
*.exe binary

# 发布源码包时忽略
.gitattributes export-ignore
.gitignore export-ignore
```

**总结**：`.gitattributes` 就像一个文件级别的“规则引擎”，让你精确控制 Git 如何处理每一种文件，是保证跨平台协作、代码整洁和自动化流程的重要配置。