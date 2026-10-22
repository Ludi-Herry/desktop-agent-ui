# 桌面参考实现手册导读与深度工程机制

本手册梳理了现代桌面 Agent 界面系统的两代架构演进（从经典基础设计到 2.0 桌面工程底座），重点拆解结构分层、底层交互物理学、Zero-JS 渲染优化与生产级避坑机制。

---

## 核心设计与分卷导读

### 第一代设计体系（经典桌面 Agent 界面架构）

| 卷目 | 核心领域 | 重点覆盖与可迁移机制 |
| --- | --- | --- |
| **01 设计哲学与令牌体系** | Surface & Design Tokens | 多层 Surface 层级（Base / Elevated / Overlay）、颜色语义模型、字体层级、微阴影外扩与空间栅格系统 |
| **02 基础原子组件与交互控件** | Atomic Controls | 按钮三态与 Loading 锁定、分段控件（Segmented Control）、状态指示灯、自适应弹窗与上下文附件卡片 |
| **03 会话核心流与指令引擎** | Core Chat & Composer | 结构化消息流、思考过程时间轴折叠（Reasoning Timeline）、多模式 Composer、模型与工具执行授权卡 |
| **04 动效物理学与环境光影** | Motion Physics & Lighting | Spring 物理预设、线性扫光动效、动态渐隐遮罩、以及展开折叠的流式锁机制 |
| **05 视听媒体与产物交互** | Artifacts & Split Layout | 多媒体预览、可拖拽弹性分栏（Resizable Split）、代码差异对比（Diff）、双栏工作区布局 |
| **06 系统级桌面架构与协作** | Desktop Shell & Accessibility | 平台窗口外壳、无边框沉浸式标题栏、原生快捷键拦截、流式输出自动滚动与无障碍焦点树 |
| **07 工程配置与资产体系** | Build & Asset Topology | 桌面打包拓扑、代码分割、字体子集化内嵌与资产路径管理 |
| **08 Agent 前端实战指南** | End-to-End Implementation | 将原子组件、状态总线与事件驱动组合为生产级 Agent 工作台的实施流程 |

### 第二代架构进阶（现代生产级桌面平台底座）

针对大规模长会话、复杂工具调用、多窗口协作与极致渲染性能，现代桌面架构进行了系统级重构：

| 模块 | 演进重点 | 生产级工程要点 |
| --- | --- | --- |
| **综述与版本边界** | 架构目标升级 | 明确区分传统纯 Web 封装与现代桌面级原生能力的边界，确立多窗口流式总线 |
| **独立窗口与任务队列** | 多窗口与离线机制 | 弹出式独立子窗口（Popped-out Window）、离线失败重试队列、Composer 下颌功能行（Chin）与思维深度调节（Effort Tier） |
| **工作台架构与视觉密度** | 密度治理与用量可视化 | 紧凑（Compact）与舒适（Comfortable）双密度体系、长会话内存回收机制与配额实时消耗投影 |
| **系统底座解密** | 纯 CSS 动效与时间令牌 | 全量 CSS Keyframes 体系、微秒级时间令牌分层、以及统一命名规范 |
| **生产级参考实现** | 完整代码模式 | 窗口生命周期管理、多窗通信总线、事件防抖节流与兜底状态机 |

---

## 深度工程机制与关键实现

### 1. Zero-JS 纯 CSS 滚动驱动渐变遮罩（Scroll-Driven Animations）

在长对话流和消息容器的滚动边界处，传统方案采用 JS 监听 `scroll` 事件或使用 `IntersectionObserver` 动态计算 DOM 几何位置并设置遮罩透明度。在大批量流式输出时，频繁的 JS 测距会导致主线程严重的 Layout Thrashing 与帧率骤降。

**现代工程解法**：
在现代 Chromium 渲染内核下，全面升级为 **CSS 滚动驱动动效（CSS Scroll-Driven Animations）**：
- 利用 CSS Houdini `@property --cds-scroll-fade-*` 定义注册属性；
- 在合成器线程直接通过 `animation-timeline: scroll()` 驱动 6 条独立时间线；
- 达成 **0.0% CPU 占用** 与 **0ms 主线程阻塞**，彻底消除长列表滚动卡顿。

```css
@property --cds-scroll-fade-top {
  syntax: '<percentage>';
  inherits: false;
  initial-value: 0%;
}

.scroll-container {
  overflow-y: auto;
  scroll-timeline: --timeline-scroll y;
  mask-image: linear-gradient(
    to bottom,
    transparent 0%,
    black var(--cds-scroll-fade-top),
    black calc(100% - var(--cds-scroll-fade-bottom)),
    transparent 100%
  );
}

@keyframes scroll-fade-in {
  from { --cds-scroll-fade-top: 0%; }
  to { --cds-scroll-fade-top: 48px; }
}

.scroll-container {
  animation: scroll-fade-in linear;
  animation-timeline: --timeline-scroll;
  animation-range: 0px 48px;
}
```

