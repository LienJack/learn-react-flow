---
title: "第 24 篇：实战：实现 Controls、Background、MiniMap"
tags:
  - react-flow
  - xyflow
  - source-code
  - mini-flow
  - controls
  - background
  - minimap
  - plugin
---

# 第 24 篇：实战：实现 Controls、Background、MiniMap

第 23 篇我们把 mini-flow 的运行时状态收进了 store：

```txt
MiniFlowProvider
  ↓
store
  ↓
useMiniStore / useMiniFlow
  ↓
renderers / interactions / plugins
```

这一篇要验证一件事：

> 当 store 和 hooks 建起来之后，插件组件其实不需要特权。

这句话听起来很轻，但它背后是 React Flow 设计里很重要的一层抽象。

很多人第一次看 React Flow 的时候，会把这些组件理解成“附属 UI”：

```tsx
<ReactFlow nodes={nodes} edges={edges}>
  <Controls />
  <Background />
  <MiniMap />
</ReactFlow>
```

于是会觉得：

```txt
ReactFlow 是主角
Controls / Background / MiniMap 是主组件内置的一些小挂件
```

这个理解不算错，但太表层。

更准确地说：

```txt
ReactFlow 提供同一个 graph runtime
Controls / Background / MiniMap 作为 children 接入这个 runtime
它们通过 hooks 读写 store
```

也就是说，插件组件不是被 `ReactFlow` 特殊识别出来再渲染的。

它们只是普通 React children。

真正让它们工作的，是第 23 篇建立的共享 store。

这一篇我们用 mini-flow 实现三个插件：

```txt
Controls
  验证命令式 viewport 操作

Background
  验证 transform 下的辅助渲染

MiniMap
  验证 nodes + viewport 的二级视图
```

顺手再实现一个更基础的 `Panel`：

```txt
Panel
  负责把插件定位到 viewport 上方的某个角落
```

这四个组件加起来，会把 React Flow 的插件接入方式讲清楚。

本章新增 / 修改：

```txt
src/mini-flow/components/MiniPanel.tsx
src/mini-flow/components/MiniControls.tsx
src/mini-flow/components/MiniBackground.tsx
src/mini-flow/components/MiniMap.tsx
src/mini-flow/minimap-utils.ts
src/mini-flow/mini-flow.css
```

本章验收 checklist：

```txt
- Controls 调用 useMiniFlow 命令，不直接改 DOM transform。
- Background 订阅 viewport，pan / zoom 时 pattern 跟着变化。
- MiniMap 能根据 nodes bounds 和当前 viewport 画出缩略视图。
- 插件作为 children 渲染，MiniFlow 不需要识别插件类型。
- 没有 Provider 时，插件 hooks 能给出明确错误。
```

和真实 React Flow 的差异：

| mini-flow 第 24 篇 | 真实 React Flow |
| --- | --- |
| MiniMap 第一版只展示 | `XYMinimap` 支持 pan / zoom 主画布 |
| Panel 只做固定定位 | React Flow 还处理 className、style、position 约定 |
| Background 简化 pattern | 真实实现处理 variant、gap、size、rfId、lineWidth |
| 暂不实现 Toolbar / NodeResizer | 真实插件还覆盖 portal 和 resizer controller |

MiniMap 几何先记住这条链：

```txt
viewport -> viewBB
nodes -> nodesBounds
nodesBounds + viewBB -> minimap bounds
flow point -> minimap point
```

`viewBB` 不是屏幕矩形缩小版，而是“当前主视口反算到 flow 坐标后的矩形”。MiniMap 再把 flow 坐标按比例映射到自己的 SVG 坐标系里。

---

## 1. 这一篇要解决的问题

上一章之后，mini-flow 已经有了这些能力：

```txt
useNodes()
useEdges()
useViewport()
useMiniFlow()
useMiniStore(selector)
```

现在我们要问：

> 一个不在 GraphView 内部的组件，怎么参与这个 flow runtime？

比如 `Controls`。

它看起来只是几个按钮：

```txt
+
-
fit
lock
```

但它实际要读写这些运行时信息：

```txt
当前 zoom
minZoom / maxZoom
是否允许交互
zoomIn / zoomOut / fitView
```

再看 `Background`。

它看起来只是背景网格。

但它必须随着 viewport 变化：

```txt
pan 时网格要平移
zoom 时网格间距要缩放
多个 background 要有独立 pattern id
```

否则用户拖动画布时，背景会像贴在屏幕上的壁纸，而不是图坐标系的一部分。

再看 `MiniMap`。

它更不像普通 UI。

它要同时知道：

```txt
所有节点的 bounds
当前 viewport 在 flow 坐标里的范围
主画布的 panZoom controller
小地图自己的 pannable / zoomable 配置
```

它不是简单把节点缩小画一遍。

它是主画布 runtime 的第二个视图。

