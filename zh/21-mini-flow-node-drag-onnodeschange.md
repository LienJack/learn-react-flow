---
title: "第 21 篇：实战：实现节点拖拽和 onNodesChange"
tags:
  - react-flow
  - xyflow
  - source-code
  - mini-flow
  - drag
  - changes
---

# 第 21 篇：实战：实现节点拖拽和 onNodesChange

第 20 篇解决的是“镜头怎么动”。

这一篇解决的是“图里的实体怎么动”。

这两个问题很像，但不能混在一起。

拖动画布时，我们改的是：

```txt
viewport.x
viewport.y
```

拖动节点时，我们改的是：

```txt
node.position
```

它们都会经过坐标转换。

但它们改变的对象完全不同。

这也是很多节点编辑器实现一开始会写偏的地方。最常见的误解是：

```txt
节点拖拽 = pointer move 时 setNodePosition
```

这个理解只对了一半。

React Flow 源码里，节点拖拽不是直接改 DOM，也不是简单把 `event.clientX - startX` 加到 `node.position` 上。

真正的链路是：

```txt
pointer / drag event
  ↓
转换到 flow 坐标
  ↓
构造 dragItems
  ↓
处理多选、snapGrid、nodeExtent
  ↓
更新内部 drag item position
  ↓
updateNodePositions
  ↓
生成 NodeChange
  ↓
triggerNodeChanges
  ↓
非受控内部 apply，受控交给 onNodesChange
```

这一篇的 mini-flow 实战也要保留这条主线。

不是为了把代码写复杂，而是为了验证我们真的读懂了 React Flow 的一个核心设计：

> 拖拽系统不应该直接拥有最终 nodes 状态，它应该产生 changes，由状态层决定怎么应用。

这就是 `onNodesChange` 的意义。

如果没有这层协议，拖拽会变成一个局部 UI 行为。

有了这层协议，拖拽、选择、删除、resize、键盘移动都可以统一变成：

```txt
changes -> applyChanges -> user callback
```

第 21 篇要实现的不是完整 `XYDrag`。

我们要实现它的最小骨架：

```txt
useNodeDrag
  ↓
pointer start / move / end
  ↓
screenToFlowPosition
  ↓
dragItems
  ↓
NodeChange[]
  ↓
applyNodeChanges / onNodesChange
```

这一篇会覆盖 roadmap 里的目标：

- 单节点拖拽。
- 多节点拖拽。
- snap grid。
- node extent。
- drag callbacks。
- `onNodesChange`。

输出：

- `NodeChange`。
- `applyNodeChanges`。
- `useNodeDrag`。
- `onNodeDragStart` / `onNodeDrag` / `onNodeDragStop`。

本章在前面文件树上新增 / 修改：

```txt
src/mini-flow/types.ts                 # NodeChange / NodeDragItem / callbacks
src/mini-flow/changes.ts               # applyNodeChanges
src/mini-flow/drag-utils.ts            # createDragItems / snap / clamp
src/mini-flow/useNodeDrag.ts           # pointer gesture -> NodeChange[]
src/mini-flow/MiniNodeRenderer.tsx     # 接入节点拖拽
src/mini-flow/MiniFlow.tsx             # onNodesChange / drag callbacks
```

本章验收 checklist：

```txt
- 拖动节点后，onNodesChange 收到 type='position' 的 change。
- pointer up 后会发送 dragging=false。
- snapGrid 不破坏多选节点相对距离。
- nodeExtent clamp 会考虑节点尺寸。
- applyNodeChanges 至少覆盖 position / select / remove。
```

和真实 React Flow 的差异：

| mini-flow 第 21 篇 | 真实 React Flow |
| --- | --- |
| pointer events 简化实现 | `useDrag + XYDrag + d3-drag` |
| selection 只作为简化输入 | 完整 selection 第 15 篇那套 click / box / multi / delete |
| 暂不做 auto pan | `XYDrag` 会和 panzoom 协作 |
| 暂不做 parent node / expandParent | 真实实现要处理 `positionAbsolute` 和 parent extent |
| 暂不做 drag handle / noDrag class | `NodeWrapper` 会注入节点级配置 |

---

## 1. 这一篇要解决的问题

第 20 篇已经有：

```ts
function screenToFlowPosition(point: XYPosition, viewport: Viewport): XYPosition {
  return {
    x: (point.x - viewport.x) / viewport.zoom,
    y: (point.y - viewport.y) / viewport.zoom,
  };
}
```

所以看起来，节点拖拽很容易：

```tsx
function onPointerMove(event) {
  const position = screenToFlowPosition(
    { x: event.clientX, y: event.clientY },
    viewport
  );

  setNodes(nodes =>
    nodes.map(node =>
      node.id === id ? { ...node, position } : node
    )
  );
}
```

这个版本的问题不少。

第一，它把鼠标位置当成节点左上角。

如果用户按住的是节点中间，拖拽一开始节点会跳一下。

正确做法是记录：

```txt
pointer 和 node.position 之间的 distance
```

第二，它只支持单节点。

真实节点编辑器里，选中多个节点后拖其中一个，所有选中节点都应该一起动。

第三，它没有 snap grid。

第四，它没有 node extent。

节点可能被限制在某个范围内。

第五，它直接 `setNodes`。

这会绕开 React Flow 的变化回流协议：

```txt
交互产生 NodeChange
用户决定如何应用
```

第六，它没有 drag callbacks。

用户希望拿到：

```txt
onNodeDragStart
onNodeDrag
onNodeDragStop
```

这些回调不应该从渲染层随便触发，而应该跟拖拽生命周期一致。

所以这一篇真正要解决的问题是：

```txt
如何把 pointer gesture 转成 node position changes？
```

