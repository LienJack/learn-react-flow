---
title: "第 20 篇：实战：实现 viewport、pan、zoom、fitView"
tags:
  - react-flow
  - xyflow
  - source-code
  - mini-flow
  - viewport
  - panzoom
---

# 第 20 篇：实战：实现 viewport、pan、zoom、fitView

上一篇我们写了一个最小 Graph Renderer。

它能把：

```txt
nodes
edges
viewport
```

渲染成一个静态图画布。

但它还有一个明显的问题：

```txt
viewport 只是一个 prop。
```

也就是说，用户只能通过改代码告诉画布：

```tsx
viewport={{ x: 40, y: 20, zoom: 1 }}
```

画布自己不会动。

这一篇要让它动起来。

但先不要急着写 wheel handler。

这是很多人第一次写节点画布时最容易踩进去的坑：

```txt
把 pan / zoom 理解成一组 DOM 事件处理函数。
```

比如：

```tsx
function onWheel(event) {
  setZoom(zoom + event.deltaY);
}

function onMouseMove(event) {
  setX(x + event.movementX);
  setY(y + event.movementY);
}
```

这个版本可以让画布动。

但它没有解释图编辑器真正关心的问题：

```txt
鼠标所在的 screen point 对应哪个 flow point？
缩放时为什么鼠标下方的图内容不应该漂走？
拖动画布时 delta 应该除以 zoom 吗？
fitView 是怎么从节点 bounds 反推出 viewport 的？
screenToFlowPosition 和 flowToScreenPosition 为什么必须成对出现？
```

React Flow 源码里也没有把 pan / zoom 当成几个零散事件。

它的链路是：

```txt
ZoomPane
  ↓ 创建并更新
XYPanZoom
  ↓ 产生 transform
store.transform
  ↓
Viewport
  ↓
DOM / SVG layer
```

`XYPanZoom` 背后用了 `d3-zoom`，但这篇不会复刻 d3。

我们要复刻的是更本质的东西：

```txt
viewport 是一个镜头状态
pan / zoom / fitView 都是在求新的镜头状态
```

这一篇的局部公式可以写成：

```txt
Viewport Runtime
  = coordinate conversion
  + zoom around point
  + pan by screen delta
  + fit bounds into viewport
  + event adapter
```

最终我们会得到五个核心函数：

```txt
screenToFlowPosition
flowToScreenPosition
zoomAtPoint
panBy
fitView
```

这五个函数不是为了凑 API。

它们分别对应 React Flow 源码里的几类能力：

| mini-flow 函数 | React Flow 对应源码 |
| --- | --- |
| `screenToFlowPosition` | `useViewportHelper` + `pointToRendererPoint` |
| `flowToScreenPosition` | `useViewportHelper` + `rendererPointToPoint` |
| `zoomAtPoint` | `XYPanZoom` / d3 zoom 的缩放语义 |
| `panBy` | store action + system `panBy` |
| `fitView` | `getInternalNodesBounds` + `getViewportForBounds` |

这篇做完后，mini-flow 仍然不是完整节点编辑器。

但它会第一次拥有一个真正能操作的“镜头”。

本章在第 19 篇文件树上新增 / 修改：

```txt
src/mini-flow/types.ts                # 增加 Viewport / Rect / options
src/mini-flow/viewport-utils.ts       # 坐标转换、zoomAtPoint、fitView
src/mini-flow/MiniFlow.tsx            # 接入 viewport state 和事件
src/mini-flow/MiniViewport.tsx        # 继续只负责应用 transform
```

本章验收 checklist：

```txt
- pan 改 viewport.x / viewport.y，不改 node.position。
- zoomAtPoint 前后，鼠标下方的 flow point 基本保持不漂移。
- fitView 后，nodes bounds 能落在容器视口内。
- screenToFlowPosition / flowToScreenPosition 成对可逆。
```

和真实 React Flow 的差异：

| mini-flow 第 20 篇 | 真实 React Flow |
| --- | --- |
| 手写 pointer / wheel adapter | `ZoomPane + XYPanZoom + d3-zoom` |
| 只做 uncontrolled viewport | 支持 controlled `viewport` / `onViewportChange` |
| 暂不做动画和 translateExtent | 支持 duration、interpolate、extent clamp |
| 简化 DOM bounds 输入 | `useViewportHelper` 结合 wrapper DOM bounds |

还有一个不要混的点：drag pan 改 `viewport.x / viewport.y`，不需要除以 zoom；node drag 改 `node.position`，才需要把 screen/container delta 换算成 flow delta。

---

## 1. 这一篇要解决的问题

第 19 篇的渲染结构是：

```txt
MiniFlow
  ↓
MiniViewport
  ↓
MiniEdgeRenderer
MiniNodeRenderer
```

其中 `MiniViewport` 只做一件事：

```tsx
const transform = `translate(${viewport.x}px, ${viewport.y}px) scale(${viewport.zoom})`;
```

这证明了：

```txt
edges 和 nodes 应该共享同一个 viewport transform
```

但现在还没有回答：

```txt
这个 viewport 应该如何变化？
```

我们要解决五个问题。

第一个问题：

```txt
鼠标在屏幕上的位置，怎么变成画布里的 flow 坐标？
```

这对应：

```txt
screenToFlowPosition
```

第二个问题：

```txt
一个 flow 坐标，怎么知道它显示在屏幕哪里？
```

这对应：