### 2. WebKit / Chromium CJK 输入法 500ms 伴生 Enter 幽灵键工程守卫

在中文、日文等 CJK 拼音输入法下，用户在候选词浮层按下 `Enter` 键确认选词时，浏览器在触发 `compositionend` 事件的同时或数毫秒后，会紧随抛出一个 keyCode 为 13 的伴生 `keydown` 幽灵事件。若仅简单检查 `e.isComposing`，在某些输入法引擎或高负荷渲染时，该伴生事件可能在 `isComposing` 置为 false 之后才到达，导致用户仅仅想选词，消息却被直接误发送。

**精密守卫机制**：
- 监听 `compositionend`，记录发生的时间戳 `lastCompositionEndTimestamp = performance.now()`；
- 在输入框 `keydown` 事件中，判断若 `performance.now() - lastCompositionEndTimestamp < 500`，说明此 `Enter` 为伴生幽灵键，坚决拦截；
- **核心闭环**：**在成功拦截该伴生键后，必须立即复位时间戳**（例如重置为 0），杜绝后续正常的快速再次击键被误吞。

```typescript
let lastCompositionEndTime = 0;

export function handleCompositionEnd() {
  lastCompositionEndTime = performance.now();
}

export function handleComposerKeyDown(e: React.KeyboardEvent<HTMLTextAreaElement>, onSend: () => void) {
  if (e.key === 'Enter' && !e.shiftKey) {
    if (e.nativeEvent.isComposing) {
      return; // 原生组字中直接放行
    }
    const elapsed = performance.now() - lastCompositionEndTime;
    if (elapsed < 500) {
      // 捕获并拦截伴生幽灵 Enter，随后立刻清空时间戳，保护下一按键
      e.preventDefault();
      lastCompositionEndTime = 0;
      return;
    }
    // 正常发送逻辑
    e.preventDefault();
    onSend();
  }
}
```

### 3. 分栏拖拽（18%/82% 折叠吸附）与 Escape 键盘取消状态机闭环

在复杂工作台分栏（如对话流 + Artifact 产物预览）中，拖拽手柄移动至极值边缘（≤18% 或 ≥82%）时会自动吸附折叠以释放主工作空间。

**状态机死锁陷阱与防御**：
当处于拖拽或吸附临界状态时，若用户按下 `Escape` 键意图取消拖拽，容易出现手柄 DOM 被直接卸载而 `isCollapsed` 状态未正确恢复，导致分栏无法再次展开的永久死锁。
**工程闭环**：
在全局键盘监听中捕获拖拽期间的 `Escape` 事件，必须严格执行三步状态机复位：
1. 取消全局事件监听与 `isDragging` 标记；
2. 显式调用 `setIsRightCollapsed(false)`，确保分割手柄组件保持挂载；
3. 将分栏比例恢复至拖拽前的有效基准 `previousRatioRef.current`。

### 4. 多窗口状态一致性与工作区写锁（Broker-Replica 模式）

桌面应用允许将对话、文档或代码预览弹出为独立窗口。多窗口协同面临主进程与各渲染窗口状态分叉的风险：
- **Broker-Replica 拓扑**：主窗口作为状态权威 Broker，弹出窗口作为 Replica 订阅不可变状态流（Immutable State Stream）；
- **工作区写锁（Workspace Baseline Lock）**：当子窗口试图写入或修改某个文件对象时，必须先向主进程申领乐观基线锁；主进程比对基线版本哈希，仲裁是否存在冲突，防止并发脏写覆盖。

---

## 可迁移的设计方法总结

1. **界面区域正交分工**：窗口外壳控制区、导航区、主会话区、执行过程区与产物工作台保持清晰的物理层级与滚动隔离。
2. **状态与事件严格对齐**：工具卡片（Tool Card）从调用发起、参数渲染、等待人工授权到执行结果，必须严格绑定单一的唯一调用 ID，禁止凭定时器推进状态。
3. **滚动服从阅读意向**：新内容追加或折叠面板展开时，计算用户视口位置；用户向上翻阅时冻结自动滚动，回到底部时自动恢复跟随。
4. **统一设计令牌（Design Tokens）先行**：在具体组件编写前，确立语义化 Surface、文本阶梯、圆角与间距规范，阻断均值收敛导致的千篇一律。