所以这一篇真正要解决的问题不是“怎么写三个组件”，而是：

```txt
一个图编辑器库如何让插件组件安全地共享运行时状态？
```

答案就是：

```txt
children 插件接入
  + shared store
  + hooks 分层
  + Panel 浮层定位
  + system controller optional
```

---

## 2. 先看用户 API 或运行效果

我们希望 mini-flow 支持这样的写法：

```tsx
export function App() {
  return (
    <MiniFlow defaultNodes={nodes} defaultEdges={edges}>
      <MiniBackground />
      <MiniControls />
      <MiniMap />
    </MiniFlow>
  );
}
```

这里有一个很关键的事实：

```txt
MiniFlow 不需要知道 children 里放了什么。
```

它不应该写成：

```tsx
function MiniFlow(props) {
  return (
    <>
      <GraphView />
      {props.showControls && <Controls />}
      {props.showBackground && <Background />}
      {props.showMiniMap && <MiniMap />}
    </>
  );
}
```

这种写法会让主组件变成配置集合。

每加一个插件，就要给 `MiniFlow` 加 props。

React Flow 选择的是另一条路：

```tsx
<ReactFlow>
  <Background />
  <Controls />
  <MiniMap />
  <Panel position="top-right">...</Panel>
</ReactFlow>
```

这让插件有了组合性：

```tsx
<Controls position="bottom-left">
  <ControlButton onClick={exportImage}>
    Export
  </ControlButton>
</Controls>
```

也让用户可以自己写插件：

```tsx
function NodeCounter() {
  const nodes = useNodes();

  return (
    <MiniPanel position="top-right">
      {nodes.length} nodes
    </MiniPanel>
  );
}
```

这类组件没有任何特殊注册过程。

它只需要满足一个条件：

```txt
它在 MiniFlowProvider 子树里
```

这就是第 23 篇的 store 和 hooks 在这一篇发挥作用的地方。

---

## 3. 核心概念解释

先把四个组件的职责分清。

| 组件 | 角色 | 依赖的 runtime 信息 | 典型动作 |
| --- | --- | --- | --- |
| `Panel` | 浮层定位容器 | 无，主要依赖 CSS | top-left / bottom-right 等定位 |
| `Controls` | 命令式控制面板 | viewport、minZoom、maxZoom、interactivity | zoomIn、zoomOut、fitView、toggle interaction |
| `Background` | 辅助坐标背景 | viewport transform | 计算 pattern offset / gap / scale |
| `MiniMap` | 二级视图 | nodes、node bounds、viewport、panZoom | 显示全局概览、显示当前视口、可选 pan / zoom |

它们代表三种插件模式。

第一种：纯布局插件。

```txt
Panel
```

它不关心 flow 状态，只负责把 children 放到某个位置。

第二种：读写 store 的插件。

```txt
Controls
```

它读取当前 zoom 和交互开关，调用 `useMiniFlow()` 改 viewport。

第三种：订阅 transform 的辅助渲染。

```txt
Background
```

它不改变 runtime，只根据 viewport 派生 SVG pattern。

第四种：带独立 controller 的插件。

```txt
MiniMap
```

它既读取 store，又把 system 层的 controller 挂到自己的 DOM 上。

React Flow 的真实源码也是这几种模式混合。

`Controls` 在 `packages/react/src/additional-components/Controls/Controls.tsx` 里同时使用：

```txt
useStore
useStoreApi
useReactFlow
Panel
```

`Background` 在 `packages/react/src/additional-components/Background/Background.tsx` 里读取：

```txt
transform
patternId
```

然后计算 SVG pattern 的 `x`、`y`、`width`、`height` 和 `patternTransform`。

`MiniMap` 在 `packages/react/src/additional-components/MiniMap/MiniMap.tsx` 里读取：

```txt
viewBB
boundingRect
panZoom
translateExtent
flowWidth / flowHeight
```

并在 `useEffect` 中创建 system 层 `XYMinimap`。

这说明 additional components 不是一层“装饰组件集合”。

它们是 React Flow runtime 的外接操作面。

---

## 4. 源码入口在哪里

这一篇主要读这些文件：

```txt
packages/react/src/additional-components/index.ts
packages/react/src/additional-components/Controls/Controls.tsx
packages/react/src/additional-components/Controls/ControlButton.tsx
packages/react/src/additional-components/Background/Background.tsx
packages/react/src/additional-components/Background/Patterns.tsx
packages/react/src/additional-components/MiniMap/MiniMap.tsx
packages/react/src/additional-components/MiniMap/MiniMapNodes.tsx
packages/react/src/additional-components/MiniMap/MiniMapNode.tsx
packages/react/src/components/Panel/index.tsx
packages/system/src/xyminimap/index.ts
```

入口关系可以这样看：