```txt
flowToScreenPosition
```

第三个问题：

```txt
wheel zoom 时，为什么鼠标下的内容要保持不动？
```

这对应：

```txt
zoomAtPoint
```

第四个问题：

```txt
拖动画布时，移动的是节点，还是镜头？
```

这对应：

```txt
panBy
```

第五个问题：

```txt
fitView 如何把一组节点完整放进视口？
```

这对应：

```txt
fitView
```

这些问题都可以用同一组坐标关系回答。

如果写成一句话：

> pan、zoom、fitView 不是三套系统，它们都是在计算新的 viewport。

这就是本篇的主线。

---

## 2. 先看用户 API 或运行效果

第 20 篇完成后，用户仍然这样使用：

```tsx
<MiniFlow nodes={nodes} edges={edges} />
```

但画布会多出这些行为：

```txt
滚轮：围绕鼠标位置缩放
按住空白处拖动：平移画布
点击 fitView：把所有节点放进视口
调用 screenToFlowPosition：把鼠标位置换成 flow 坐标
调用 flowToScreenPosition：把 flow 坐标换回屏幕坐标
```

可以想象一个最小 demo：

```tsx
export function App() {
  return (
    <MiniFlow
      nodes={nodes}
      edges={edges}
      defaultViewport={{ x: 0, y: 0, zoom: 1 }}
      minZoom={0.25}
      maxZoom={2}
    />
  );
}
```

内部会维护 viewport state：

```tsx
const [viewport, setViewport] = useState(defaultViewport);
```

渲染层仍然和第 19 篇一样：

```tsx
<MiniViewport viewport={viewport}>
  <MiniEdgeRenderer edges={edges} nodeLookup={nodeLookup} />
  <MiniNodeRenderer nodes={nodes} />
</MiniViewport>
```

变化发生在外面的 interaction layer：

```tsx
<div
  className="mini-flow"
  ref={containerRef}
  onWheel={handleWheel}
  onPointerDown={handlePointerDown}
>
  ...
</div>
```

也就是说，第 20 篇不是重写 renderer。

它是在第 19 篇 renderer 外面补一层 viewport controller。

这和 React Flow 的真实结构是同一个方向：

```txt
FlowRenderer / ZoomPane 负责交互
Viewport 负责应用 transform
NodeRenderer / EdgeRenderer 负责渲染图元素
```

---

## 3. 核心概念解释

先把坐标公式写清楚。

第 19 篇里，viewport transform 是：

```txt
screenX = flowX * zoom + x
screenY = flowY * zoom + y
```

这里的 `screenX/screenY` 更准确地说，是相对于画布容器左上角的坐标。

如果要转换浏览器事件里的 `clientX/clientY`，还要先减掉容器的 DOM 位置：

```txt
containerX = clientX - containerRect.left
containerY = clientY - containerRect.top
```

所以完整链路是：

```txt
client position
  ↓ 减去 container bounds
container position
  ↓ 根据 viewport 反算
flow position
```

反过来则是：

```txt
flow position
  ↓ 应用 viewport
container position
  ↓ 加上 container bounds
client position
```

这个关系在 React Flow 源码里有两个工具函数。

`pointToRendererPoint` 做 screen/container 到 renderer/flow 的转换：

```txt
packages/system/src/utils/general.ts:159
```

核心公式是：

```txt
x: (x - tx) / tScale
y: (y - ty) / tScale
```

`rendererPointToPoint` 做 flow 到 screen/container 的转换：

```txt
packages/system/src/utils/general.ts:173
```

核心公式是：

```txt
x: x * tScale + tx
y: y * tScale + ty
```

第 20 篇 mini-flow 的坐标转换就照这个数学关系写。

注意这里有一个很重要的判断：

```txt
拖拽 viewport 时，delta 不需要除以 zoom。
拖拽 node 时，delta 才需要转换到 flow 坐标。
```

为什么？

因为拖动画布是在移动镜头。

用户鼠标在屏幕上移动 20px，镜头的 screen translation 就应该移动 20px：

```txt
viewport.x += 20
viewport.y += 0
```

但拖动节点是在改变节点的 flow 坐标。

如果当前 zoom 是 2，鼠标移动 20px，节点在 flow 世界里只移动 10：

```txt
node.position.x += 20 / 2
```

这就是为什么第 20 篇只处理 viewport，节点拖拽留到第 21 篇。

它们都和坐标转换有关，但改变的对象不同：

```txt
pan 改 viewport
drag 改 node position
```

---

## 4. 源码入口在哪里

第 20 篇对应的源码入口有五组。

### 4.1 ZoomPane：React 层接入 panZoom controller

看：

```txt
packages/react/src/container/ZoomPane/index.tsx:68
```

`ZoomPane` 在 mount 时创建 `XYPanZoom`：

```txt
XYPanZoom({
  domNode,
  minZoom,
  maxZoom,
  translateExtent,
  viewport,
  onPanZoom,
  onPanZoomStart,
  onPanZoomEnd
})
```

创建完成后，它会读取 `panZoom.current.getViewport()`，把：

```txt
panZoom
transform: [x, y, zoom]
domNode
```

写进 store。证据在：

```txt
packages/react/src/container/ZoomPane/index.tsx:95
packages/react/src/container/ZoomPane/index.tsx:97
```

然后在另一个 effect 里，`ZoomPane` 会把 pan / zoom 相关配置传给 `panZoom.update`：