这句话有三个关键词：

```txt
pointer gesture
node position
changes
```

React Flow 的源码也是围绕这三个词组织的。

---

## 2. 先看用户 API 或运行效果

这一篇完成后，用户可以这样使用：

```tsx
const [nodes, setNodes] = useState(initialNodes);

const onNodesChange = useCallback((changes: NodeChange[]) => {
  setNodes((currentNodes) => applyNodeChanges(changes, currentNodes));
}, []);

export function App() {
  return (
    <MiniFlow
      nodes={nodes}
      edges={edges}
      onNodesChange={onNodesChange}
      onNodeDragStart={(event, node) => {
        console.log('drag start', node.id);
      }}
      onNodeDrag={(event, node) => {
        console.log('dragging', node.position);
      }}
      onNodeDragStop={(event, node) => {
        console.log('drag stop', node.position);
      }}
      snapToGrid
      snapGrid={[20, 20]}
      nodeExtent={[
        [0, 0],
        [1000, 700],
      ]}
    />
  );
}
```

这里有一个重点：

`MiniFlow` 不直接要求用户传 `setNodes`。

它发出的是：

```ts
onNodesChange(changes)
```

用户可以选择：

```ts
setNodes((nodes) => applyNodeChanges(changes, nodes));
```

也可以选择写自己的逻辑：

```ts
const onNodesChange = (changes) => {
  analytics.track('nodes changed', changes);
  setNodes((nodes) => applyNodeChanges(changes, nodes));
};
```

这就是 React Flow 的受控思路。

交互系统只说：

```txt
发生了什么变化
```

最终状态由用户决定。

第 21 篇的运行效果是：

```txt
按住节点
  ↓
节点进入 dragging 状态
  ↓
移动鼠标
  ↓
节点位置按 flow 坐标更新
  ↓
多选节点一起移动
  ↓
snapGrid 可选生效
  ↓
nodeExtent 可选限制范围
  ↓
释放鼠标
  ↓
dragging 变 false
```

---

## 3. 核心概念解释

节点拖拽至少要分成四层。

```txt
Pointer Layer
  处理 pointerdown / pointermove / pointerup

Coordinate Layer
  把 screen / client 坐标转成 flow 坐标

Drag Runtime Layer
  维护 dragItems、distance、start position、snap、extent

Change Layer
  把拖拽结果转成 NodeChange，并触发 onNodesChange
```

如果少了 Coordinate Layer，就会写出：

```txt
client delta 直接加到 node.position
```

在 zoom 不是 1 时立刻错。

如果少了 Drag Runtime Layer，就会写出：

```txt
鼠标位置直接等于 node.position
```

节点会跳。

如果少了 Change Layer，就会写出：

```txt
拖拽内部直接 setNodes
```

受控模式会变得很难组织。

React Flow 的 `XYDrag` 就是 Drag Runtime Layer。

它不是 React 组件，也不是 store。

它是一个 interaction controller。

React 绑定层 `useDrag` 做的是：

```txt
拿 node DOM ref
创建 XYDrag
把 store.getState 注入 XYDrag
在 props 变化时调用 xyDrag.update(...)
```

证据在：

```txt
packages/react/src/hooks/useDrag.ts:22
packages/react/src/hooks/useDrag.ts:36
packages/react/src/hooks/useDrag.ts:59
```

system 层 `XYDrag` 接收的是：

```txt
getStoreItems
onDragStart
onDrag
onDragStop
```

证据在：

```txt
packages/system/src/xydrag/XYDrag.ts:77
packages/system/src/xydrag/XYDrag.ts:100
```

也就是说：

```txt
React 负责生命周期
system 负责拖拽算法
store 负责状态和 change 回流
```

第 21 篇 mini-flow 不引入完整 store。

但仍然保留这个分工：

```txt
MiniNodeView
  绑定 useNodeDrag

useNodeDrag
  处理 pointer 生命周期

MiniFlow
  保存当前 nodes / selected / viewport 运行时上下文

applyNodeChanges
  应用 NodeChange
```

---

## 4. 源码入口在哪里

这一篇主要看六组源码。

### 4.1 useDrag：React 绑定层

源码位置：

```txt
packages/react/src/hooks/useDrag.ts
```

`useDrag` 的核心是：

```txt
useEffect 创建 XYDrag
  ↓
把 store.getState 注入 getStoreItems
  ↓
返回 dragging 状态给 NodeWrapper
  ↓
另一个 effect 调用 xyDrag.update 绑定 DOM
```

关键证据：

```txt
packages/react/src/hooks/useDrag.ts:31
packages/react/src/hooks/useDrag.ts:36
packages/react/src/hooks/useDrag.ts:54
packages/react/src/hooks/useDrag.ts:59
```

这说明 React 包没有把拖拽算法写在组件里。

它把 DOM ref 和 store 能力交给 system controller。

### 4.2 XYDrag：拖拽控制器

源码位置：

```txt
packages/system/src/xydrag/XYDrag.ts
```

`XYDrag` 内部有几个重要运行时变量：

```txt
lastPos
dragItems
mousePosition
containerBounds
dragStarted
nodePositionsChanged
dragEvent
```

证据在：

```txt
packages/system/src/xydrag/XYDrag.ts:107
packages/system/src/xydrag/XYDrag.ts:109
packages/system/src/xydrag/XYDrag.ts:112
packages/system/src/xydrag/XYDrag.ts:113
packages/system/src/xydrag/XYDrag.ts:116
```

这些变量说明一件事：

```txt
拖拽不是一次性的事件回调，而是一段持续运行的交互会话。
```

第 21 篇的 `useNodeDrag` 也会维护一个 `dragState`。

