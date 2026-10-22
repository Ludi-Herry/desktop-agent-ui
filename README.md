# Desktop Agent UI

> 现代桌面 Agent / AI 应用的设计规范、交互物理学与深度工程实践指南。

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Skill: Agent UI](https://img.shields.io/badge/Skill-Desktop%20Agent%20UI-emerald.svg)](SKILL.md)

---

## 📖 项目简介

**Desktop Agent UI** 是一套专为现代桌面 AI / Agent 应用打造的设计系统与工程避坑规范。

在桌面大屏、长文本会话流、异步工具调用与复杂多面板工作台的场景下，传统的 Web 聊天界面往往会遭遇严重的阴影裁切、滚动性能瓶颈、输入法误触与状态机死锁。本项目沉淀了业界经过大规模生产验证的深度交互物理学、CSS 规范标准与自动化验收工具，帮助开发者打造丝滑、可控且坚固的桌面级 Agent 用户体验。

---

## ⚡ 核心工程亮点

1. **Zero-JS 纯 CSS 滚动渐变遮罩（Scroll-Driven Animations）**：
   - Chromium 115+ 环境下利用 CSS Houdini `@property` 与 `animation-timeline: scroll()` 在合成器线程驱动；
   - 达成 **0.0% CPU 占用** 与 **0ms 主线程阻塞**，彻底消除长列表滚动卡顿。
2. **CJK 输入法 500ms 伴生 Enter 幽灵键精密守卫**：
   - 监听 `compositionend` 时间戳，精准拦截中文拼音选词确认瞬间伴生的冒充 Enter；
   - 拦截后立即复位时间戳，保证正常连续击键畅通无阻，杜绝消息误发。
3. **分栏拖拽（18%/82% 折叠吸附）与 Escape 取消状态机闭环**：
   - 双栏工作台边缘自适应吸附折叠；
   - 捕获拖拽中途的 `Escape` 键，执行安全的三步状态机复位，防止手柄 DOM 卸载导致界面死锁。
4. **内外分层的阴影与裁切规约 + 自动化审计脚本**：
   - 严格遵循 `offset + blur + spread` 物理外扩余量与内外分层（内层圆角、外层投影）；
   - 内置 [`scripts/clip-audit.js`](scripts/clip-audit.js)，一键在 DevTools / 浏览器环境中自动化遍历所有祖先容器，以 `cut === 0` 作为交付硬性门禁。
5. **智能自适应增高输入框（Composer）与局部滚动条收敛**：
   - 结合 `field-sizing: content` 与尺寸监听的弹性高度过渡；
   - 达到最大高度后局部消除粗糙的灰色系统滚动条外观，同时完整保留原生滚动语义与可访问性。
6. **多窗口状态一致性与工作区写锁（Broker-Replica）**：
   - 主窗口权威 Broker 与弹出子窗口 Replica 订阅不可变状态流，通过乐观基线锁仲裁并发冲突。

---

## 💡 设计系统与技术栈边界（Framework & Design System Agnostic）

- **不强制绑定特定组件库**：本 Skill **不设定、不限定任何特定 UI 库**（无论您的项目使用 `shadcn/ui`、`Ant Design`、`Radix UI`、`Tailwind CSS` 还是原生自绘，本规约均 100% 适用）；
- **不包含预制代码模板**：本 Skill **不包含任何预制的设计系统模板或样式代码包**。它的核心定位是桌面 Agent 场景下的**底层交互物理学、状态机设计原则、输入法防误触与自动化避坑验收门禁**；
- **Agent 执行决策流**：
  1. **已有项目**：Agent 优先自动检测项目环境，**严格沿用项目既有的成熟设计系统与组件库**，严禁引入重复依赖；
  2. **从零起步**：若未检测到既定设计系统，Agent 会主动询问您是否有指定的组件库偏好；
  3. **若需使用 Claude 设计系统风格**：若您希望复用或参考 Claude 的桌面设计系统（Claude Design System），可直接前往上游 [Open Claude UI](https://github.com/Caho1/open-claude-ui) 与 [在线文档](https://openclaudeui.com/) 获取其完整的 Design Tokens 与组件源码。

---

## 📂 仓库结构

```text
desktop-agent-ui/
├── SKILL.md                          # Skill 入口定义与推荐工作流
├── README.md                         # 仓库介绍与使用说明
├── LICENSE                           # MIT 开源协议
├── references/
│   ├── open-claude-manual.md         # 现代桌面体系导读与核心工程机制
│   ├── desktop-pitfalls.md           # 桌面工程避坑指南（阴影、IME、焦点、滚动跳动）
│   ├── agent-interaction.md          # Agent 交互结构、生命周期与 RICH 委托控制权
│   ├── components-motion.md          # 核心组件实现、自适应高度配方与动效物理学
│   ├── acceptance.md                 # 质量验收清单与 clip-audit 审计指南
│   └── sources.md                    # 架构分类与技术参考索引（ENG-01 ~ ENG-14）
└── scripts/
    └── clip-audit.js                 # 阴影与祖先裁切自动化计算脚本
```

---

## 🚀 使用方法

### 作为 Agent Skill 使用
将本仓库放置或链接至您的 Agent 技能目录（如 `~/.agents/skills/desktop-agent-ui` 或 `~/.gemini/antigravity-cli/skills/desktop-agent-ui`），Agent 在处理桌面端应用、AI 对话工作台或修复窗口裁切问题时将自动选用并遵照规范执行。

### 运行自动化阴影裁切审计
在浏览器控制台或自动化测试脚本（Puppeteer / Playwright）中运行内置的 [`scripts/clip-audit.js`](scripts/clip-audit.js)：
```javascript
// 在无边框透明窗口下进行全量审计
const audit = (/* 将 clip-audit.js 内容粘贴于此 */);
const result = audit({ viewport: true, tol: 1 });
console.log(result); // 确保 result.cut === 0
```

---

## 🙏 致谢（Acknowledgments）

本项目深度参考、吸收并致敬以下优秀的上游开源工作与技术成果：

1. [**Open Claude UI 手册** (`Caho1/open-claude-ui`)](https://github.com/Caho1/open-claude-ui)（在线文档：[openclaudeui.com](https://openclaudeui.com/)）：感谢上游手册项目对现代桌面级 Agent 客户端界面拓扑、设计令牌体系、原子控件与架构细节的系统性拆解与开创性整理；
2. [**REA（Reverse Engineered Artifacts）仓库** (`morluto/rea`)](https://github.com/morluto/rea)：感谢 REA 项目提供的底层精度审计、深入逆向洞察与生产级架构参考实现支持；
3. 感谢社区相关桌面设计规范与动效物理学 Skill 的启发与贡献。

---

## 📄 开源许可证

本项目采用 [MIT 许可证](LICENSE)。