```txt
packages/react/src/index.ts
  ↓
export * from './additional-components'
  ↓
additional-components/index.ts
  ↓
Background / Controls / MiniMap / NodeResizer / NodeToolbar / EdgeToolbar
```

React 包入口直接导出 additional components。

这意味着：

```tsx
import { ReactFlow, Controls, Background, MiniMap } from "@xyflow/react";
```

这些组件是正式公共 API，不是内部私有小组件。

再往下看组件实现。

`Controls` 的源码里有一个 selector：

```ts
const selector = (s: ReactFlowState) => ({
  isInteractive: s.nodesDraggable || s.nodesConnectable || s.elementsSelectable,
  minZoomReached: s.transform[2] <= s.minZoom,
  maxZoomReached: s.transform[2] >= s.maxZoom,
  ariaLabelConfig: s.ariaLabelConfig,
});
```

它说明 `Controls` 不是靠 props 得知当前 zoom 边界。

它直接从 store 读。

`Background` 的 selector 更轻：

```ts
const selector = (s: ReactFlowState) => ({
  transform: s.transform,
  patternId: `pattern-${s.rfId}`,
});
```

它只关心 transform 和 pattern id。

`MiniMap` 的 selector 最复杂：

```ts
const selector = (s: ReactFlowState) => {
  const viewBB = {
    x: -s.transform[0] / s.transform[2],
    y: -s.transform[1] / s.transform[2],
    width: s.width / s.transform[2],
    height: s.height / s.transform[2],
  };

  return {
    viewBB,
    boundingRect: ...,
    panZoom: s.panZoom,
    translateExtent: s.translateExtent,
    flowWidth: s.width,
    flowHeight: s.height,
  };
};
```

它把主画布 viewport 反算成 flow 坐标里的矩形。

这正是第 9 篇坐标系统在插件里的实际应用。

---

## 5. 源码调用链

把插件接入链路画出来，大概是这样：

```txt
<ReactFlow>
  <Background />
  <Controls />
  <MiniMap />
</ReactFlow>
  ↓
ReactFlow
  ↓
Wrapper
  ↓
ReactFlowProvider
  ↓
StoreUpdater
  ↓
GraphView
  ↓
children
    ↓
    Background / Controls / MiniMap
      ↓
      useStore / useStoreApi / useReactFlow
      ↓
      shared ReactFlow store
```

注意这里的 `children` 位置。

第 5 篇读 `ReactFlow` 主组件时，我们已经看到：

```txt
GraphView
SelectionListener
children
Attribution
A11yDescriptions
```

都在同一个 `Wrapper` 子树下。

这意味着 children 不只是普通 React children。

从运行时归属看，它们是：

```txt
拥有同一个 StoreContext 的插件区域
```

所以 `Controls` 可以调用：

```ts
const { zoomIn, zoomOut, fitView } = useReactFlow();
```

`Background` 可以调用：

```ts
const { transform, patternId } = useStore(selector, shallow);
```

`MiniMap` 可以调用：

```ts
const store = useStoreApi();
const { boundingRect, viewBB, panZoom } = useStore(selector, shallow);
```

这些 hook 调用没有任何特殊入口。

它们能工作，只因为 Provider 已经在上面。

这一点对我们实现 mini-flow 非常重要：

```txt
不要给插件组件开后门
让它们像用户组件一样使用 public hooks
```

这样 children 插件模型才会自然扩展。

---

## 6. 关键数据结构

第 23 篇的 store 还不够支持第 24 篇。

我们需要补几类字段和 helper。

### 6.1 viewport 相关字段

```ts
type MiniFlowState = {
  viewport: Viewport;
  width: number;
  height: number;
  minZoom: number;
  maxZoom: number;

  setViewport(viewport: Viewport): void;
};
```

`Controls` 至少需要知道：

```txt
当前 zoom
minZoom
maxZoom
```

`MiniMap` 需要知道：

```txt
主画布宽高
当前 viewport
```

如果没有 `width` 和 `height`，MiniMap 只能画节点范围，画不出当前视口范围。

### 6.2 interaction flags

React Flow 的 `Controls` 有一个锁定按钮。

它切换的是：

```txt
nodesDraggable
nodesConnectable
elementsSelectable
```

mini-flow 可以简化成：

```ts
type MiniFlowState = {
  nodesDraggable: boolean;
  nodesConnectable: boolean;
  elementsSelectable: boolean;

  setInteractive(interactive: boolean): void;
};
```

`isInteractive` 就可以派生为：

```ts
const isInteractive =
  state.nodesDraggable ||
  state.nodesConnectable ||
  state.elementsSelectable;
```

这个派生状态很适合放在 `Controls` 的 selector 里，而不是写成 store 字段。

### 6.3 bounds helper

MiniMap 需要把 nodes 变成一个全局矩形。

先写一个简化版：

