
<p align="center">
  <h1 align="center">📘 ZNotes</h1>
  <p align="center">
    借助 ChatGPT 辅助分析、系统化整理的个人学习笔记合集
  </p>


<p align="center">
  <img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/tool-Obsidian-7C3AED" alt="Obsidian">
  <img src="https://img.shields.io/badge/AI-ChatGPT-74AA9C" alt="ChatGPT">
</p>

---

## 📖 目录

- [关于本项目](#关于本项目)
- [内容结构](#内容结构)
- [阅读指南](#阅读指南)
- [知识领域](#知识领域)
- [如何使用](#如何使用)
- [鸣谢](#鸣谢)
- [许可证](#许可证)

---

## 关于本项目

**ZNotes** 是一套基于 [Obsidian](https://obsidian.md/) 构建的「第二大脑」知识库，涵盖了后端开发、AI 应用、开源项目研究等多个技术领域。所有笔记均经过系统化梳理，并借助 ChatGPT 进行辅助分析和内容优化，力求以清晰的结构呈现核心知识。

### 特点

- 📐 **结构清晰** —— 每个主题独立成篇，章节分明，开头结尾均设有总结性章节
- 🧠 **AI 辅助** —— 使用 ChatGPT 对复杂概念进行拆解、对比和归纳
- 🎨 **可视化** —— 大量 Mermaid 流程图、架构图、示意图辅助理解
- 🔗 **双向链接** —— 利用 Obsidian 的 Wiki-link 特性，构建知识间的关联网络
- ♻️ **持续更新** —— 随着学习深入和实践积累，笔记内容不断完善

---

## 内容结构

```
.
├── 01学习笔记/                   # 学习笔记主体
│   ├── AI学习笔记/                # AI & 机器学习
│   │   ├── AI科普/                #   AI 基础概念科普
│   │   ├── AI使用实践/             #   AI 工具使用经验
│   │   └── AI使用踩坑记录/         #   实践中的问题与解决方案
│   ├── Java学习笔记/              # Java 生态
│   │   ├── Java基础/              #   Java 核心基础知识
│   │   ├── Java21新特性/          #   Java 21 新特性详解
│   │   ├── spring/                #   Spring 框架
│   │   ├── redis/                 #   Redis 缓存
│   │   ├── 微服务/                #   微服务架构
│   │   ├── DDD领域驱动设计/        #   领域驱动设计
│   │   ├── K8S/                   #   Kubernetes
│   │   ├── 响应式编程/             #   Reactive Programming
│   │   └── 软件开发/              #   软件工程实践
│   └── github项目学习笔记/         # 优秀开源项目源码研读
├── 02输出/                        # 技术输出 & 博客文章
├── 03模板/                        # Obsidian 笔记模板
└── png/                           # 图片资源
```

---

## 阅读指南

推荐以下阅读顺序，以获得最佳学习效果：

1. **先通览全局** —— 阅读每个主题开头和结尾的总结性章节，快速建立知识框架
2. **建立认知地图** —— 结合 Mermaid 流程图、架构图和示意图，在脑中形成系统性的认知
3. **理论先行** —— 先过一遍理论概念，理解「是什么」和「为什么」
4. **代码验证** —— 结合具体的代码案例，回过头来再品读理论概念，形成「理论 ↔ 实践」的闭环

---

## 知识领域

| 领域 | 内容 |
|------|------|
| ☕ **Java** | 核心基础、Java 21 新特性、Spring 全家桶 |
| 🏗️ **架构设计** | 微服务、DDD 领域驱动设计、响应式编程 |
| ☁️ **云原生** | Kubernetes（K8S）基础与实践 |
| 🗄️ **数据存储** | Redis 缓存策略与实战 |
| 🤖 **人工智能** | AI 科普、ChatGPT 使用实践、踩坑记录 |
| 📖 **源码研读** | GitHub 优秀开源项目学习笔记 |
| 🛠️ **软件工程** | 开发规范、最佳实践 |

---

## 如何使用

### 在线阅读

直接通过 GitHub 浏览仓库中的 Markdown 文件即可阅读笔记内容。

### 本地浏览（推荐）

1. 克隆仓库到本地：

   ```bash
   git clone https://github.com/seaFall98/ZNotes.git
   ```

2. 使用 [Obsidian](https://obsidian.md/) 打开该文件夹作为 Vault，设置-文件与链接-附件文件夹路径-手动找到“附件”这个文件夹

3. 享受完整的双向链接、图谱视图和 Mermaid 图表渲染体验

---

## 鸣谢

- 感谢 [小傅哥 (CodeGuide)](https://github.com/fuzhengwei/CodeGuide) 的技术分享与启发
- 感谢所有笔记中引用的开源项目作者和社区贡献者

---

## 许可证

本项目采用 [MIT License](LICENSE) 开源协议。

---

<p align="center">
  <sub>Made with ❤️ and ☕</sub>
</p>