### 4.3 getPointerPosition：把事件坐标转成 flow 坐标

源码位置：

```txt
packages/system/src/utils/dom.ts:11
```

它做：

```txt
event client position
  ↓
减去 containerBounds
  ↓
pointToRendererPoint(..., transform)
  ↓
snapPosition 可选
```

关键证据：

```txt
packages/system/src/utils/dom.ts:15
packages/system/src/utils/dom.ts:16
packages/system/src/utils/dom.ts:17
packages/system/src/utils/dom.ts:20
```

这就是第 20 篇 `screenToFlowPosition` 的应用场景。

### 4.4 getDragItems：收集要拖拽的节点

源码位置：

```txt
packages/system/src/xydrag/utils.ts:34
```

它会扫描 `nodeLookup`，把选中的节点和当前 node 转成 `NodeDragItem`。

每个 drag item 记录：

```txt
id
position
distance
extent
parentId
origin
internals.positionAbsolute
measured
```

证据在：

```txt
packages/system/src/xydrag/utils.ts:43
packages/system/src/xydrag/utils.ts:52
packages/system/src/xydrag/utils.ts:55
packages/system/src/xydrag/utils.ts:63
```

第 21 篇的 mini-flow 会简化为：

```txt
id
startPosition
distance
```

但保留核心思想：

```txt
拖拽开始时先冻结一份 dragItems 快照
```

### 4.5 updateNodes：拖拽过程中的位置计算

`XYDrag.update` 内部定义了 `updateNodes`。

它做：

```txt
读取 nodeLookup / snapGrid / nodeExtent / callbacks / updateNodePositions
  ↓
判断是否多选拖拽
  ↓
计算多选 snap offset
  ↓
遍历 dragItems
  ↓
nextPosition = pointer - distance
  ↓
snap / extent / calculateNodePosition
  ↓
更新 dragItem.position
  ↓
updateNodePositions(dragItems, true)
  ↓
触发 onNodeDrag / onSelectionDrag
```

关键证据：

```txt
packages/system/src/xydrag/XYDrag.ts:130
packages/system/src/xydrag/XYDrag.ts:146
packages/system/src/xydrag/XYDrag.ts:166
packages/system/src/xydrag/XYDrag.ts:192
packages/system/src/xydrag/XYDrag.ts:214
packages/system/src/xydrag/XYDrag.ts:223
```

这就是本篇 mini-flow 的核心算法。

### 4.6 updateNodePositions 和 triggerNodeChanges

React store 里的 `updateNodePositions` 在：

```txt
packages/react/src/store/index.ts:210
```

它把每个 drag item 转成：

```ts
{
  id,
  type: 'position',
  position,
  dragging
}
```

证据在：

```txt
packages/react/src/store/index.ts:220
packages/react/src/store/index.ts:229
```

最后调用：

```txt
triggerNodeChanges(changes)
```

证据在：

```txt
packages/react/src/store/index.ts:262
```

`triggerNodeChanges` 负责受控 / 非受控分流：

```txt
hasDefaultNodes
  -> applyNodeChanges(changes, nodes)
  -> setNodes(updatedNodes)

始终调用 onNodesChange?.(changes)
```

证据在：

```txt
packages/react/src/store/index.ts:264
packages/react/src/store/index.ts:268
packages/react/src/store/index.ts:277
```

这条链路是第 21 篇必须保留的设计骨架。

---

## 5. 源码调用链

真实 React Flow 的节点拖拽链路可以压缩成：

```txt
NodeWrapper
  ↓
useDrag(nodeRef, nodeId, dragHandle, noDragClassName)
  ↓
XYDrag({ getStoreItems: store.getState })
  ↓
xyDrag.update({ domNode, nodeId, handleSelector })
  ↓
d3-drag start
  ↓
getPointerPosition(event, transform, containerBounds)
  ↓
getDragItems(nodeLookup, selected nodes, pointerPos)
  ↓
d3-drag drag
  ↓
updateNodes(pointerPos)
  ↓
snap / extent / calculateNodePosition
  ↓
updateNodePositions(dragItems, true)
  ↓
NodeChange[]
  ↓
triggerNodeChanges(changes)
  ↓
onNodesChange / internal applyNodeChanges
  ↓
d3-drag end
  ↓
updateNodePositions(dragItems, false)
  ↓
onNodeDragStop
```

mini-flow 的第 21 篇压缩成：

```txt
MiniNodeView
  ↓
useNodeDrag(nodeId)
  ↓
pointerdown
  ↓
screenToFlowPosition(event)
  ↓
createDragItems(nodes, selectedIds, pointer)
  ↓
pointermove
  ↓
next position = pointer - distance
  ↓
snap / clamp to nodeExtent
  ↓
NodeChange[]
  ↓
onNodesChange(changes)
  ↓
用户 applyNodeChanges
  ↓
pointerup
  ↓
NodeChange dragging=false
  ↓
onNodeDragStop
```

这不是逐行复刻。

它复刻的是设计压力：

```txt
拖拽控制器处理 pointer 和临时 dragItems
状态层处理 nodes 和 changes
用户 API 接收 onNodesChange
```

---

## 6. 关键数据结构

### 6.1 NodeChange

第 21 篇先实现一种 change：

```ts
type NodePositionChange = {
  id: string;
  type: 'position';
  position: XYPosition;
  dragging?: boolean;
};

type NodeSelectionChange = {
  id: string;
  type: 'select';
  selected: boolean;
};

type NodeChange = NodePositionChange | NodeSelectionChange;
```

为什么要加 `select`？

因为多节点拖拽需要知道哪些节点被选中。

我们可以先把 selection 做得很小：

```txt
点击节点时 selected=true
按住 Shift 可多选
拖拽时如果节点 selected，则拖所有 selected nodes
```

