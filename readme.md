# 读懂 React Flow 源码

这个仓库是一组面向前端工程师的中文源码导读文章，目标不是教你“怎么使用 React Flow”，而是解释一个节点编辑器库如何从 `nodes` 和 `edges` 两个数组，长成一套能渲染、能缩放、能拖拽、能连线、能选择、能扩展，并把交互变化稳定交还给用户的运行时系统。

## 写作思路

这组文章不按目录树平铺源码，而是先抓住 React Flow 的承重链路：

```txt
用户 API
  -> ReactFlow 门面组件
  -> Wrapper / StoreUpdater
  -> Zustand Store
  -> GraphView 渲染总装层
  -> Viewport / EdgeRenderer / NodeRenderer / ConnectionLine
  -> system 层的 graph / viewport / drag / handle / panzoom 工具
  -> changes / callbacks 回流到用户状态
```

每篇文章都尽量遵守同一个节奏：先讲为什么需要这个机制，再讲它在用户 API 里表现成什么，然后回到源码里找关键文件、数据结构和调用链，最后用一个最小实现思路验证理解。这样读源码时不会停留在“把代码翻译成中文”，而是能看到它为什么要拆成现在的样子。

贯穿全系列的例子是“拖动一个节点”：它会经过坐标转换、拖拽控制器、store action、`NodeChange`、用户回调、`setNodes`、`InternalNode` 更新和画布重渲染。这个例子足够小，但能穿过 React Flow 最核心的运行时设计。

源码证据默认使用 xyflow 仓库相对路径；如果正文出现行号，默认基于 `xyflow@7e972983b392b747342cda68ece4e5bf082860a1`。版本更新后行号可能漂移，但文章关注的文件边界、调用关系和架构判断仍然是阅读重点。

## 章节目录

### 第一部分：建立问题域和源码地图

1. [React Flow 解决的到底是什么问题？](./zh/01-what-problem-react-flow-solves.md)  
   从“画节点和线”的常见误解切入，说明 React Flow 真正解决的是图编辑器运行时问题：数据模型、视口、渲染、交互和受控回流如何稳定协作。

2. [React Flow 的核心概念：Node、Edge、Handle、Viewport、Store](./zh/02-core-concept-map.md)  
   建立后续读源码需要的词汇表，解释节点、边、连接点、视口和状态中心分别承担什么责任，以及这些概念为什么不能简单合并。

3. [xyflow monorepo 架构：system / react / svelte 为什么要拆开？](./zh/03-xyflow-monorepo-architecture.md)  
   通过包边界理解 xyflow 的复用策略：哪些能力属于框架无关的图编辑器核心，哪些能力只是 React 或 Svelte 的运行时绑定。

4. [从入口文件看公共 API 设计](./zh/04-public-api-entrypoints.md)  
   从包入口和导出面观察公共 API 的组织方式，理解 ReactFlow、组件、hooks、类型和 system 工具如何被包装成用户可使用的接口。

### 第二部分：进入 React Flow 核心运行时

5. [ReactFlow 主组件：门面组件如何组织运行时？](./zh/05-reactflow-main-component-runtime-shell.md)  
   阅读 `ReactFlow` 主组件如何接收 props、创建宿主 DOM、挂载 provider、接入 children，并把用户 API 转成内部运行时需要的结构。

6. [GraphView：节点、边、连接线和 viewport 的渲染分层](./zh/06-graphview-layered-rendering.md)  
   解释 GraphView 为什么不是一个大组件，而是把 FlowRenderer、Viewport、EdgeRenderer、NodeRenderer 和 ConnectionLine 拆成清晰层级。

7. [Store：React Flow 的状态中心](./zh/07-store-state-center.md)  
   进入 Zustand store，理解 nodes、edges、lookup、transform、selection、connection 和 callbacks 如何被集中管理并通过 action 协作。

8. [InternalNode：为什么用户节点需要被增强？](./zh/08-internal-node-why-user-node-is-not-enough.md)  
   说明用户传入的 Node 为什么不够用，内部必须补充尺寸、绝对坐标、层级、handle bounds 和原始 userNode 引用。

9. [坐标系统：screen、container、flow、viewport](./zh/09-coordinate-system-screen-container-flow-viewport.md)  
   梳理屏幕坐标、容器坐标、flow 坐标和 viewport transform 的区别，解释拖拽、缩放、fitView 为什么都离不开坐标转换。

10. [XYPanZoom：缩放和平移系统](./zh/10-xypanzoom-pan-and-zoom-system.md)  
    阅读 panzoom 控制器如何接管 wheel、drag、zoom、translateExtent 和 viewport 约束，并把视口变化同步回 store。

11. [XYDrag：节点拖拽系统](./zh/11-xydrag-node-drag-system.md)  
    以节点拖拽为主线，解释 pointer 事件如何结合 zoom、snapGrid、nodeExtent 和 parent 约束，最终生成位置变化。