```txt
packages/react/src/container/ZoomPane/index.tsx:109
```

这说明 React 层不是自己处理所有 wheel、drag、pinch 细节。

它负责：

```txt
创建 controller
传入 DOM 和配置
接收 viewport 变化
把结果同步进 store
```

第 20 篇的 mini-flow 可以不写 `XYPanZoom` 类。

但要保留这个边界：

```txt
事件层负责产生 viewport 更新
Viewport 层只负责应用 transform
```

### 4.2 XYPanZoom：真正的 viewport 控制器

看：

```txt
packages/system/src/xypanzoom/XYPanZoom.ts:36
```

`XYPanZoom` 接收 `domNode`、`minZoom`、`maxZoom`、`translateExtent`、`viewport` 和回调。

内部创建：

```txt
d3ZoomInstance = zoom().scaleExtent(...).translateExtent(...)
d3Selection = select(domNode).call(d3ZoomInstance)
```

证据在：

```txt
packages/system/src/xypanzoom/XYPanZoom.ts:56
packages/system/src/xypanzoom/XYPanZoom.ts:57
```

初始化时，它会调用 `setViewportConstrained`：

```txt
packages/system/src/xypanzoom/XYPanZoom.ts:60
```

这说明初始 viewport 也要经过约束。

第 20 篇先不实现 `translateExtent`，只实现 `minZoom/maxZoom`。

但会把约束写成独立函数：

```ts
function clampZoom(zoom: number, minZoom: number, maxZoom: number) {
  return Math.min(Math.max(zoom, minZoom), maxZoom);
}
```

### 4.3 useViewportHelper：公共坐标转换 API

看：

```txt
packages/react/src/hooks/useViewportHelper.ts:84
```

这里实现了 `screenToFlowPosition`。

它先拿 `domNode.getBoundingClientRect()`，把浏览器 client 坐标修正成容器坐标：

```txt
x: clientPosition.x - domX
y: clientPosition.y - domY
```

然后调用：

```txt
pointToRendererPoint(correctedPosition, transform, ...)
```

`flowToScreenPosition` 则反过来：

```txt
rendererPointToPoint(flowPosition, transform)
再加回 domX / domY
```

证据在：

```txt
packages/react/src/hooks/useViewportHelper.ts:94
packages/react/src/hooks/useViewportHelper.ts:102
packages/react/src/hooks/useViewportHelper.ts:104
packages/react/src/hooks/useViewportHelper.ts:112
```

这正是 mini-flow 要实现的两个函数。

### 4.4 panBy：平移 viewport 而不是节点

React store 里有 `panBy` action：

```txt
packages/react/src/store/index.ts:416
```

它把当前 `transform`、画布尺寸、`translateExtent` 和 `panZoom` 交给 system 层的 `panBy`。

system 层的 `panBy` 在：

```txt
packages/system/src/utils/store.ts:507
```

核心逻辑是：

```txt
x: transform[0] + delta.x
y: transform[1] + delta.y
zoom: transform[2]
```

也就是说，pan 改的是 viewport translation。

zoom 不变。

第 20 篇的 `panBy` 就实现这个最小语义。

### 4.5 fitView：从 bounds 反推 viewport

fitView 的源头有两处。

初始化时，`initialState` 会在 `fitView && width && height` 时计算初始 transform：

```txt
packages/react/src/store/initialState.ts:67
```

它先用 `getInternalNodesBounds` 算节点范围，再用 `getViewportForBounds` 算 viewport：

```txt
packages/react/src/store/initialState.ts:68
packages/react/src/store/initialState.ts:72
```

运行时，system 层的 `fitViewport` 也是同一条链：

```txt
packages/system/src/utils/graph.ts:354
```

它做：

```txt
nodes -> getInternalNodesBounds -> getViewportForBounds -> panZoom.setViewport
```

`getViewportForBounds` 的核心计算在：

```txt
packages/system/src/utils/general.ts:295
```

它根据 bounds、viewport 宽高、minZoom、maxZoom 和 padding，算出一个居中 viewport。

这正是第 20 篇要实现的 `fitView`。

---

## 5. 源码调用链

把真实 React Flow 的 viewport 链路压缩一下：

```txt
ReactFlow props
  ↓
GraphView
  ↓
FlowRenderer
  ↓
ZoomPane
  ↓
XYPanZoom(domNode, minZoom, maxZoom, translateExtent, viewport)
  ↓
d3-zoom 处理 wheel / drag / pinch / dblclick
  ↓
onTransformChange / onPanZoom
  ↓
store.transform = [x, y, zoom]
  ↓
Viewport style.transform = translate(x, y) scale(zoom)
  ↓
EdgeRenderer / NodeRenderer 跟着镜头变化
```

用户调用命令式 API 时，则是另一条链：

```txt
useReactFlow()
  ↓
useViewportHelper()
  ↓
zoomIn / zoomOut / zoomTo / setViewport / fitBounds
  ↓
panZoom.scaleBy / scaleTo / setViewport
  ↓
store.transform 更新
```

坐标转换 API 的链路是：

```txt
screenToFlowPosition(clientPosition)
  ↓
clientPosition - domNode.getBoundingClientRect()
  ↓
pointToRendererPoint(correctedPosition, transform)
  ↓
flow position
```

fitView 的链路是：

```txt
nodeLookup
  ↓
getInternalNodesBounds
  ↓
getViewportForBounds
  ↓
panZoom.setViewport
  ↓
store.transform
```