但本篇重点不是完整选择系统。

这里只把 `selected` 当成多选拖拽的输入。

### 6.2 NodeDragItem

拖拽开始时，不能每次 pointermove 都从最新 nodes 里重新推导所有信息。

要冻结一份拖拽会话快照：

```ts
type NodeDragItem = {
  id: string;
  startPosition: XYPosition;
  distance: XYPosition;
  width: number;
  height: number;
};
```

`distance` 是关键：

```txt
pointer flow position - node.position
```

有了它，移动时才能算：

```txt
nextPosition = pointerFlowPosition - distance
```

这样节点不会跳。

真实 React Flow 的 `getDragItems` 也记录了 `distance`：

```txt
packages/system/src/xydrag/utils.ts:55
```

### 6.3 NodeExtent

node extent 是节点允许移动的范围：

```ts
type CoordinateExtent = [[number, number], [number, number]];
```

例如：

```ts
const nodeExtent: CoordinateExtent = [
  [0, 0],
  [1000, 700],
];
```

节点左上角不能小于 `[0, 0]`。

节点右下角不能超过 `[1000, 700]`。

所以 clamp position 时要考虑节点尺寸：

```txt
x <= extentMaxX - node.width
y <= extentMaxY - node.height
```

### 6.4 DragCallbacks

```ts
type NodeDragHandler = (
  event: PointerEvent | React.PointerEvent,
  node: MiniNode,
  nodes: MiniNode[]
) => void;
```

三个回调：

```txt
onNodeDragStart
onNodeDrag
onNodeDragStop
```

它们接收：

```txt
当前触发拖拽的 node
当前被拖拽的一组 nodes
```

这对应 `XYDrag` 里 `getEventHandlerParams` 的作用。

真实源码会从 dragItems 和 nodeLookup 里组装回调参数：

```txt
packages/system/src/xydrag/utils.ts:83
```

---

## 7. 关键实现思路

### 7.1 applyNodeChanges

先实现 change 应用器：

```ts
function applyNodeChanges(changes: NodeChange[], nodes: MiniNode[]): MiniNode[] {
  const changesById = new Map<string, NodeChange[]>();

  for (const change of changes) {
    const changes = changesById.get(change.id) ?? [];
    changes.push(change);
    changesById.set(change.id, changes);
  }

  return nodes.map((node) => {
    const changes = changesById.get(node.id);

    if (!changes) {
      return node;
    }

    let nextNode = { ...node };

    for (const change of changes) {
      if (change.type === 'position') {
        nextNode = {
          ...nextNode,
          position: change.position,
          dragging: change.dragging,
        };
      }

      if (change.type === 'select') {
        nextNode = {
          ...nextNode,
          selected: change.selected,
        };
      }
    }

    return nextNode;
  });
}
```

真实 React Flow 的 `applyNodeChanges` 调用内部 `applyChanges`。

`applyChanges` 会先按 id 建 change map，再逐个 element 应用。证据在：

```txt
packages/react/src/utils/changes.ts:19
packages/react/src/utils/changes.ts:25
packages/react/src/utils/changes.ts:53
packages/react/src/utils/changes.ts:82
```

`position` change 会写入 `element.position` 和 `element.dragging`：

```txt
packages/react/src/utils/changes.ts:114
packages/react/src/utils/changes.ts:119
```

### 7.2 triggerNodeChanges

mini-flow 可以先只做受控模式：

```ts
function triggerNodeChanges(changes: NodeChange[]) {
  onNodesChange?.(changes);
}
```

但为了让文章完整，可以给出非受控模式的形状：

```ts
function triggerNodeChanges(changes: NodeChange[]) {
  if (hasDefaultNodes) {
    setInternalNodes((nodes) => applyNodeChanges(changes, nodes));
  }

  onNodesChange?.(changes);
}
```

真实 React Flow 也是这个分流：

```txt
hasDefaultNodes -> internal applyNodeChanges + setNodes
始终调用 onNodesChange
```

这让拖拽系统不用关心用户当前是哪种模式。

### 7.3 createDragItems

```ts
function createDragItems(
  nodes: MiniNode[],
  nodeId: string,
  pointer: XYPosition
): Map<string, NodeDragItem> {
  const clickedNode = nodes.find((node) => node.id === nodeId);

  if (!clickedNode) {
    return new Map();
  }

  const shouldDragSelection = clickedNode.selected;
  const nodesToDrag = shouldDragSelection
    ? nodes.filter((node) => node.selected)
    : [clickedNode];

  const dragItems = new Map<string, NodeDragItem>();

  for (const node of nodesToDrag) {
    const size = getNodeSize(node);

    dragItems.set(node.id, {
      id: node.id,
      startPosition: node.position,
      distance: {
        x: pointer.x - node.position.x,
        y: pointer.y - node.position.y,
      },
      width: size.width,
      height: size.height,
    });
  }

  return dragItems;
}
```

这个函数解决两个问题：

```txt
节点不会跳
多选节点可以一起动
```

### 7.4 snapPosition

```ts
function snapPosition(position: XYPosition, snapGrid: [number, number]): XYPosition {
  return {
    x: snapGrid[0] * Math.round(position.x / snapGrid[0]),
    y: snapGrid[1] * Math.round(position.y / snapGrid[1]),
  };
}
```

真实 React Flow 也有同名工具：

```txt
packages/system/src/utils/general.ts:152
```

多选拖拽时，真实源码还有 `calculateSnapOffset`，避免每个节点分别吸附导致相对位置被破坏。

第 21 篇 mini 版本可以先做一个简单策略：

```txt
以当前被拖节点的 nextPosition 计算 snap delta
同一个 delta 应用到所有 dragItems
```