```ts
type Rect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

function getNodesBounds(nodes: MiniNode[]): Rect {
  if (nodes.length === 0) {
    return { x: 0, y: 0, width: 0, height: 0 };
  }

  const rects = nodes.map((node) => ({
    x: node.position.x,
    y: node.position.y,
    width: node.width ?? 150,
    height: node.height ?? 40,
  }));

  const minX = Math.min(...rects.map((rect) => rect.x));
  const minY = Math.min(...rects.map((rect) => rect.y));
  const maxX = Math.max(...rects.map((rect) => rect.x + rect.width));
  const maxY = Math.max(...rects.map((rect) => rect.y + rect.height));

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}
```

React Flow 的真实源码用的是 system 层工具：

```txt
getInternalNodesBounds
getBoundsOfRects
```

原因是它处理的是 `InternalNode`，要考虑 measured、hidden、positionAbsolute、parent node 等情况。

mini-flow 暂时只处理普通节点。

但结构要对：

```txt
nodes
  -> nodes bounds
  -> bounds + viewBB
  -> minimap viewBox
```

### 6.4 viewBB：当前主视口在 flow 坐标里的矩形

MiniMap 不能直接用屏幕坐标画当前视口。

它要把主画布 viewport 反算成 flow 坐标。

```ts
function getViewBB(
  viewport: Viewport,
  width: number,
  height: number
): Rect {
  return {
    x: -viewport.x / viewport.zoom,
    y: -viewport.y / viewport.zoom,
    width: width / viewport.zoom,
    height: height / viewport.zoom,
  };
}
```

这和 React Flow MiniMap 源码里的公式一致：

```ts
const viewBB = {
  x: -s.transform[0] / s.transform[2],
  y: -s.transform[1] / s.transform[2],
  width: s.width / s.transform[2],
  height: s.height / s.transform[2],
};
```

这里再次说明：

```txt
viewport 不是 UI 细节
它决定所有插件如何理解当前画布
```

---

## 7. 关键实现思路

我们按从简单到复杂的顺序实现。

```txt
1. MiniPanel
2. MiniControls
3. MiniBackground
4. MiniMap
```

### 7.1 MiniPanel：插件浮层的共同底座

Panel 不需要读 store。

它只是提供位置 class。

```tsx
type PanelPosition =
  | "top-left"
  | "top-center"
  | "top-right"
  | "bottom-left"
  | "bottom-center"
  | "bottom-right";

type MiniPanelProps = {
  position?: PanelPosition;
  className?: string;
  style?: CSSProperties;
  children: ReactNode;
};

export function MiniPanel({
  position = "top-left",
  className,
  style,
  children,
}: MiniPanelProps) {
  const positionClasses = position.split("-");

  return (
    <div
      className={["mini-flow__panel", ...positionClasses, className]
        .filter(Boolean)
        .join(" ")}
      style={style}
    >
      {children}
    </div>
  );
}
```

对应 React Flow 的 `Panel`：

```tsx
<div className={cc(["react-flow__panel", className, ...positionClasses])}>
  {children}
</div>
```

它很薄。

但它很重要。

因为 Controls、MiniMap 这类插件都不应该自己重复处理定位。

这也是 additional components 的一个小规律：

```txt
复杂插件 = runtime hooks + Panel 布局
```

### 7.2 MiniControls：命令式操作 viewport

Controls 的实现重点不是按钮样式，而是它如何读写 runtime。

先扩展 `useMiniFlow()`：

```ts
function clamp(value: number, min: number, max: number) {
  return Math.min(max, Math.max(min, value));
}

export function useMiniFlow() {
  const store = useMiniStoreApi();

  return useMemo(
    () => ({
      getViewport: () => store.getState().viewport,

      setViewport: (viewport: Viewport) => {
        store.getState().setViewport(viewport);
      },

      zoomIn: () => {
        const state = store.getState();
        const zoom = clamp(
          state.viewport.zoom * 1.2,
          state.minZoom,
          state.maxZoom
        );

        state.setViewport({ ...state.viewport, zoom });
      },

      zoomOut: () => {
        const state = store.getState();
        const zoom = clamp(
          state.viewport.zoom / 1.2,
          state.minZoom,
          state.maxZoom
        );

        state.setViewport({ ...state.viewport, zoom });
      },

      fitView: () => {
        const state = store.getState();
        const bounds = getNodesBounds(state.nodes);
        const viewport = getViewportForBounds(
          bounds,
          state.width,
          state.height,
          state.minZoom,
          state.maxZoom,
          0.1
        );

        state.setViewport(viewport);
      },
    }),
    [store]
  );
}
```

再写 Controls：

