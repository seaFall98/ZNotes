# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository type

This is an Obsidian vault / Markdown knowledge base, not an application codebase. There are no package manifests or build/test scripts in the repository, so do not invent npm/maven/pytest commands.

## Common commands

- Check repository state: `git status --short`
- Review changed files: `git diff --stat` and `git diff -- <path>`
- Find notes by filename: use Claude Code `Glob` with patterns like `**/*关键词*.md`
- Search note content: use Claude Code `Grep` over `*.md` files
- There is no build, lint, or test command configured. For Markdown changes, verify by reading the edited file and, when relevant, checking Obsidian-compatible wiki links and Mermaid code fences manually.

## Vault structure

- `01学习笔记/` is the main note collection.
  - `AI学习笔记/` covers AI concepts, AI usage practice, Claude Code/Codex notes, and AI pitfalls.
  - `Java学习笔记/` covers Java foundations, Java 21, Spring, MyBatis, middleware, distributed systems, DDD, JVM, microservices, Kubernetes, Redis, and software-development practice.
  - `github项目学习笔记/` contains source-reading notes for open-source projects.
- `02输出/` contains polished output/blog-style articles derived from the notes.
- `03模板/` contains Obsidian templates. The current note template is `03模板/属性模板.md` with frontmatter fields `tags`, `description`, `aliases`, `created`, and `author`.
- `Clippings/` contains clipped/reference articles.
- `附件/` contains image attachments; `.obsidian/app.json` sets this as the attachment folder.
- `README.md` describes the project as “ZNotes”, a ChatGPT-assisted personal learning-note vault for backend development, AI applications, and open-source project study.

## Obsidian conventions

- Keep notes as standard Markdown compatible with Obsidian.
- Wiki links (`[[...]]`) and embedded attachments (`![[...]]`) are used throughout the vault; preserve or update them when renaming/moving notes.
- Mermaid diagrams are common and should remain fenced as ```` ```mermaid ```` blocks.
- Obsidian is configured with `alwaysUpdateLinks: true`, templates enabled with folder `03模板`, and community plugins `obsidian-git`, `obsidian42-brat`, and `realclaudian`.
- Obsidian Git uses commit messages like `vault backup: {{date}}` with `YYYY-MM-DD HH:mm:ss` formatting and pulls before push.

## Editing guidance

- Prefer Chinese for vault content unless the target note is already primarily English.
- Match the surrounding note’s heading style and depth; many long-form notes use numbered section headings and horizontal rules between major sections.
- For new notes, use the existing template frontmatter shape when appropriate:
  ```yaml
  ---
  tags:
  description:
  aliases:
  created: YYYY-MM-DD
  author: z
  ---
  ```
- Avoid broad automated formatting across the vault. This repository has many long Markdown notes, Chinese filenames, wiki links, and Mermaid diagrams; make narrowly scoped edits.

## Git/repository notes

- `.gitignore` excludes `.obsidian/`, `.claude/`, and `.claudian/`, but those directories may still exist locally. Do not rely on them being committed unless `git status` confirms it.
- `.gitattributes` sets text normalization and treats Markdown files as text.