12. [XYHandle：Handle 和连线系统](./zh/12-xyhandle-handle-and-connection-system.md)  
    说明连接交互为什么独立于边渲染：Handle 负责命中、校验、连接状态和临时连接线，最终通过 onConnect 交还用户。

13. [Edge path：边路径与图工具函数](./zh/13-edge-path-and-graph-utils.md)  
    从 straight、bezier、smooth step 等路径函数理解 Edge 只是关系数据，真正的 SVG path 需要由节点位置、handle 和工具函数计算。

14. [controlled / uncontrolled：交互变化如何回流给用户？](./zh/14-controlled-uncontrolled-change-flow.md)  
    解释受控和非受控模式下 changes 的分流逻辑，为什么拖拽、选择、删除等交互最终都要变成 NodeChange 或 EdgeChange。

15. [Selection：点击、框选、多选和删除](./zh/15-selection-click-box-multi-delete.md)  
    阅读 selection 如何串起 Pane、store、键盘事件和 change 回调，理解单选、多选、框选、取消选择和删除的状态流。

16. [Hooks API：useReactFlow、useNodes、useEdges、useViewport](./zh/16-hooks-api-runtime-adapter.md)  
    观察 hooks 如何把内部 store 包装成稳定的用户 API，让外部代码既能读取运行时状态，也能调用 viewport、node、edge 操作。

17. [插件组件：Controls、Background、MiniMap、Panel](./zh/17-plugin-components-children-runtime.md)  
    解释插件组件不是装饰层，而是通过 store、portal、Panel 和 system 控制器接入运行时，复用主画布的状态与交互能力。

18. [性能设计：lookup map、selector、memo、visible elements](./zh/18-performance-lookup-selector-memo-visible-elements.md)  
    聚焦高频交互下的性能设计：lookup map、细粒度 selector、组件拆分、可见元素裁剪和引用复用如何减少无谓重渲染。

19. [React Flow 性能优化手段与原理](./zh/19-react-flow-performance-optimization-principles.md)  
    从项目落地和面试复盘角度，总结稳定引用、细粒度订阅、高频交互治理、大图渲染策略、节点/边瘦身和性能诊断方法。

### 第三部分：用 mini-flow 验证理解

20. [实战：实现最小 Graph Renderer](./zh/20-mini-flow-minimal-graph-renderer.md)  
    从最小节点和边渲染开始复刻画布骨架，不追求功能完整，而是验证 nodes、edges、NodeRenderer、EdgeRenderer 的基本分工。

21. [实战：实现 viewport、pan、zoom、fitView](./zh/21-mini-flow-viewport-pan-zoom-fitview.md)  
    给 mini-flow 加上视口系统，手写 transform、平移、缩放和 fitView，体会 viewport 为什么是图编辑器交互的地基。

22. [实战：实现节点拖拽和 onNodesChange](./zh/22-mini-flow-node-drag-onnodeschange.md)  
    复刻节点拖拽的最小链路：从 pointer movement 到 flow 坐标，再到 position change 和用户侧状态更新。

23. [实战：实现 Handle、ConnectionLine 和 onConnect](./zh/23-mini-flow-handle-connectionline-onconnect.md)  
    给 mini-flow 加入连接点、临时连接线和 onConnect 回调，验证连接系统为什么需要独立建模，而不是由边渲染顺手完成。

24. [实战：实现 store、Provider 和 hooks](./zh/24-mini-flow-store-provider-hooks.md)  
    把前面分散的状态收拢进 store 和 Provider，再写出最小 hooks，让 mini-flow 从组件组合走向真正的运行时结构。

25. [实战：实现 Controls、Background、MiniMap](./zh/25-mini-flow-controls-background-minimap.md)  
    实现常见插件组件，理解它们如何读取 viewport、节点 bounds 和画布尺寸，并通过公开能力反向控制主画布。

26. [总结：React Flow 源码的架构模式和可复用经验](./zh/26-react-flow-architecture-patterns-summary.md)  
    收束全系列，提炼 React Flow 在包边界、运行时拆分、交互回流、状态建模和性能优化上的可复用架构经验。

## 建议阅读顺序

如果你第一次系统读 React Flow 源码，建议按章节顺序阅读。前四篇先建立问题域、概念地图、包边界和 API 地图；第 5 到第 18 篇进入源码核心；第 19 篇补一层项目落地视角的性能优化总结；第 20 到第 26 篇再通过 mini-flow 把理解落到实现。

如果你已经会用 React Flow，但源码读不进去，可以优先看第 3、4、6、7、9、11、12 篇。它们能最快解释：React Flow 为什么不是一个“大 React 组件”，而是一套由数据、视口、渲染和交互共同组成的运行时架构。