第 20 篇的 mini-flow 压缩成：

```txt
MiniFlow
  ↓
useState(viewport)
  ↓
onWheel -> zoomAtPoint -> setViewport
  ↓
onPointerDrag -> panBy -> setViewport
  ↓
fitView -> getNodesBounds -> getViewportForBounds -> setViewport
  ↓
MiniViewport style.transform
```

这个 mini 版本没有 d3、没有动画、没有 translateExtent、没有 controlled viewport。

但它保留了最重要的设计关系：

```txt
所有操作都收敛成新的 viewport
渲染层只消费 viewport
```

---

## 6. 关键数据结构

第 20 篇在第 19 篇的基础上补几个类型。

### 6.1 Viewport

```ts
type Viewport = {
  x: number;
  y: number;
  zoom: number;
};
```

`x` 和 `y` 是 screen/container translation。

`zoom` 是 flow 到 screen 的缩放比例。

它们共同决定：

```txt
screen = flow * zoom + translation
```

### 6.2 Rect

fitView 需要 bounds，所以要定义一个矩形：

```ts
type Rect = {
  x: number;
  y: number;
  width: number;
  height: number;
};
```

这个 `Rect` 处在 flow 坐标系中。

也就是说，节点 bounds 不是屏幕上的 DOM rect，而是图世界里的范围。

### 6.3 ViewportOptions

```ts
type ViewportOptions = {
  minZoom: number;
  maxZoom: number;
};
```

真实 React Flow 还有：

- `translateExtent`
- animation duration
- easing
- interpolation
- controlled viewport
- wheel / pinch / drag filter

第 20 篇先不写。

我们只保留 `minZoom/maxZoom`，因为它们直接影响 `zoomAtPoint` 和 `fitView`。

### 6.4 MiniFlowProps

第 19 篇的 props 是：

```ts
type MiniFlowProps = {
  nodes: MiniNode[];
  edges: MiniEdge[];
  viewport?: Viewport;
};
```

第 20 篇改成：

```ts
type MiniFlowProps = {
  nodes: MiniNode[];
  edges: MiniEdge[];
  defaultViewport?: Viewport;
  minZoom?: number;
  maxZoom?: number;
};
```

为什么不用 `viewport`？

因为这篇先实现非受控 viewport。

也就是：

```txt
MiniFlow 自己管理 viewport state
用户只给 defaultViewport
```

真实 React Flow 同时支持 controlled / uncontrolled viewport。

但第 20 篇如果同时写受控模式，文章会从坐标系统跑到状态协议。

controlled viewport 留到第 23 篇 store / hooks 部分更自然。

---

## 7. 关键实现思路

这一节先写纯函数，再写 React 事件接线。

### 7.1 clampZoom

```ts
function clamp(value: number, min: number, max: number) {
  return Math.min(Math.max(value, min), max);
}

function clampZoom(zoom: number, minZoom: number, maxZoom: number) {
  return clamp(zoom, minZoom, maxZoom);
}
```

真实 `XYPanZoom` 里初始化 viewport 时也会 clamp zoom：

```txt
packages/system/src/xypanzoom/XYPanZoom.ts:60
packages/system/src/xypanzoom/XYPanZoom.ts:64
```

第 20 篇只做 zoom 约束。

### 7.2 screenToFlowPosition

如果传入的是容器坐标：

```ts
function screenToFlowPosition(point: XYPosition, viewport: Viewport): XYPosition {
  return {
    x: (point.x - viewport.x) / viewport.zoom,
    y: (point.y - viewport.y) / viewport.zoom,
  };
}
```

如果传入的是浏览器事件坐标，就先修正：

```ts
function clientToContainerPosition(
  point: XYPosition,
  container: HTMLElement
): XYPosition {
  const rect = container.getBoundingClientRect();

  return {
    x: point.x - rect.left,
    y: point.y - rect.top,
  };
}
```

组合起来：

```ts
function clientToFlowPosition(
  point: XYPosition,
  viewport: Viewport,
  container: HTMLElement
): XYPosition {
  return screenToFlowPosition(
    clientToContainerPosition(point, container),
    viewport
  );
}
```

这里的数学和 React Flow 的 `pointToRendererPoint` 是同一个：

```txt
x = (x - tx) / scale
y = (y - ty) / scale
```

### 7.3 flowToScreenPosition

反向转换：

```ts
function flowToScreenPosition(point: XYPosition, viewport: Viewport): XYPosition {
  return {
    x: point.x * viewport.zoom + viewport.x,
    y: point.y * viewport.zoom + viewport.y,
  };
}
```

如果要得到浏览器 client 坐标：

```ts
function flowToClientPosition(
  point: XYPosition,
  viewport: Viewport,
  container: HTMLElement
): XYPosition {
  const rect = container.getBoundingClientRect();
  const screen = flowToScreenPosition(point, viewport);

  return {
    x: screen.x + rect.left,
    y: screen.y + rect.top,
  };
}
```

这和 React Flow 的 `rendererPointToPoint` 加 DOM rect 是同一个链路。

### 7.4 zoomAtPoint

这是这篇最重要的函数。

先说目标：

```txt
缩放后，鼠标指向的 flow point 仍然落在同一个 screen point。
```

假设鼠标在容器坐标：

```txt
point = { x: 300, y: 200 }
```

缩放前，它对应的 flow point 是：