```tsx
const controlsSelector = (state: MiniFlowState) => ({
  zoom: state.viewport.zoom,
  minZoom: state.minZoom,
  maxZoom: state.maxZoom,
  isInteractive:
    state.nodesDraggable ||
    state.nodesConnectable ||
    state.elementsSelectable,
});

export function MiniControls({
  position = "bottom-left",
}: {
  position?: PanelPosition;
}) {
  const store = useMiniStoreApi();
  const { zoom, minZoom, maxZoom, isInteractive } =
    useMiniStore(controlsSelector);
  const { zoomIn, zoomOut, fitView } = useMiniFlow();

  return (
    <MiniPanel position={position} className="mini-flow__controls">
      <button onClick={zoomIn} disabled={zoom >= maxZoom}>+</button>
      <button onClick={zoomOut} disabled={zoom <= minZoom}>-</button>
      <button onClick={fitView}>fit</button>
      <button
        onClick={() => {
          store.setState({
            nodesDraggable: !isInteractive,
            nodesConnectable: !isInteractive,
            elementsSelectable: !isInteractive,
          });
        }}
      >
        {isInteractive ? "unlock" : "lock"}
      </button>
    </MiniPanel>
  );
}
```

真实 React Flow 的 `Controls` 做法非常接近：

```txt
useStore(selector, shallow)
  读 isInteractive、minZoomReached、maxZoomReached

useReactFlow()
  拿 zoomIn、zoomOut、fitView

useStoreApi()
  切换 nodesDraggable、nodesConnectable、elementsSelectable

Panel
  定位控制面板
```

所以 Controls 是一个典型的“读写 store 插件”。

它既不是 GraphView 的一部分，也不是 ReactFlow 主组件的特殊分支。

它只是一个使用 hooks 的普通组件。

### 7.3 MiniBackground：让网格活在 flow 坐标系里

Background 最容易被写错。

错误版本通常是：

```tsx
function MiniBackground() {
  return <div className="grid-background" />;
}
```

这会得到一个固定在屏幕上的背景。

但节点编辑器里的网格，应该是跟随画布坐标系变化的。

也就是说：

```txt
pan 画布
  -> pattern offset 变化

zoom 画布
  -> gap 和 size 变化
```

mini-flow 可以用 SVG pattern 实现：

```tsx
type MiniBackgroundProps = {
  gap?: number;
  size?: number;
  color?: string;
};

export function MiniBackground({
  gap = 20,
  size = 1,
  color = "#d6d6d6",
}: MiniBackgroundProps) {
  const viewport = useViewport();
  const scaledGap = gap * viewport.zoom || 1;
  const scaledSize = size * viewport.zoom;
  const patternId = "mini-flow-pattern";

  return (
    <svg className="mini-flow__background">
      <pattern
        id={patternId}
        x={viewport.x % scaledGap}
        y={viewport.y % scaledGap}
        width={scaledGap}
        height={scaledGap}
        patternUnits="userSpaceOnUse"
      >
        <circle
          cx={scaledGap / 2}
          cy={scaledGap / 2}
          r={scaledSize}
          fill={color}
        />
      </pattern>
      <rect width="100%" height="100%" fill={`url(#${patternId})`} />
    </svg>
  );
}
```

React Flow 的 `Background` 更完整：

```txt
支持 dots / lines / cross
支持 gap 为 number 或 [x, y]
支持 size、lineWidth、offset
支持多个 Background 通过 id 避免 pattern 冲突
```

但关键公式一样：

```txt
scaledGap = gap * zoom
pattern x = transform.x % scaledGap
pattern y = transform.y % scaledGap
```

这里的重点是：

> Background 不是装饰背景，它是 viewport transform 的可视化辅助层。

如果读者理解了第 9 篇坐标系统，这段源码就很好理解。

如果没有理解 viewport，这段就会看起来像奇怪的 SVG 数学。

### 7.4 MiniMap：从 nodes 和 viewport 生成二级视图

MiniMap 比 Controls 和 Background 都复杂。

因为它不是一个按钮，也不是一个背景。

它是主画布的缩略视图。

实现 MiniMap 需要三步。

第一步：计算当前主视口在 flow 坐标里的矩形。

```ts
function getViewBB(state: MiniFlowState): Rect {
  const { viewport, width, height } = state;

  return {
    x: -viewport.x / viewport.zoom,
    y: -viewport.y / viewport.zoom,
    width: width / viewport.zoom,
    height: height / viewport.zoom,
  };
}
```

第二步：计算所有节点和当前视口的总 bounds。

```ts
function getBoundsOfRects(a: Rect, b: Rect): Rect {
  const minX = Math.min(a.x, b.x);
  const minY = Math.min(a.y, b.y);
  const maxX = Math.max(a.x + a.width, b.x + b.width);
  const maxY = Math.max(a.y + a.height, b.y + b.height);

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}
```

第三步：把 bounds 映射成 SVG viewBox。

```tsx
const minimapSelector = (state: MiniFlowState) => {
  const viewBB = getViewBB(state);
  const nodesBounds =
    state.nodes.length > 0 ? getNodesBounds(state.nodes) : viewBB;

  return {
    nodes: state.nodes,
    viewBB,
    boundingRect: getBoundsOfRects(nodesBounds, viewBB),
  };
};
```

然后组件渲染：

```tsx
export function MiniMap({
  width = 200,
  height = 150,
  position = "bottom-right",
}: {
  width?: number;
  height?: number;
  position?: PanelPosition;
}) {
  const { nodes, viewBB, boundingRect } = useMiniStore(minimapSelector);

  const scale = Math.max(
    boundingRect.width / width,
    boundingRect.height / height
  ) || 1;

  const viewWidth = width * scale;
  const viewHeight = height * scale;
  const x = boundingRect.x - (viewWidth - boundingRect.width) / 2;
  const y = boundingRect.y - (viewHeight - boundingRect.height) / 2;

  return (
    <MiniPanel position={position} className="mini-flow__minimap">
      <svg width={width} height={height} viewBox={`${x} ${y} ${viewWidth} ${viewHeight}`}>
        {nodes.map((node) => (
          <rect
            key={node.id}
            x={node.position.x}
            y={node.position.y}
            width={node.width ?? 150}
            height={node.height ?? 40}
            className="mini-flow__minimap-node"
          />
        ))}

        <path
          className="mini-flow__minimap-mask"
          d={`
            M${x},${y}h${viewWidth}v${viewHeight}h${-viewWidth}z
            M${viewBB.x},${viewBB.y}h${viewBB.width}v${viewBB.height}h${-viewBB.width}z
          `}
          fillRule="evenodd"
        />
      </svg>
    </MiniPanel>
  );
}
```

这里的 `path` 用了两个矩形：

```txt
外层矩形
当前 minimap 可见区域