这样比每个节点各自 snap 更接近 React Flow 的意图。

### 7.5 clampNodePosition

```ts
function clampNodePosition(
  position: XYPosition,
  size: { width: number; height: number },
  extent?: CoordinateExtent
): XYPosition {
  if (!extent) {
    return position;
  }

  return {
    x: clamp(position.x, extent[0][0], extent[1][0] - size.width),
    y: clamp(position.y, extent[0][1], extent[1][1] - size.height),
  };
}
```

真实 React Flow 的 `calculateNodePosition` 会处理更多情况：

- parent node。
- node origin。
- node extent。
- parent-relative position。
- measured size。

证据在：

```txt
packages/system/src/utils/graph.ts:393
```

第 21 篇只做全局 extent。

### 7.6 updateDragItems

拖拽移动时，核心算法是：

```txt
pointer flow position
  ↓
next position = pointer - distance
  ↓
snap
  ↓
extent
  ↓
NodeChange[]
```

代码：

```ts
function getDragChanges({
  pointer,
  dragItems,
  activeNodeId,
  snapToGrid,
  snapGrid,
  nodeExtent,
}: {
  pointer: XYPosition;
  dragItems: Map<string, NodeDragItem>;
  activeNodeId: string;
  snapToGrid: boolean;
  snapGrid: [number, number];
  nodeExtent?: CoordinateExtent;
}): NodeChange[] {
  const activeItem = dragItems.get(activeNodeId);

  if (!activeItem) {
    return [];
  }

  const rawActivePosition = {
    x: pointer.x - activeItem.distance.x,
    y: pointer.y - activeItem.distance.y,
  };

  const snappedActivePosition = snapToGrid
    ? snapPosition(rawActivePosition, snapGrid)
    : rawActivePosition;

  const delta = {
    x: snappedActivePosition.x - activeItem.startPosition.x,
    y: snappedActivePosition.y - activeItem.startPosition.y,
  };

  const changes: NodeChange[] = [];

  for (const dragItem of dragItems.values()) {
    const nextPosition = {
      x: dragItem.startPosition.x + delta.x,
      y: dragItem.startPosition.y + delta.y,
    };

    const clampedPosition = clampNodePosition(
      nextPosition,
      { width: dragItem.width, height: dragItem.height },
      nodeExtent
    );

    changes.push({
      id: dragItem.id,
      type: 'position',
      position: clampedPosition,
      dragging: true,
    });
  }

  return changes;
}
```

这个版本比真实 `XYDrag.updateNodes` 少很多能力。

但它保留了三个关键点：

```txt
通过 distance 防止节点跳动
通过 active node 的 snap delta 保持多选相对位置
通过 NodeChange 输出结果
```

### 7.7 pointer end 时发 dragging=false

拖拽结束时，React Flow 会在位置变化过的情况下再次调用：

```txt
updateNodePositions(dragItems, false)
```

证据在：

```txt
packages/system/src/xydrag/XYDrag.ts:373
packages/system/src/xydrag/XYDrag.ts:377
```

这一步很重要。

拖拽中节点的 `dragging` 是 `true`。

结束后要把它改回 `false`。

mini-flow 可以这样写：

```ts
function getDragStopChanges(dragItems: Map<string, NodeDragItem>, nodes: MiniNode[]) {
  return Array.from(dragItems.keys()).map((id) => {
    const node = nodes.find((node) => node.id === id)!;

    return {
      id,
      type: 'position' as const,
      position: node.position,
      dragging: false,
    };
  });
}
```

---

## 8. 这部分源码的设计取舍

### 8.1 为什么拖拽不直接改 DOM

直接改 DOM 可以非常快：

```ts
nodeElement.style.transform = ...
```

但它绕过了图数据。

节点编辑器里，节点位置不仅影响视觉，还影响：

- edge path。
- handle position。
- selection bounds。
- MiniMap。
- undo / redo。
- 持久化。
- 外部业务状态。
- controlled mode。

所以 React Flow 必须让拖拽最终回到 `nodes` 数据。

DOM 只是渲染结果。

`NodeChange` 才是状态变化。

### 8.2 为什么不在 XYDrag 里 applyNodeChanges

`XYDrag` 位于 system 层。

它不应该知道 React store，也不应该知道用户是 controlled 还是 uncontrolled。

它只应该知道：

```txt
pointer 怎么变成 dragItems position
```

最终怎么应用，是 store 的责任。

这就是为什么真实链路里：

```txt
XYDrag
  -> updateNodePositions
  -> triggerNodeChanges
```

而不是：

```txt
XYDrag
  -> setNodes
```

这个边界让拖拽系统可以被 React / Svelte 共享，也让状态协议保持统一。

### 8.3 为什么多选拖拽需要 dragItems map

如果多选拖拽时每次都扫描 nodes 并直接加 delta，短期能工作。

但真实场景会遇到：

- 节点可能在拖拽中被删除。
- parent node 会影响绝对位置。
- snap grid 要保持相对位置。
- node extent 要考虑整体 bounds。
- 回调需要知道当前被拖的一组节点。

所以 React Flow 在拖拽开始时构造 `dragItems`。

它是拖拽会话中的工作集。

第 21 篇 mini-flow 也用 `Map<string, NodeDragItem>`，从一开始就保留这个形状。

### 8.4 为什么 snap 不能每个节点各自算

如果多选节点分别 snap：

```txt
node A -> 吸到 20 网格
node B -> 吸到 20 网格
```

它们之间的相对距离可能变化。

真实 React Flow 在多选拖拽时会计算 `multiDragSnapOffset`：

```txt
packages/system/src/xydrag/XYDrag.ts:146
packages/system/src/xydrag/XYDrag.ts:148
```