```ts
const flowPoint = screenToFlowPosition(point, viewport);
```

缩放后，新的 zoom 是 `nextZoom`。

我们希望：

```txt
point.x = flowPoint.x * nextZoom + nextViewport.x
point.y = flowPoint.y * nextZoom + nextViewport.y
```

反过来求：

```txt
nextViewport.x = point.x - flowPoint.x * nextZoom
nextViewport.y = point.y - flowPoint.y * nextZoom
```

代码就是：

```ts
function zoomAtPoint(
  viewport: Viewport,
  point: XYPosition,
  nextZoom: number,
  options: ViewportOptions
): Viewport {
  const zoom = clampZoom(nextZoom, options.minZoom, options.maxZoom);
  const flowPoint = screenToFlowPosition(point, viewport);

  return {
    x: point.x - flowPoint.x * zoom,
    y: point.y - flowPoint.y * zoom,
    zoom,
  };
}
```

这就是“围绕鼠标缩放”的核心。

如果你只写：

```ts
return { ...viewport, zoom };
```

缩放中心会固定在画布原点。

用户看到的效果就是：滚轮一动，鼠标下方的节点漂走。

这不是视觉小问题。

它说明你没有把 zoom 理解成 viewport 变换。

### 7.5 panBy

pan 是最简单的：

```ts
function panBy(viewport: Viewport, delta: XYPosition): Viewport {
  return {
    x: viewport.x + delta.x,
    y: viewport.y + delta.y,
    zoom: viewport.zoom,
  };
}
```

注意 `delta` 是 screen/container delta。

所以不要除以 zoom。

这和 React Flow system 层 `panBy` 一致：

```txt
x = transform[0] + delta.x
y = transform[1] + delta.y
zoom = transform[2]
```

证据在：

```txt
packages/system/src/utils/store.ts:526
```

### 7.6 getNodesBounds

fitView 需要节点范围。

第 20 篇仍然使用固定节点尺寸：

```ts
const defaultNodeSize = {
  width: 150,
  height: 44,
};

function getNodeSize(node: MiniNode) {
  return {
    width: node.width ?? defaultNodeSize.width,
    height: node.height ?? defaultNodeSize.height,
  };
}
```

然后算所有可见节点的 bounds：

```ts
function getNodesBounds(nodes: MiniNode[]): Rect {
  const visibleNodes = nodes.filter((node) => !node.hidden);

  if (visibleNodes.length === 0) {
    return { x: 0, y: 0, width: 0, height: 0 };
  }

  let minX = Infinity;
  let minY = Infinity;
  let maxX = -Infinity;
  let maxY = -Infinity;

  for (const node of visibleNodes) {
    const size = getNodeSize(node);
    minX = Math.min(minX, node.position.x);
    minY = Math.min(minY, node.position.y);
    maxX = Math.max(maxX, node.position.x + size.width);
    maxY = Math.max(maxY, node.position.y + size.height);
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}
```

真实 React Flow 对应的是 `getInternalNodesBounds`。

它用 internal node 的 bounds 算整体矩形：

```txt
packages/system/src/utils/graph.ts:240
```

第 20 篇还没有 internal node，所以直接用用户 node。

但概念一样：

```txt
nodes -> bounds
```

### 7.7 getViewportForBounds

fitView 的核心是：

```txt
bounds -> viewport
```

给定：

```txt
bounds: flow 坐标中的节点范围
width/height: 容器尺寸
padding: 留白
minZoom/maxZoom: 缩放约束
```

求：

```txt
让 bounds 居中且完整可见的 viewport
```

最小实现：

```ts
function getViewportForBounds(
  bounds: Rect,
  width: number,
  height: number,
  minZoom: number,
  maxZoom: number,
  padding = 0.1
): Viewport {
  if (!bounds.width || !bounds.height || !width || !height) {
    return { x: 0, y: 0, zoom: 1 };
  }

  const paddedWidth = bounds.width * (1 + padding * 2);
  const paddedHeight = bounds.height * (1 + padding * 2);

  const xZoom = width / paddedWidth;
  const yZoom = height / paddedHeight;
  const zoom = clampZoom(Math.min(xZoom, yZoom), minZoom, maxZoom);

  const boundsCenterX = bounds.x + bounds.width / 2;
  const boundsCenterY = bounds.y + bounds.height / 2;

  return {
    x: width / 2 - boundsCenterX * zoom,
    y: height / 2 - boundsCenterY * zoom,
    zoom,
  };
}
```

React Flow 的真实 `getViewportForBounds` 更完整。

它支持多种 padding 写法，还会处理非对称 padding。证据在：

```txt
packages/system/src/utils/general.ts:295
packages/system/src/utils/general.ts:303
packages/system/src/utils/general.ts:318
```

但最小版本抓住了本质：

```txt
选一个 zoom，让 bounds 能放进 viewport
再选 x/y，让 bounds center 对齐 viewport center
```

### 7.8 fitView

有了前两个函数，fitView 就很薄：

```ts
function fitView(
  nodes: MiniNode[],
  viewportSize: { width: number; height: number },
  options: ViewportOptions & { padding?: number }
): Viewport {
  const bounds = getNodesBounds(nodes);

  return getViewportForBounds(
    bounds,
    viewportSize.width,
    viewportSize.height,
    options.minZoom,
    options.maxZoom,
    options.padding ?? 0.1
  );
}
```