内层矩形
主画布当前 viewport
```

配合 `fillRule="evenodd"`，就能画出一个遮罩：

```txt
小地图整体被 mask 覆盖
当前 viewport 区域被挖空或高亮
```

React Flow 的真实 MiniMap 也是这个结构。

它在 `MiniMap.tsx` 里渲染：

```txt
<MiniMapNodes />
<path className="react-flow__minimap-mask" d="..." fillRule="evenodd" />
```

然后 `MiniMapNodes` 只订阅 node ids，再把每个节点交给 `NodeComponentWrapper`。

这和 `NodeRenderer` 的性能思路一致：

```txt
外层订阅 ids
单节点 wrapper 订阅单节点数据
```

所以 MiniMap 虽然是插件，但它也继承了 React Flow 的性能设计。

---

## 8. 这部分源码的设计取舍

### 8.1 为什么插件用 children，而不是配置项？

如果 React Flow 把插件都做成 props，会变成：

```tsx
<ReactFlow
  showControls
  controlsPosition="bottom-left"
  showMiniMap
  miniMapNodeColor="red"
  showBackground
  backgroundGap={20}
/>
```

这在短期内很方便。

但它会让主组件 API 不断膨胀。

更麻烦的是，用户没法自然组合：

```tsx
<Controls>
  <ControlButton />