mini-flow 的简化策略是：

```txt
只按 active node 计算 snap delta
其他节点使用同一个 delta
```

这能保持相对位置。

### 8.5 为什么 node extent 要考虑尺寸

如果只 clamp 左上角：

```txt
x <= maxX
y <= maxY
```

节点右侧和底部会超出范围。

所以要：

```txt
x <= maxX - width
y <= maxY - height
```

真实 React Flow 的 `calculateNodePosition` 会用 measured size 处理约束。

mini-flow 用固定尺寸或用户传入尺寸。

这是第 19 篇以来一直保留的简化。

### 8.6 为什么 callbacks 和 onNodesChange 都要有

`onNodeDrag` 和 `onNodesChange` 不是一回事。

`onNodeDrag` 是交互生命周期回调：

```txt
用户正在拖哪个节点？
当前拖拽中的节点集合是什么？
```

`onNodesChange` 是状态变化回调：

```txt
哪些节点发生了什么变化？
```

一个面向行为观察。

一个面向状态回流。

真实 React Flow 两者都会触发。

`XYDrag.updateNodes` 会调用 `onNodeDrag`，同时 `updateNodePositions` 会走 `triggerNodeChanges`。

mini-flow 也保留这两个出口。

---

## 9. 如果我们自己实现，最小版本应该怎么写

下面是一份完整的实现草图。

它接在第 20 篇的 `MiniFlow` 上。

### 9.1 类型定义

```ts
type XYPosition = {
  x: number;
  y: number;
};

type CoordinateExtent = [[number, number], [number, number]];

type MiniNode<Data = { label?: string }> = {
  id: string;
  position: XYPosition;
  data?: Data;
  width?: number;
  height?: number;
  selected?: boolean;
  dragging?: boolean;
  hidden?: boolean;
};

type NodePositionChange = {
  id: string;
  type: 'position';
  position: XYPosition;
  dragging?: boolean;
};

type NodeSelectionChange = {
  id: string;
  type: 'select';
  selected: boolean;
};

type NodeChange = NodePositionChange | NodeSelectionChange;

type NodeDragItem = {
  id: string;
  startPosition: XYPosition;
  distance: XYPosition;
  width: number;
  height: number;
};

type NodeDragHandler = (
  event: PointerEvent | React.PointerEvent,
  node: MiniNode,
  nodes: MiniNode[]
) => void;
```

### 9.2 applyNodeChanges

```ts
export function applyNodeChanges(
  changes: NodeChange[],
  nodes: MiniNode[]
): MiniNode[] {
  const changesById = new Map<string, NodeChange[]>();

  for (const change of changes) {
    const itemChanges = changesById.get(change.id) ?? [];
    itemChanges.push(change);
    changesById.set(change.id, itemChanges);
  }

  return nodes.map((node) => {
    const itemChanges = changesById.get(node.id);

    if (!itemChanges) {
      return node;
    }

    let nextNode = { ...node };

    for (const change of itemChanges) {
      if (change.type === 'position') {
        nextNode = {
          ...nextNode,
          position: change.position,
          dragging: change.dragging,
        };
      }

      if (change.type === 'select') {
        nextNode = {
          ...nextNode,
          selected: change.selected,
        };
      }
    }

    return nextNode;
  });
}
```

### 9.3 drag 工具函数

```ts
function createDragItems(
  nodes: MiniNode[],
  nodeId: string,
  pointer: XYPosition
): Map<string, NodeDragItem> {
  const clickedNode = nodes.find((node) => node.id === nodeId);

  if (!clickedNode) {
    return new Map();
  }

  const nodesToDrag = clickedNode.selected
    ? nodes.filter((node) => node.selected && !node.hidden)
    : [clickedNode];

  return new Map(
    nodesToDrag.map((node) => {
      const size = getNodeSize(node);

      return [
        node.id,
        {
          id: node.id,
          startPosition: node.position,
          distance: {
            x: pointer.x - node.position.x,
            y: pointer.y - node.position.y,
          },
          width: size.width,
          height: size.height,
        },
      ];
    })
  );
}

function snapPosition(position: XYPosition, snapGrid: [number, number]) {
  return {
    x: snapGrid[0] * Math.round(position.x / snapGrid[0]),
    y: snapGrid[1] * Math.round(position.y / snapGrid[1]),
  };
}

function clampNodePosition(
  position: XYPosition,
  size: { width: number; height: number },
  extent?: CoordinateExtent
): XYPosition {
  if (!extent) {
    return position;
  }

  return {
    x: clamp(position.x, extent[0][0], extent[1][0] - size.width),
    y: clamp(position.y, extent[0][1], extent[1][1] - size.height),
  };
}
```

### 9.4 getDragChanges

```ts
function getDragChanges({
  pointer,
  dragItems,
  activeNodeId,
  snapToGrid,
  snapGrid,
  nodeExtent,
  dragging,
}: {
  pointer: XYPosition;
  dragItems: Map<string, NodeDragItem>;
  activeNodeId: string;
  snapToGrid: boolean;
  snapGrid: [number, number];
  nodeExtent?: CoordinateExtent;
  dragging: boolean;
}): NodeChange[] {
  const activeItem = dragItems.get(activeNodeId);

  if (!activeItem) {
    return [];
  }

  const rawActivePosition = {
    x: pointer.x - activeItem.distance.x,
    y: pointer.y - activeItem.distance.y,
  };

  const snappedActivePosition = snapToGrid
    ? snapPosition(rawActivePosition, snapGrid)
    : rawActivePosition;

  const delta = {
    x: snappedActivePosition.x - activeItem.startPosition.x,
    y: snappedActivePosition.y - activeItem.startPosition.y,
  };

  const changes: NodeChange[] = [];

  for (const dragItem of dragItems.values()) {
    const nextPosition = {
      x: dragItem.startPosition.x + delta.x,
      y: dragItem.startPosition.y + delta.y,
    };

    const position = clampNodePosition(
      nextPosition,
      { width: dragItem.width, height: dragItem.height },
      nodeExtent
    );

    changes.push({
      id: dragItem.id,
      type: 'position',
      position,
      dragging,
    });
  }

  return changes;
}
```