真实 React Flow 的 `fitViewport` 也是：

```txt
getFitViewNodes
  ↓
getInternalNodesBounds
  ↓
getViewportForBounds
  ↓
panZoom.setViewport
```

证据在：

```txt
packages/system/src/utils/graph.ts:365
packages/system/src/utils/graph.ts:367
packages/system/src/utils/graph.ts:369
packages/system/src/utils/graph.ts:378
```

第 20 篇没有动画，所以直接返回 viewport。

---

## 8. 这部分源码的设计取舍

### 8.1 为什么 React Flow 不把 pan / zoom 写进 Viewport 组件

`Viewport` 组件的源码很小：

```txt
读取 transform
设置 style.transform
渲染 children
```

这不是偷懒。

这是边界清晰：

```txt
Viewport 负责应用镜头
XYPanZoom 负责改变镜头
```

如果把 wheel、drag、pinch、dblclick、controlled viewport 同步都塞进 `Viewport`，它会同时承担：

- DOM event controller
- transform state machine
- React renderer container
- d3 bridge
- user callback dispatcher

这样一来，渲染层和交互层就粘在一起了。

第 20 篇的 mini-flow 也保持这个边界。

`MiniViewport` 仍然只渲染 transform。

事件处理放在 `MiniFlow` 外层容器上。

### 8.2 为什么真实源码用 d3-zoom

这篇我们手写 pan / zoom。

但这不意味着 d3-zoom 没必要。

真实 React Flow 需要处理：

- wheel zoom
- trackpad pinch
- pan on drag
- pan on scroll
- double click zoom
- click distance
- event filter
- className 过滤
- touch 行为
- translate extent
- min/max scale extent
- animated transform
- controlled viewport sync

这些细节如果全部手写，会非常容易散落到 React 组件里。

所以 React Flow 把底层手势控制交给 d3-zoom，再在 `XYPanZoom` 外面包一层图编辑器语义。

`XYPanZoom` 暴露的 public functions 包括：

```txt
update
destroy
setViewport
setViewportConstrained
getViewport
scaleTo
scaleBy
syncViewport
setClickDistance
```

证据在：

```txt
packages/system/src/xypanzoom/XYPanZoom.ts:286
```

这个边界很值得学：

```txt
d3-zoom 解决输入设备和 transform mechanics
XYPanZoom 解决 React Flow 的 viewport 语义
React store 保存最终运行时状态
```

### 8.3 为什么 fitView 依赖节点尺寸

fitView 看起来只是：

```txt
把所有节点放进视口
```

但它其实依赖一个前提：

```txt
你知道节点有多大
```

如果只有 node.position，没有 width / height，fitView 只能把节点左上角放进视口，不能保证节点完整可见。

这也是为什么真实 React Flow 需要 `InternalNode` 和 DOM 测量。

第 20 篇用固定节点尺寸绕过测量。

但文章必须讲清楚：

```txt
这是为了聚焦 viewport 计算
不是因为真实图编辑器不需要测量
```

### 8.4 为什么 screenToFlowPosition 是公共 API

很多交互都需要这个函数：

- 点击画布创建节点。
- 拖拽外部元素到画布。
- 连接线跟随鼠标。
- 框选把屏幕矩形换算成 flow 矩形。
- 自定义 overlay 对齐图中元素。

如果用户只能拿到 `clientX/clientY`，就无法稳定地把外部交互接进图世界。

所以 React Flow 在 `useReactFlow` 实例上暴露 `screenToFlowPosition` 和 `flowToScreenPosition`。

第 20 篇虽然还没写 hooks，但先把这两个函数作为独立工具实现。

第 23 篇实现 store / hooks 时，再把它们挂到 `useMiniFlow` 上。

### 8.5 为什么 zoomAtPoint 不是简单改 zoom

这可能是本篇最容易被低估的设计点。

用户的直觉是：

```txt
滚轮向上 -> zoom 变大
滚轮向下 -> zoom 变小
```

但图编辑器的交互直觉其实是：

```txt
我指着哪里，就放大哪里
```

这要求 zoom 时同时改变：

```txt
zoom
x
y
```

如果只改 zoom，不改 x/y，缩放中心就是 flow 原点。

这会让画布像“从左上角吸附缩放”，体验很差。

`zoomAtPoint` 的价值就是维持这个不变量：

```txt
缩放前后，鼠标下方对应同一个 flow point。
```

这比事件本身更重要。

---

## 9. 如果我们自己实现，最小版本应该怎么写

下面是一份完整的 mini-flow viewport 实现草图。

它接在第 19 篇的 renderer 后面。

### 9.1 类型和工具函数