</Controls>
```

也没法把自定义插件放进同一个运行时：

```tsx
<MyAnalyticsPanel />
<MySelectionInspector />
<MyExportButton />
```

children 插件模式的收益是：

```txt
主组件保持稳定
插件可以独立演进
用户可以组合官方插件和自定义插件
所有插件共享同一个 Provider
```

代价是：

```txt
插件必须依赖 hooks 和 store
Provider 边界变得重要
错误使用时需要清晰报错
插件之间的层级和 z-index 要由 CSS 管理
```

这也是为什么第 23 篇先实现 store 和 Provider，第 24 篇再做插件。

如果没有 Provider，children 插件模式只是 JSX 语法糖。

有了 Provider，它才是 runtime 扩展机制。

### 8.2 为什么 Controls 不直接改 DOM transform？

Controls 点击 zoom in 时，最简单的做法是：

```ts
viewportElement.style.transform = ...
```

但这会绕开 store。

一旦绕开 store，就会出现：

```txt
GraphView 看起来 zoom 了
Background 不知道
MiniMap 不知道
useViewport 不知道
onViewportChange 不知道
```

React Flow 的 Controls 调用 `useReactFlow().zoomIn()`、`zoomOut()`、`fitView()`。

也就是说：

```txt
插件不要直接操作渲染结果
插件应该调用 runtime API
```

mini-flow 也要保持这个习惯。

`Controls` 只调用 `useMiniFlow()`，让 store 成为唯一状态来源。

### 8.3 为什么 Background 用 SVG pattern？

CSS background 当然也能画网格。

但 SVG pattern 有两个优势。

第一，它可以精确地跟随 transform 计算：

```txt
x = transform.x % scaledGap
y = transform.y % scaledGap
```

第二，它可以自然支持 dots、lines、cross 等变体。

React Flow 把 dots 和 line pattern 拆在：

```txt
Background/Patterns.tsx
```

这让 `Background.tsx` 专注于：

```txt
读 transform
算 scaled gap / size / offset
选择 pattern
渲染 rect fill=url(#pattern)
```

这里的取舍是：

```txt
SVG pattern 更可控，但实现比 CSS background 复杂
```

对一个节点编辑器库来说，这个复杂度是值得的。

因为背景网格是坐标系统的一部分，不只是页面装饰。

### 8.4 为什么 MiniMap 要接 system 层 XYMinimap？

如果 MiniMap 只负责展示，React 组件就够了。

但 React Flow 的 MiniMap 还支持：

```txt
pannable
zoomable
inversePan
zoomStep
```

这意味着用户可以在 MiniMap 上拖拽或滚轮，反过来控制主画布。

这就进入 interaction controller 的范围。

所以源码里 MiniMap 会在 `useEffect` 里创建：

```ts
XYMinimap({
  domNode: svg.current,
  panZoom,
  getTransform: () => store.getState().transform,
  getViewScale: () => viewScaleRef.current,
});
```

`XYMinimap` 在 system 层使用 `d3-zoom` 和 `d3-selection`，把 MiniMap 上的 wheel / pan 转成主画布的 `panZoom.scaleTo` 或 `panZoom.setViewportConstrained`。

这个分层很有代表性：

```txt
React MiniMap
  负责读 store、渲染 SVG、管理生命周期

system XYMinimap
  负责 DOM 事件和 pan/zoom 控制
```

mini-flow 的第一版可以只做展示。

如果要加 MiniMap 内拖拽，就应该模仿这个分层：

```txt
MiniMap component
  -> useEffect 创建 MiniMapController
  -> controller 处理 pointer / wheel
  -> controller 调用 useMiniFlow 或 store action 更新 viewport
```

不要把所有 pointer 逻辑塞进 JSX 里。

---

## 9. 如果我们自己实现，最小版本应该怎么写

这一节把四个插件合在一起。

### 9.1 Panel 与 Controls

```tsx
export function MiniControls({
  position = "bottom-left",
}: {
  position?: PanelPosition;
}) {
  const store = useMiniStoreApi();
  const { zoom, minZoom, maxZoom, isInteractive } =
    useMiniStore((state) => ({
      zoom: state.viewport.zoom,
      minZoom: state.minZoom,
      maxZoom: state.maxZoom,
      isInteractive:
        state.nodesDraggable ||
        state.nodesConnectable ||
        state.elementsSelectable,
    }));
  const { zoomIn, zoomOut, fitView } = useMiniFlow();

  return (
    <MiniPanel position={position} className="mini-flow__controls">
      <button type="button" onClick={zoomIn} disabled={zoom >= maxZoom}>
        +
      </button>
      <button type="button" onClick={zoomOut} disabled={zoom <= minZoom}>
        -
      </button>
      <button type="button" onClick={fitView}>
        fit
      </button>
      <button
        type="button"
        onClick={() =>
          store.setState({
            nodesDraggable: !isInteractive,
            nodesConnectable: !isInteractive,
            elementsSelectable: !isInteractive,
          })
        }
      >
        {isInteractive ? "unlock" : "lock"}
      </button>
    </MiniPanel>
  );
}
```

这段代码验证的是：

```txt
插件可以通过 useMiniFlow 调 runtime 命令
插件也可以通过 useMiniStoreApi 做高级状态更新
```

真实 React Flow 的 Controls 就是这个模式。

### 9.2 Background

```tsx
export function MiniBackground({
  id = "default",
  gap = 20,
  size = 1,
  color = "#d6d6d6",
}: MiniBackgroundProps) {
  const viewport = useViewport();
  const scaledGap = gap * viewport.zoom || 1;
  const scaledSize = size * viewport.zoom;
  const patternId = `mini-flow-pattern-${id}`;

  return (
    <svg className="mini-flow__background">
      <pattern
        id={patternId}
        x={viewport.x % scaledGap}
        y={viewport.y % scaledGap}
        width={scaledGap}
        height={scaledGap}
        patternUnits="userSpaceOnUse"
      >
        <circle
          cx={scaledGap / 2}
          cy={scaledGap / 2}
          r={scaledSize}
          fill={color}
        />
      </pattern>
      <rect x="0" y="0" width="100%" height="100%" fill={`url(#${patternId})`} />
    </svg>
  );
}
```

这段代码验证的是：

```txt
辅助渲染层也应该订阅 viewport
而不是直接依赖 DOM transform
```

### 9.3 MiniMap

```tsx
export function MiniMap({
  width = 200,
  height = 150,
  position = "bottom-right",
}: MiniMapProps) {
  const { nodes, viewBB, boundingRect } = useMiniStore((state) => {
    const viewBB = getViewBB(state);
    const nodesBounds =
      state.nodes.length > 0 ? getNodesBounds(state.nodes) : viewBB;

    return {
      nodes: state.nodes,
      viewBB,
      boundingRect: getBoundsOfRects(nodesBounds, viewBB),
    };
  });

  const scale = Math.max(
    boundingRect.width / width,
    boundingRect.height / height
  ) || 1;

  const viewWidth = width * scale;
  const viewHeight = height * scale;
  const x = boundingRect.x - (viewWidth - boundingRect.width) / 2;
  const y = boundingRect.y - (viewHeight - boundingRect.height) / 2;

  return (
    <MiniPanel position={position} className="mini-flow__minimap">
      <svg width={width} height={height} viewBox={`${x} ${y} ${viewWidth} ${viewHeight}`}>
        {nodes.map((node) => (
          <rect
            key={node.id}
            x={node.position.x}
            y={node.position.y}
            width={node.width ?? 150}
            height={node.height ?? 40}
            rx={4}
            ry={4}
            className="mini-flow__minimap-node"
          />
        ))}

        <path
          className="mini-flow__minimap-mask"
          d={`
            M${x},${y}h${viewWidth}v${viewHeight}h${-viewWidth}z
            M${viewBB.x},${viewBB.y}h${viewBB.width}v${viewBB.height}h${-viewBB.width}z
          `}
          fillRule="evenodd"
          pointerEvents="none"
        />
      </svg>
    </MiniPanel>
  );
}
```

这个版本还没有 pannable / zoomable。

但它已经验证了 MiniMap 的核心：

```txt
MiniMap = nodes bounds + current viewport bounds + SVG viewBox
```

如果要继续进阶，就补一个 controller：

```txt
MiniMapController
  pointer down
  pointer move
  把 minimap 坐标换成 flow 坐标
  更新主 viewport

wheel
  根据 zoomStep 计算 nextZoom
  调用 setViewport
```

这时就和 React Flow 的 `XYMinimap` 对齐了。

### 9.4 插件接入方式

最后，`MiniFlow` 不需要改太多。

第 23 篇已经写成：

```tsx
export function MiniFlow(props: MiniFlowProps) {
  return (
    <MiniFlowWrapper {...props}>
      <MiniStoreUpdater {...props} />
      <MiniGraphView />
      {props.children}
    </MiniFlowWrapper>
  );
}
```

第 24 篇只是在 `children` 里放插件：

```tsx
<MiniFlow defaultNodes={nodes} defaultEdges={edges}>
  <MiniBackground />
  <MiniControls />
  <MiniMap />
  <MiniPanel position="top-right">
    Custom panel
  </MiniPanel>
</MiniFlow>
```

这就是 children 插件模型的最小闭环。

```txt
MiniFlow 不识别插件
Provider 提供 runtime
插件通过 hooks 自己接入 runtime
```

---

## 10. 本篇总结

这一篇用 mini-flow 实现了四类插件：

```txt
Panel
  浮层定位底座

Controls
  读写 store 的命令式控制面板

Background
  订阅 viewport transform 的辅助渲染层

MiniMap
  基于 nodes + viewport 的二级视图
```

对应 React Flow 源码：

```txt
packages/react/src/additional-components/index.ts
packages/react/src/additional-components/Controls/Controls.tsx
packages/react/src/additional-components/Background/Background.tsx
packages/react/src/additional-components/MiniMap/MiniMap.tsx
packages/react/src/components/Panel/index.tsx
packages/system/src/xyminimap/index.ts
```

这一篇最重要的结论是：

> React Flow 的插件组件不是主组件里的硬编码分支，而是共享同一个 Provider / store / hooks 的普通 children。

这也是 React Flow 架构里很可复用的一点。

当你自己的系统也有“主运行时 + 多个扩展面板 / 工具 / 可视化辅助层”时，可以借鉴这个模式：

```txt
核心运行时放在 Provider
扩展组件通过 hooks 接入
主组件只负责提供边界
插件自己决定读什么、写什么、渲染什么
```

这样主组件不会被配置项压垮，插件也不需要特权。

---

## 11. 下一篇读什么

下一篇是整个系列的收束：

```txt
第 25 篇：总结：React Flow 源码的架构模式和可复用经验
```

我们会把前 24 篇的内容重新压缩成几条架构经验：

```txt
React Flow 的核心抽象是什么？
system / react 分层解决了什么问题？
store 为什么更像运行时控制面，而不是普通 React state？
viewport / transform 为什么是交互系统的地基？
renderer / interaction / plugin 为什么要分层？
哪些模式适合迁移到自己的项目？
哪些模式不该过度模仿？
```

到那一篇，mini-flow 不再继续加功能。

它会回到这套源码导读系列最开始的问题：

```txt
React Flow 的核心不是 React 组件，而是一套围绕 graph data、viewport 和 interaction-to-change pipeline 的系统。
```