### 9.5 useNodeDrag

`useNodeDrag` 需要拿到运行时上下文：

```ts
type UseNodeDragParams = {
  nodeId: string;
  nodes: MiniNode[];
  viewport: Viewport;
  containerRef: React.RefObject<HTMLDivElement | null>;
  snapToGrid: boolean;
  snapGrid: [number, number];
  nodeExtent?: CoordinateExtent;
  onNodesChange?: (changes: NodeChange[]) => void;
  onNodeDragStart?: NodeDragHandler;
  onNodeDrag?: NodeDragHandler;
  onNodeDragStop?: NodeDragHandler;
};
```

实现：

```tsx
function useNodeDrag({
  nodeId,
  nodes,
  viewport,
  containerRef,
  snapToGrid,
  snapGrid,
  nodeExtent,
  onNodesChange,
  onNodeDragStart,
  onNodeDrag,
  onNodeDragStop,
}: UseNodeDragParams) {
  const dragStateRef = useRef<{
    pointerId: number;
    dragItems: Map<string, NodeDragItem>;
  } | null>(null);

  function getPointer(event: React.PointerEvent): XYPosition | null {
    const container = containerRef.current;

    if (!container) {
      return null;
    }

    const rect = container.getBoundingClientRect();
    const containerPoint = {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top,
    };

    return screenToFlowPosition(containerPoint, viewport);
  }

  function getDraggedNodes(changes: NodeChange[]) {
    return applyNodeChanges(changes, nodes).filter((node) =>
      changes.some((change) => change.id === node.id)
    );
  }

  function handlePointerDown(event: React.PointerEvent<HTMLDivElement>) {
    if (event.button !== 0) {
      return;
    }

    event.stopPropagation();
    event.currentTarget.setPointerCapture(event.pointerId);

    const pointer = getPointer(event);

    if (!pointer) {
      return;
    }

    const dragItems = createDragItems(nodes, nodeId, pointer);

    if (!dragItems.size) {
      return;
    }

    dragStateRef.current = {
      pointerId: event.pointerId,
      dragItems,
    };

    const changes = getDragChanges({
      pointer,
      dragItems,
      activeNodeId: nodeId,
      snapToGrid,
      snapGrid,
      nodeExtent,
      dragging: true,
    });

    onNodesChange?.(changes);

    const draggedNodes = getDraggedNodes(changes);
    const currentNode = draggedNodes.find((node) => node.id === nodeId) ?? draggedNodes[0];

    if (currentNode) {
      onNodeDragStart?.(event, currentNode, draggedNodes);
    }
  }

  function handlePointerMove(event: React.PointerEvent<HTMLDivElement>) {
    const dragState = dragStateRef.current;

    if (!dragState || dragState.pointerId !== event.pointerId) {
      return;
    }

    const pointer = getPointer(event);

    if (!pointer) {
      return;
    }

    const changes = getDragChanges({
      pointer,
      dragItems: dragState.dragItems,
      activeNodeId: nodeId,
      snapToGrid,
      snapGrid,
      nodeExtent,
      dragging: true,
    });

    if (!changes.length) {
      return;
    }

    onNodesChange?.(changes);

    const draggedNodes = getDraggedNodes(changes);
    const currentNode = draggedNodes.find((node) => node.id === nodeId) ?? draggedNodes[0];

    if (currentNode) {
      onNodeDrag?.(event, currentNode, draggedNodes);
    }
  }

  function handlePointerUp(event: React.PointerEvent<HTMLDivElement>) {
    const dragState = dragStateRef.current;

    if (!dragState || dragState.pointerId !== event.pointerId) {
      return;
    }

    const pointer = getPointer(event);

    if (pointer) {
      const changes = getDragChanges({
        pointer,
        dragItems: dragState.dragItems,
        activeNodeId: nodeId,
        snapToGrid,
        snapGrid,
        nodeExtent,
        dragging: false,
      });

      onNodesChange?.(changes);

      const draggedNodes = getDraggedNodes(changes);
      const currentNode = draggedNodes.find((node) => node.id === nodeId) ?? draggedNodes[0];

      if (currentNode) {
        onNodeDragStop?.(event, currentNode, draggedNodes);
      }
    }

    event.currentTarget.releasePointerCapture(event.pointerId);
    dragStateRef.current = null;
  }

  return {
    onPointerDown: handlePointerDown,
    onPointerMove: handlePointerMove,
    onPointerUp: handlePointerUp,
    onPointerCancel: handlePointerUp,
  };
}
```

这段代码还有简化。

它没有实现：

- drag threshold。
- auto pan。
- drag handle selector。
- noDragClassName。
- parent node。
- expandParent。
- connection.from 更新。

但它已经保留了这一篇最重要的骨架：

```txt
pointer -> flow position -> dragItems -> NodeChange -> onNodesChange
```

### 9.6 MiniNodeView 接入 useNodeDrag

```tsx
function MiniNodeView({
  node,
  dragHandlers,
}: {
  node: MiniNode;
  dragHandlers: ReturnType<typeof useNodeDrag>;
}) {
  const size = getNodeSize(node);

  return (
    <div
      className={node.dragging ? 'mini-flow__node dragging' : 'mini-flow__node'}
      style={{
        width: size.width,
        height: size.height,
        transform: `translate(${node.position.x}px, ${node.position.y}px)`,
      }}
      data-node-id={node.id}
      {...dragHandlers}
    >
      {node.data?.label ?? node.id}
    </div>
  );
}
```