```ts
type XYPosition = {
  x: number;
  y: number;
};

type Viewport = {
  x: number;
  y: number;
  zoom: number;
};

type Rect = {
  x: number;
  y: number;
  width: number;
  height: number;
};

type ViewportOptions = {
  minZoom: number;
  maxZoom: number;
};

function clamp(value: number, min: number, max: number) {
  return Math.min(Math.max(value, min), max);
}

function clampZoom(zoom: number, minZoom: number, maxZoom: number) {
  return clamp(zoom, minZoom, maxZoom);
}

function screenToFlowPosition(point: XYPosition, viewport: Viewport): XYPosition {
  return {
    x: (point.x - viewport.x) / viewport.zoom,
    y: (point.y - viewport.y) / viewport.zoom,
  };
}

function flowToScreenPosition(point: XYPosition, viewport: Viewport): XYPosition {
  return {
    x: point.x * viewport.zoom + viewport.x,
    y: point.y * viewport.zoom + viewport.y,
  };
}

function zoomAtPoint(
  viewport: Viewport,
  point: XYPosition,
  nextZoom: number,
  options: ViewportOptions
): Viewport {
  const zoom = clampZoom(nextZoom, options.minZoom, options.maxZoom);
  const flowPoint = screenToFlowPosition(point, viewport);

  return {
    x: point.x - flowPoint.x * zoom,
    y: point.y - flowPoint.y * zoom,
    zoom,
  };
}

function panBy(viewport: Viewport, delta: XYPosition): Viewport {
  return {
    x: viewport.x + delta.x,
    y: viewport.y + delta.y,
    zoom: viewport.zoom,
  };
}
```

这组函数就是 viewport runtime 的数学核心。

它们不依赖 React。

也不依赖 DOM。

只有 `client -> container` 的修正需要 DOM。

### 9.2 DOM 坐标修正

```ts
function clientToContainerPosition(
  point: XYPosition,
  container: HTMLElement
): XYPosition {
  const rect = container.getBoundingClientRect();

  return {
    x: point.x - rect.left,
    y: point.y - rect.top,
  };
}
```

使用时：

```ts
const point = clientToContainerPosition(
  { x: event.clientX, y: event.clientY },
  container
);
```

这一步对应 React Flow `screenToFlowPosition` 里减掉 `domNode.getBoundingClientRect()` 的逻辑。

### 9.3 bounds 和 fitView

```ts
const defaultNodeSize = {
  width: 150,
  height: 44,
};

function getNodeSize(node: MiniNode) {
  return {
    width: node.width ?? defaultNodeSize.width,
    height: node.height ?? defaultNodeSize.height,
  };
}

function getNodesBounds(nodes: MiniNode[]): Rect {
  const visibleNodes = nodes.filter((node) => !node.hidden);

  if (visibleNodes.length === 0) {
    return { x: 0, y: 0, width: 0, height: 0 };
  }

  let minX = Infinity;
  let minY = Infinity;
  let maxX = -Infinity;
  let maxY = -Infinity;

  for (const node of visibleNodes) {
    const size = getNodeSize(node);
    minX = Math.min(minX, node.position.x);
    minY = Math.min(minY, node.position.y);
    maxX = Math.max(maxX, node.position.x + size.width);
    maxY = Math.max(maxY, node.position.y + size.height);
  }

  return {
    x: minX,
    y: minY,
    width: maxX - minX,
    height: maxY - minY,
  };
}

function getViewportForBounds(
  bounds: Rect,
  width: number,
  height: number,
  minZoom: number,
  maxZoom: number,
  padding = 0.1
): Viewport {
  if (!bounds.width || !bounds.height || !width || !height) {
    return { x: 0, y: 0, zoom: 1 };
  }

  const paddedWidth = bounds.width * (1 + padding * 2);
  const paddedHeight = bounds.height * (1 + padding * 2);

  const zoom = clampZoom(
    Math.min(width / paddedWidth, height / paddedHeight),
    minZoom,
    maxZoom
  );

  const boundsCenterX = bounds.x + bounds.width / 2;
  const boundsCenterY = bounds.y + bounds.height / 2;

  return {
    x: width / 2 - boundsCenterX * zoom,
    y: height / 2 - boundsCenterY * zoom,
    zoom,
  };
}

function fitView(
  nodes: MiniNode[],
  viewportSize: { width: number; height: number },
  options: ViewportOptions & { padding?: number }
): Viewport {
  return getViewportForBounds(
    getNodesBounds(nodes),
    viewportSize.width,
    viewportSize.height,
    options.minZoom,
    options.maxZoom,
    options.padding ?? 0.1
  );
}
```

这里的 `fitView` 没有动画。

真实 React Flow 会调用 `panZoom.setViewport(viewport, { duration, ease, interpolate })`。

第 20 篇先只关心最终 viewport。

### 9.4 MiniFlow 组件接线

```tsx
import { useMemo, useRef, useState } from 'react';

type MiniFlowProps = {
  nodes: MiniNode[];
  edges: MiniEdge[];
  defaultViewport?: Viewport;
  minZoom?: number;
  maxZoom?: number;
};

const initialViewport: Viewport = {
  x: 0,
  y: 0,
  zoom: 1,
};

export function MiniFlow({
  nodes,
  edges,
  defaultViewport = initialViewport,
  minZoom = 0.25,
  maxZoom = 2,
}: MiniFlowProps) {
  const containerRef = useRef<HTMLDivElement | null>(null);
  const lastPanPoint = useRef<XYPosition | null>(null);
  const [viewport, setViewport] = useState(defaultViewport);

  const nodeLookup = useMemo(() => createNodeLookup(nodes), [nodes]);
  const viewportOptions = { minZoom, maxZoom };

  function handleWheel(event: React.WheelEvent<HTMLDivElement>) {
    const container = containerRef.current;

    if (!container) {
      return;
    }

    event.preventDefault();

    const point = clientToContainerPosition(
      { x: event.clientX, y: event.clientY },
      container
    );

    const zoomFactor = Math.exp(-event.deltaY * 0.001);

    setViewport((current) =>
      zoomAtPoint(
        current,
        point,
        current.zoom * zoomFactor,
        viewportOptions
      )
    );
  }

  function handlePointerDown(event: React.PointerEvent<HTMLDivElement>) {
    if (event.button !== 0) {
      return;
    }

    event.currentTarget.setPointerCapture(event.pointerId);
    lastPanPoint.current = { x: event.clientX, y: event.clientY };
  }

  function handlePointerMove(event: React.PointerEvent<HTMLDivElement>) {
    const lastPoint = lastPanPoint.current;

    if (!lastPoint) {
      return;
    }

    const nextPoint = { x: event.clientX, y: event.clientY };
    const delta = {
      x: nextPoint.x - lastPoint.x,
      y: nextPoint.y - lastPoint.y,
    };

    lastPanPoint.current = nextPoint;
    setViewport((current) => panBy(current, delta));
  }

  function handlePointerUp(event: React.PointerEvent<HTMLDivElement>) {
    event.currentTarget.releasePointerCapture(event.pointerId);
    lastPanPoint.current = null;
  }

  function handleFitView() {
    const container = containerRef.current;

    if (!container) {
      return;
    }

    const rect = container.getBoundingClientRect();

    setViewport(
      fitView(nodes, { width: rect.width, height: rect.height }, {
        minZoom,
        maxZoom,
        padding: 0.15,
      })
    );
  }

  return (
    <div className="mini-flow-shell">
      <div className="mini-flow-toolbar">
        <button type="button" onClick={handleFitView}>
          Fit
        </button>
      </div>

      <div
        ref={containerRef}
        className="mini-flow"
        onWheel={handleWheel}
        onPointerDown={handlePointerDown}
        onPointerMove={handlePointerMove}
        onPointerUp={handlePointerUp}
        onPointerCancel={handlePointerUp}
      >
        <MiniViewport viewport={viewport}>
          <MiniEdgeRenderer edges={edges} nodeLookup={nodeLookup} />
          <MiniNodeRenderer nodes={nodes} />
        </MiniViewport>
      </div>
    </div>
  );
}
```

这段代码有几个故意简化的地方。

第一，空白处和节点处都会触发 pan。

真实 React Flow 会通过 className、事件 filter、selection 状态、connection 状态判断能不能 pan。

第二，没有处理 trackpad pinch、double click zoom、pan on scroll。

这些属于 `XYPanZoom.update` 里的复杂配置。

第三，没有 controlled viewport。

第 20 篇只做 uncontrolled viewport。

第四，没有动画。

真实 React Flow 会把 duration/ease/interpolate 传给 `panZoom.setViewport`。

这里直接 set state。

这些简化都不影响本篇目标：

```txt
用最小代码验证 viewport 数学和 interaction adapter 的边界。
```

### 9.5 MiniViewport 仍然很小

```tsx
function MiniViewport({
  viewport,
  children,
}: {
  viewport: Viewport;
  children: React.ReactNode;
}) {
  const transform = `translate(${viewport.x}px, ${viewport.y}px) scale(${viewport.zoom})`;

  return (
    <div className="mini-flow__viewport" style={{ transform }}>
      {children}
    </div>
  );
}
```

这正是我们想要的结果。

viewport 变复杂了。

但 `MiniViewport` 没变复杂。

这说明边界是稳的。

---

## 10. 本篇总结

第 20 篇把第 19 篇的静态 viewport 变成了一个可操作的 viewport runtime。

这篇最重要的结论不是“我们写了 wheel zoom 和 drag pan”。

真正重要的是：

```txt
pan、zoom、fitView 都是在求新的 viewport。
```

具体来说：

```txt
screenToFlowPosition
  把容器坐标反算成 flow 坐标

flowToScreenPosition
  把 flow 坐标应用 viewport 得到容器坐标

zoomAtPoint
  在改变 zoom 的同时调整 x/y，保持鼠标下方 flow point 不漂移

panBy
  根据 screen delta 改 viewport.x / viewport.y

fitView
  从 nodes bounds 反推出能容纳它们的 viewport
```

这些函数对应 React Flow 源码里的：

```txt
pointToRendererPoint
rendererPointToPoint
XYPanZoom
panBy
getInternalNodesBounds
getViewportForBounds
fitViewport
```

第 20 篇也解释了一个关键边界：

```txt
Viewport 负责应用 transform
Viewport controller 负责产生 transform
Renderer 只消费 transform
```

这个边界一旦站稳，后面实现节点拖拽就很清楚了。

拖动画布时：

```txt
改 viewport
```

拖动节点时：

```txt
改 node.position
```

两者都用坐标转换，但改变的对象完全不同。

这就是下一篇的入口。

---

## 11. 下一篇读什么

下一篇进入：

```txt
第 21 篇：实战：实现节点拖拽和 onNodesChange
```

现在我们已经能把 screen 坐标转成 flow 坐标。

节点拖拽就可以基于这个能力继续往下走：

```txt
pointer down
  ↓
记录起点 flow position
  ↓
pointer move
  ↓
把 screen movement 转成 flow delta
  ↓
更新 node.position
  ↓
产生 NodeChange
  ↓
触发 onNodesChange
```

对应回 React Flow 源码，就是：

```txt
XYDrag
  ↓
getPointerPosition
  ↓
calculateNodePosition
  ↓
updateNodePositions
  ↓
triggerNodeChanges
```

也就是说，第 20 篇解决的是“镜头怎么动”。

第 21 篇解决的是“图里的实体怎么动”。