真实 React Flow 的 `NodeWrapper` 也是节点 DOM 和拖拽系统接起来的地方。

它从 `useDrag` 拿到 dragging 状态，再把事件、className、style、节点组件组织在一起。

第 21 篇 mini-flow 把这一步简化成 `dragHandlers`。

### 9.7 MiniNodeRenderer 传入运行时上下文

```tsx
function MiniNodeRenderer({
  nodes,
  viewport,
  containerRef,
  snapToGrid,
  snapGrid,
  nodeExtent,
  onNodesChange,
  onNodeDragStart,
  onNodeDrag,
  onNodeDragStop,
}: MiniNodeRendererProps) {
  return (
    <div className="mini-flow__nodes">
      {nodes.map((node) => (
        <MiniNodeWrapper
          key={node.id}
          node={node}
          nodes={nodes}
          viewport={viewport}
          containerRef={containerRef}
          snapToGrid={snapToGrid}
          snapGrid={snapGrid}
          nodeExtent={nodeExtent}
          onNodesChange={onNodesChange}
          onNodeDragStart={onNodeDragStart}
          onNodeDrag={onNodeDrag}
          onNodeDragStop={onNodeDragStop}
        />
      ))}
    </div>
  );
}

function MiniNodeWrapper(props: MiniNodeWrapperProps) {
  const dragHandlers = useNodeDrag({
    nodeId: props.node.id,
    nodes: props.nodes,
    viewport: props.viewport,
    containerRef: props.containerRef,
    snapToGrid: props.snapToGrid,
    snapGrid: props.snapGrid,
    nodeExtent: props.nodeExtent,
    onNodesChange: props.onNodesChange,
    onNodeDragStart: props.onNodeDragStart,
    onNodeDrag: props.onNodeDrag,
    onNodeDragStop: props.onNodeDragStop,
  });

  return <MiniNodeView node={props.node} dragHandlers={dragHandlers} />;
}
```

这里的结构已经接近 React Flow 的性能方向：

```txt
NodeRenderer
  ↓
NodeWrapper
  ↓
useDrag
  ↓
NodeComponent
```

第 21 篇还没有 selector 和 store，所以 `nodes` 会传得比较多。

第 23 篇引入 store / hooks 后，再把它收敛。

### 9.8 MiniFlow props

```ts
type MiniFlowProps = {
  nodes: MiniNode[];
  edges: MiniEdge[];
  onNodesChange?: (changes: NodeChange[]) => void;
  onNodeDragStart?: NodeDragHandler;
  onNodeDrag?: NodeDragHandler;
  onNodeDragStop?: NodeDragHandler;
  snapToGrid?: boolean;
  snapGrid?: [number, number];
  nodeExtent?: CoordinateExtent;
};
```

`MiniFlow` 渲染 `MiniNodeRenderer` 时传入这些运行时参数即可。

重点不是 props 多。

重点是这些参数都有明确归属：

```txt
onNodesChange
  状态回流出口

onNodeDrag*
  拖拽生命周期回调

snapToGrid / snapGrid
  拖拽位置约束

nodeExtent
  节点移动范围约束
```

---

## 10. 本篇总结

这一篇实现了 mini-flow 的节点拖拽。

但真正重要的不是“节点可以被拖动”。

真正重要的是我们复刻了 React Flow 拖拽系统的核心分层：

```txt
pointer event
  ↓
screenToFlowPosition
  ↓
dragItems
  ↓
snap / extent
  ↓
NodeChange
  ↓
onNodesChange / applyNodeChanges
```

这条链路解释了几个关键设计：

```txt
1. 拖拽不是直接移动 DOM。
2. 拖拽也不是简单 setNodes。
3. pointer 坐标必须先进入 flow 坐标系。
4. dragItems 是拖拽会话的工作集。
5. 多选拖拽要保持相对位置。
6. snap 和 extent 属于拖拽位置计算层。
7. onNodeDrag 是生命周期回调，onNodesChange 是状态回流协议。
8. dragging=true / false 也应该通过 position change 回流。
```

React Flow 的真实实现更复杂：

- `d3-drag`。
- drag threshold。
- auto pan。
- parent node。
- expandParent。
- drag handle。
- noDrag class。
- connection in progress 更新。
- selected parent 过滤。
- store selector 和高频性能优化。

但这些复杂度都围绕同一条主线展开。

读懂第 21 篇的 mini-flow，回头再看 `XYDrag`，就不会只看到事件处理和一堆条件判断。

你会看到它在守住三件事：

```txt
坐标正确
拖拽工作集正确
change 回流正确
```

---

## 11. 下一篇读什么

下一篇进入：

```txt
第 22 篇：实战：实现 Handle、ConnectionLine 和 onConnect
```

节点拖拽解决的是：

```txt
移动已有实体
```

连接系统解决的是：

```txt
创建实体之间的关系
```

它的链路会变成：

```txt
pointer down on source handle
  ↓
创建 connection in progress
  ↓
pointer move
  ↓
更新 connection line
  ↓
寻找 target handle
  ↓
校验 connection
  ↓
pointer up
  ↓
onConnect / cancelConnection
```

对应 React Flow 源码：

```txt
Handle
  ↓
XYHandle
  ↓
ConnectionLine
  ↓
addEdge / onConnect
```

如果说 `XYDrag` 把 pointer gesture 转成 node position changes，`XYHandle` 就是把 pointer gesture 转成 graph relationship changes。
