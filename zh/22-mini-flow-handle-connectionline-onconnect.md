---
title: "第 22 篇：实战：实现 Handle、ConnectionLine 和 onConnect"
tags:
  - react-flow
  - xyflow
  - source-code
  - mini-flow
  - handle
  - connection
---

# 第 22 篇：实战：实现 Handle、ConnectionLine 和 onConnect

第 21 篇解决的是节点怎么动。

这一篇解决的是节点之间的关系怎么被创建。

这两个问题看起来都叫“拖拽”，但本质完全不同。

节点拖拽的结果是：

```txt
node.position 变化
```

连线拖拽的结果是：

```txt
source node / handle
target node / handle
形成一个 Connection
```

很多人第一次实现节点编辑器时，会把连接系统写成：

```txt
鼠标从一个节点拖到另一个节点
  -> 直接创建 edge
```

这个理解太粗了。

React Flow 里的连接系统真正处理的是一段中间态：

```txt
用户按下 source handle
  ↓
connection in progress
  ↓
鼠标移动时渲染临时 ConnectionLine
  ↓
持续寻找 target handle
  ↓
持续校验连接是否合法
  ↓
pointer up 时，如果合法，触发 onConnect(Connection)
  ↓
用户决定是否 addEdge
```

也就是说：

> 连接过程不是 EdgeRenderer 的工作。EdgeRenderer 只渲染已经存在的 edges。连接过程由 Handle、XYHandle、connection state 和 ConnectionLine 协作完成。

这一篇的 mini-flow 实战就要复刻这条链路：

```txt
<Handle />
  ↓
useConnectionDrag
  ↓
connection state
  ↓
<ConnectionLine />
  ↓
onConnect(Connection)
  ↓
addEdge(connection, edges)
```

这里有一个非常重要的边界：

```txt
Connection 不是 Edge。
```

`Connection` 只描述：

```txt
source
sourceHandle
target
targetHandle
```

它没有 edge id，没有 type，没有 style，没有 marker，没有 label。

真实 React Flow 的 `Connection` 类型也是这四个字段。源码里还明确说明，`addEdge` 可以把 `Connection` 升级成 `Edge`。证据在：

```txt
packages/system/src/types/general.ts:68
```

所以这一篇的主线不是“画一条线”。

而是：

```txt
一次 pointer gesture 如何变成一个合法 Connection？
```

本章在前面文件树上新增 / 修改：

```txt
src/mini-flow/types.ts                  # Handle / Connection / ConnectionState
src/mini-flow/connection-utils.ts       # addEdge / find handle / validate
src/mini-flow/useConnectionDrag.ts      # pointer gesture -> Connection
src/mini-flow/Handle.tsx
src/mini-flow/ConnectionLine.tsx
src/mini-flow/MiniNodeRenderer.tsx      # 渲染 source / target handles
src/mini-flow/MiniFlow.tsx              # connection state / onConnect
```

本章验收 checklist：

```txt
- pointer down on Handle 后出现 ConnectionLine。
- pointer up 到合法 target handle 才触发 onConnect。
- 无效连接不会新增 edge。
- onConnect 收到的是 Connection，不是 Edge。
- addEdge 负责补 edge id，并做最小去重。
```

和真实 React Flow 的差异：

| mini-flow 第 22 篇 | 真实 React Flow |
| --- | --- |
| 直接从 nodes 计算简化 handle 坐标 | 依赖 `InternalNode.internals.handleBounds` |
| pointer capture 简化监听 | document / shadow root 级 move/up 监听 |
| 只做 strict mode | 支持 strict / loose mode |
| 简化目标查找 | 同时考虑 handleBelow、closest handle、DOM data 属性 |
| 暂不做 autoPanOnConnect / click connect / reconnect | 真实连接系统覆盖这些交互 |

---

## 1. 这一篇要解决的问题

先看一个很容易写出来的版本：

```tsx
function Handle({ nodeId }) {
  function onPointerUp(targetNodeId) {
    setEdges((edges) => [
      ...edges,
      {
        id: `${nodeId}-${targetNodeId}`,
        source: nodeId,
        target: targetNodeId,
      },
    ]);
  }

  return <div className="handle" />;
}
```

它的问题不在于不能连线。

它的问题是跳过了连接系统真正复杂的部分：

```txt
从哪个 handle 出发？
当前是否正在连接？
临时线应该怎么画？
鼠标现在靠近哪个 handle？
source handle 能不能连 target handle？
source->source 或 target->target 要不要允许？
用户自定义 isValidConnection 怎么介入？
pointer up 时如果不合法要不要触发 onConnect？
Connection 什么时候变成 Edge？
```

这些问题不能靠 `setEdges([...edges, edge])` 解决。

React Flow 的源码把它拆成几层：

```txt
Handle component
  负责 DOM、className、data attributes、pointer down

XYHandle
  负责连接手势生命周期、目标 handle 查找、合法性判断

connection state
  保存 inProgress / from / to / isValid / handles

ConnectionLineWrapper
  只根据 connection state 渲染临时线

onConnect / addEdge
  把合法 Connection 交给用户或非受控 edges
```

这一篇要解决的问题就是：

```txt
如何把 pointer gesture 转成 Connection，而不是直接绕过中间态创建 Edge？
```

---

## 2. 先看用户 API 或运行效果

完成后，用户可以这样写：

```tsx
const [edges, setEdges] = useState<MiniEdge[]>([]);

const onConnect = useCallback((connection: Connection) => {
  setEdges((edges) => addEdge(connection, edges));
}, []);

const nodes: MiniNode[] = [
  {
    id: 'input',
    position: { x: 80, y: 120 },
    data: { label: 'Input' },
  },
  {
    id: 'output',
    position: { x: 420, y: 180 },
    data: { label: 'Output' },
  },
];

export function App() {
  return (
    <MiniFlow
      nodes={nodes}
      edges={edges}
      onConnect={onConnect}
      isValidConnection={(connection) => connection.source !== connection.target}
    />
  );
}
```

节点内部会渲染 handle：

```tsx
function MiniNodeView({ node }) {
  return (
    <div className="mini-flow__node">
      <Handle type="target" position="left" />
      {node.data?.label}
      <Handle type="source" position="right" />
    </div>
  );
}
```

用户看到的行为是：

```txt
按住 source handle
  ↓
出现临时连接线
  ↓
拖到 target handle 附近
  ↓
临时线进入 valid 状态
  ↓
松手
  ↓
onConnect(connection)
  ↓
用户 addEdge(connection, edges)
  ↓
EdgeRenderer 渲染正式 edge
```

如果拖到空白处松手：

```txt
取消 connection
不触发 onConnect
```

如果 `isValidConnection` 返回 `false`：

```txt
临时线可以显示 invalid 状态
pointer up 不触发 onConnect
```

这就是连接系统最小闭环。

---

## 3. 核心概念解释

这一篇只需要四个概念：

```txt
Handle
Connection
ConnectionState
ConnectionLine
```

### 3.1 Handle

Handle 是节点上的连接点。

它不是 edge。

它也不是 node。

它是：

```txt
node 上允许开始连接或结束连接的一个点
```

最小字段是：

```ts
type HandleType = 'source' | 'target';
type HandlePosition = 'left' | 'right' | 'top' | 'bottom';
```

真实 React Flow 的 `Handle` 组件会在 DOM 上写入 `data-nodeid`、`data-handleid`、`data-handlepos`、className 等信息，让 `XYHandle` 可以从 DOM 反查当前鼠标下方的 handle。

第 22 篇 mini-flow 也会这么做。

### 3.2 Connection

Connection 是一次合法连接的最小描述：

```ts
type Connection = {
  source: string;
  target: string;
  sourceHandle: string | null;
  targetHandle: string | null;
};
```

它不是 Edge。

Edge 至少还需要：

```txt
id
source
target
sourceHandle
targetHandle
```

还可能有：

```txt
type
data
style
marker
label
selected
```

所以 `onConnect` 只交出 `Connection` 是合理的。

用户可以决定：

- 是否真的创建 edge。
- 创建什么 id。
- 使用什么 edge type。
- 是否附加业务 data。
- 是否拒绝这次连接。

### 3.3 ConnectionState

连接进行中，需要保存更多临时状态：

```ts
type ConnectionState = {
  inProgress: boolean;
  isValid: boolean | null;
  from: XYPosition;
  to: XYPosition;
  fromNodeId: string;
  fromHandleId: string | null;
  fromHandleType: HandleType;
  toNodeId: string | null;
  toHandleId: string | null;
  toHandleType: HandleType | null;
};
```

这不是最终数据。

它是连接交互过程里的中间态。

真实 React Flow 把它放在 store 的 `connection` 字段里，初始化为 `initialConnection`。证据在：

```txt
packages/react/src/store/initialState.ts:134
```

`XYHandle` 通过 store action 更新它：

```txt
updateConnection(connection)
cancelConnection()
```

证据在：

```txt
packages/react/src/store/index.ts:441
packages/react/src/store/index.ts:446
```

### 3.4 ConnectionLine

ConnectionLine 只负责画临时线。

它不负责找 handle。

它不负责校验连接。

它不负责创建 edge。

真实 `ConnectionLineWrapper` 的判断非常直接：

```txt
width 存在
nodesConnectable
connection.inProgress
  -> render connection line
```

证据在：

```txt
packages/react/src/components/ConnectionLine/index.tsx:32
packages/react/src/components/ConnectionLine/index.tsx:38
packages/react/src/components/ConnectionLine/index.tsx:41
```

这就是本篇 mini-flow 也要保留的边界：

```txt
连接状态由 useConnectionDrag 更新
临时线由 ConnectionLine 消费 connection state
```

---

## 4. 源码入口在哪里

### 4.1 Handle：把 React store 能力注入 XYHandle

源码位置：

```txt
packages/react/src/components/Handle/index.tsx
```

`Handle` 内部有一个 `onConnectExtended`。

当连接合法时，它会：

```txt
合并 defaultEdgeOptions
如果 hasDefaultEdges，则内部 setEdges(addEdge(...))
调用 store onConnect
调用当前 Handle props 上的 onConnect
```

证据在：

```txt
packages/react/src/components/Handle/index.tsx:102
packages/react/src/components/Handle/index.tsx:109
packages/react/src/components/Handle/index.tsx:111
packages/react/src/components/Handle/index.tsx:114
```

这说明 React Flow 有非受控 edges 的内部 addEdge，也有受控模式下交给用户的 onConnect。

`Handle` 的 `onPointerDown` 会调用：

```txt
XYHandle.onPointerDown(event.nativeEvent, { ... })
```

并把这些能力注入进去：

```txt
nodeLookup
connectionMode
connectionRadius
panBy
cancelConnection
updateConnection
onConnectStart
onConnectEnd
onConnect
isValidConnection
getTransform
getFromHandle
```

证据在：

```txt
packages/react/src/components/Handle/index.tsx:130
packages/react/src/components/Handle/index.tsx:136
packages/react/src/components/Handle/index.tsx:142
packages/react/src/components/Handle/index.tsx:146
packages/react/src/components/Handle/index.tsx:147
```

这和第 21 篇 `useDrag` 注入 `getStoreItems` 是同一个模式：

```txt
React 绑定层负责注入环境
system controller 负责交互算法
```

### 4.2 XYHandle.onPointerDown：建立连接起点

源码位置：

```txt
packages/system/src/xyhandle/XYHandle.ts
```

`XYHandle.onPointerDown` 开始时会：

```txt
读取事件位置
判断 handle type
读取 container bounds
找到 fromHandleInternal
计算 from handle 的实际位置
构造 previousConnection
```

关键证据：

```txt
packages/system/src/xyhandle/XYHandle.ts:60
packages/system/src/xyhandle/XYHandle.ts:61
packages/system/src/xyhandle/XYHandle.ts:69
packages/system/src/xyhandle/XYHandle.ts:100
packages/system/src/xyhandle/XYHandle.ts:102
```

`previousConnection` 里包含：

```txt
inProgress
isValid
from
fromHandle
fromNode
to
toHandle
toNode
pointer
```

这说明连接开始时，还没有 target。

只有起点和当前 pointer。

### 4.3 pointer move：寻找目标 handle 并更新 connection state

pointer move 时，`XYHandle` 会：

```txt
把 pointer 转成 renderer / flow 坐标
根据 connectionRadius 找最近 handle
再用 isValidHandle 进行合法性校验
更新 connection.to / toHandle / isValid
```

关键证据：

```txt
packages/system/src/xyhandle/XYHandle.ts:147
packages/system/src/xyhandle/XYHandle.ts:149
packages/system/src/xyhandle/XYHandle.ts:161
packages/system/src/xyhandle/XYHandle.ts:183
packages/system/src/xyhandle/XYHandle.ts:197
```

其中 `getClosestHandle` 在：

```txt
packages/system/src/xyhandle/utils.ts:28
```

它会从附近节点的 handleBounds 中找连接半径内最近的 handle。

这解释了为什么 InternalNode 要保存 `handleBounds`。

没有 handle 测量，连接系统就只能靠 DOM 猜。

### 4.4 isValidHandle：ConnectionMode 和 isValidConnection

`XYHandle.isValidHandle` 会优先看鼠标下方的 DOM handle，再回退到最近 handle。

它会构造：

```ts
const connection: Connection = {
  source,
  sourceHandle,
  target,
  targetHandle,
};
```

证据在：

```txt
packages/system/src/xyhandle/XYHandle.ts:272
packages/system/src/xyhandle/XYHandle.ts:278
packages/system/src/xyhandle/XYHandle.ts:298
```

合法性检查包括：

```txt
handle connectable
connectable end
strict 模式下 source 只能连 target / target 只能连 source
loose 模式下不能连同一个 handle
用户传入 isValidConnection
```

证据在：

```txt
packages/system/src/xyhandle/XYHandle.ts:307
packages/system/src/xyhandle/XYHandle.ts:309
packages/system/src/xyhandle/XYHandle.ts:315
```

第 22 篇 mini-flow 会先实现 strict 模式。

也就是：

```txt
source -> target
target -> source
```

### 4.5 pointer up：只在合法连接时触发 onConnect

pointer up 时，源码逻辑很清楚：

```txt
if connectionStarted:
  if closestHandle/resultHandleDomNode && connection && isValid:
    onConnect(connection)
  onConnectEnd(finalConnectionState)
cancelConnection()
remove event listeners
```

证据在：

```txt
packages/system/src/xyhandle/XYHandle.ts:201
packages/system/src/xyhandle/XYHandle.ts:207
packages/system/src/xyhandle/XYHandle.ts:209
packages/system/src/xyhandle/XYHandle.ts:223
packages/system/src/xyhandle/XYHandle.ts:230
```

这给 mini-flow 一个明确规则：

```txt
无效连接不会触发 onConnect
无论是否有效，连接结束都要清理 connection state
```

### 4.6 ConnectionLineWrapper：只消费 connection state

源码位置：

```txt
packages/react/src/components/ConnectionLine/index.tsx
```

`ConnectionLineWrapper` 只判断要不要渲染。

真正的 `ConnectionLine` 会从 `useConnection()` 读取：

```txt
from
to
fromNode
fromHandle
toNode
toHandle
fromPosition
toPosition
pointer
```

证据在：

```txt
packages/react/src/components/ConnectionLine/index.tsx:72
```

然后根据 `ConnectionLineType` 选择 path 算法：

```txt
Bezier
SimpleBezier
Step
SmoothStep
Straight
```

证据在：

```txt
packages/react/src/components/ConnectionLine/index.tsx:111
packages/react/src/components/ConnectionLine/index.tsx:128
```

mini-flow 先画 straight path。

### 4.7 addEdge：Connection 到 Edge 的转换

源码位置：

```txt
packages/system/src/utils/edges/general.ts
```

`addEdge` 会：

```txt
校验 source / target
生成 edge id
避免重复连接
删除 null handle 字段
返回新的 edges array
```

证据在：

```txt
packages/system/src/utils/edges/general.ts:134
packages/system/src/utils/edges/general.ts:139
packages/system/src/utils/edges/general.ts:151
packages/system/src/utils/edges/general.ts:157
packages/system/src/utils/edges/general.ts:169
```

这也说明：

```txt
addEdge 是数据工具，不是渲染工具。
```

---

## 5. 源码调用链

真实 React Flow 的连接链路可以压缩成：

```txt
Handle DOM pointerdown
  ↓
Handle.onPointerDown
  ↓
XYHandle.onPointerDown(event, store capabilities)
  ↓
getHandle(nodeId, handleType, handleId, nodeLookup)
  ↓
previousConnection(inProgress=true, from, to=pointer)
  ↓
updateConnection(previousConnection)
  ↓
ConnectionLineWrapper 读取 store.connection 并渲染临时线
  ↓
pointermove
  ↓
pointToRendererPoint(position, transform)
  ↓
getClosestHandle(...)
  ↓
isValidHandle(...)
  ↓
updateConnection(newConnection)
  ↓
pointerup
  ↓
if valid: onConnect(connection)
  ↓
onConnectEnd(...)
  ↓
cancelConnection()
```

mini-flow 第 22 篇压缩成：

```txt
<Handle />
  ↓
onPointerDown
  ↓
createConnectionState(from handle, pointer)
  ↓
setConnection(inProgress)
  ↓
window pointermove
  ↓
findHandleAtPoint / getClosestHandle
  ↓
isValidConnection
  ↓
setConnection(next)
  ↓
<ConnectionLine connection={connection} />
  ↓
window pointerup
  ↓
if valid: onConnect(connection)
  ↓
setConnection(null)
```

这个 mini 版本不复刻所有细节。

但它保留了关键边界：

```txt
Handle 发起连接
connection state 表达中间态
ConnectionLine 只负责渲染
onConnect 只接收 Connection
addEdge 才创建 Edge
```

---

## 6. 关键数据结构

### 6.1 HandleType 和 HandlePosition

```ts
type HandleType = 'source' | 'target';
type HandlePosition = 'left' | 'right' | 'top' | 'bottom';
```

position 用来决定 handle 在节点上的位置，也用于 connection line 的 path 方向。

mini-flow 先只做 left / right：

```txt
target handle 在左侧
source handle 在右侧
```

### 6.2 HandleInfo

```ts
type HandleInfo = {
  nodeId: string;
  handleId: string | null;
  type: HandleType;
  position: HandlePosition;
  x: number;
  y: number;
};
```

这里的 `x/y` 是 flow 坐标。

真实 React Flow 的 handle bounds 也是相对节点测量后，再结合 node position 计算绝对位置。

第 22 篇为了简化，可以根据节点 position 和固定尺寸直接计算 handle center。

### 6.3 Connection

```ts
type Connection = {
  source: string;
  target: string;
  sourceHandle: string | null;
  targetHandle: string | null;
};
```

它是 `onConnect` 的参数。

不要提前塞进 edge id。

### 6.4 ConnectionState

```ts
type ConnectionState = {
  inProgress: boolean;
  isValid: boolean | null;
  from: XYPosition;
  to: XYPosition;
  fromHandle: HandleInfo;
  toHandle: HandleInfo | null;
  connection: Connection | null;
};
```

`from/to` 用来画临时线。

`connection` 用来 pointer up 时触发 `onConnect`。

`isValid` 用来给临时线加状态样式。

### 6.5 MiniFlowProps

```ts
type MiniFlowProps = {
  nodes: MiniNode[];
  edges: MiniEdge[];
  onConnect?: (connection: Connection) => void;
  isValidConnection?: (connection: Connection) => boolean;
  connectionRadius?: number;
};
```

这里不直接传 `setEdges`。

原因和 `onNodesChange` 一样：

```txt
MiniFlow 负责产生交互结果
用户负责决定是否应用
```

---

## 7. 关键实现思路

### 7.1 addEdge

先写数据工具：

```ts
function getEdgeId(connection: Connection) {
  return `mini-edge__${connection.source}${connection.sourceHandle ?? ''}-${connection.target}${connection.targetHandle ?? ''}`;
}

function connectionExists(connection: Connection, edges: MiniEdge[]) {
  return edges.some((edge) => {
    return (
      edge.source === connection.source &&
      edge.target === connection.target &&
      (edge.sourceHandle ?? null) === connection.sourceHandle &&
      (edge.targetHandle ?? null) === connection.targetHandle
    );
  });
}

export function addEdge(connection: Connection, edges: MiniEdge[]): MiniEdge[] {
  if (!connection.source || !connection.target) {
    return edges;
  }

  if (connectionExists(connection, edges)) {
    return edges;
  }

  return edges.concat({
    id: getEdgeId(connection),
    source: connection.source,
    target: connection.target,
    sourceHandle: connection.sourceHandle,
    targetHandle: connection.targetHandle,
  });
}
```

这个逻辑对应真实 `addEdge`：

```txt
校验 source / target
生成 id
过滤重复连接
返回新数组
```

### 7.2 计算 handle 位置

第 22 篇不引入 DOM 测量系统，先根据节点位置和尺寸计算：

```ts
function getNodeHandlePosition(
  node: MiniNode,
  type: HandleType,
  position: HandlePosition
): XYPosition {
  const size = getNodeSize(node);

  if (position === 'left') {
    return {
      x: node.position.x,
      y: node.position.y + size.height / 2,
    };
  }

  if (position === 'right') {
    return {
      x: node.position.x + size.width,
      y: node.position.y + size.height / 2,
    };
  }

  if (position === 'top') {
    return {
      x: node.position.x + size.width / 2,
      y: node.position.y,
    };
  }

  return {
    x: node.position.x + size.width / 2,
    y: node.position.y + size.height,
  };
}
```

真实 React Flow 会用 `handleBounds` 和 `getHandlePosition`。

第 22 篇先用固定位置。

### 7.3 收集所有 handle

```ts
function getNodeHandles(node: MiniNode): HandleInfo[] {
  return [
    {
      nodeId: node.id,
      handleId: 'target',
      type: 'target',
      position: 'left',
      ...getNodeHandlePosition(node, 'target', 'left'),
    },
    {
      nodeId: node.id,
      handleId: 'source',
      type: 'source',
      position: 'right',
      ...getNodeHandlePosition(node, 'source', 'right'),
    },
  ];
}

function getAllHandles(nodes: MiniNode[]) {
  return nodes.flatMap(getNodeHandles);
}
```

真实源码里，这一步来自 `nodeLookup` 的 `internals.handleBounds`。

mini-flow 直接从 nodes 计算。

### 7.4 找最近 handle

```ts
function getClosestHandle(
  point: XYPosition,
  handles: HandleInfo[],
  radius: number,
  fromHandle: HandleInfo
) {
  let closest: HandleInfo | null = null;
  let minDistance = Infinity;

  for (const handle of handles) {
    if (
      handle.nodeId === fromHandle.nodeId &&
      handle.type === fromHandle.type &&
      handle.handleId === fromHandle.handleId
    ) {
      continue;
    }

    const distance = Math.hypot(handle.x - point.x, handle.y - point.y);

    if (distance <= radius && distance < minDistance) {
      closest = handle;
      minDistance = distance;
    }
  }

  return closest;
}
```

真实 `getClosestHandle` 还会先缩小到附近 nodes，再遍历 handles，并在多个 handle 等距时偏好 opposite handle。

第 22 篇 mini 版直接遍历所有 handle。

### 7.5 构造 Connection

```ts
function buildConnection(from: HandleInfo, to: HandleInfo): Connection {
  const fromIsTarget = from.type === 'target';

  return {
    source: fromIsTarget ? to.nodeId : from.nodeId,
    sourceHandle: fromIsTarget ? to.handleId : from.handleId,
    target: fromIsTarget ? from.nodeId : to.nodeId,
    targetHandle: fromIsTarget ? from.handleId : to.handleId,
  };
}
```

这个逻辑对应真实 `isValidHandle` 中的构造：

```txt
source: isTarget ? handleNodeId : fromNodeId
target: isTarget ? fromNodeId : handleNodeId
```

证据在：

```txt
packages/system/src/xyhandle/XYHandle.ts:298
```

### 7.6 校验连接

```ts
function isStrictHandlePair(from: HandleInfo, to: HandleInfo) {
  return (
    (from.type === 'source' && to.type === 'target') ||
    (from.type === 'target' && to.type === 'source')
  );
}

function validateConnection(
  from: HandleInfo,
  to: HandleInfo,
  isValidConnection?: (connection: Connection) => boolean
) {
  if (!isStrictHandlePair(from, to)) {
    return { isValid: false, connection: null };
  }

  const connection = buildConnection(from, to);
  const userValid = isValidConnection?.(connection) ?? true;

  return {
    isValid: userValid,
    connection: userValid ? connection : null,
  };
}
```

第 22 篇只实现 strict mode。

loose mode 可以留给扩展。

### 7.7 useConnectionDrag

这是 mini-flow 的 `XYHandle`。

它做：

```txt
pointerdown:
  创建 connection state
  绑定 window pointermove / pointerup

pointermove:
  screen -> flow
  找最近 handle
  校验连接
  更新 connection state

pointerup:
  如果 connection valid，触发 onConnect
  清理 connection state
```

实现草图：

```tsx
type UseConnectionDragParams = {
  nodes: MiniNode[];
  viewport: Viewport;
  containerRef: React.RefObject<HTMLDivElement | null>;
  connectionRadius: number;
  setConnection: (connection: ConnectionState | null) => void;
  onConnect?: (connection: Connection) => void;
  isValidConnection?: (connection: Connection) => boolean;
};

function useConnectionDrag({
  nodes,
  viewport,
  containerRef,
  connectionRadius,
  setConnection,
  onConnect,
  isValidConnection,
}: UseConnectionDragParams) {
  const sessionRef = useRef<{
    fromHandle: HandleInfo;
    connection: Connection | null;
    isValid: boolean | null;
  } | null>(null);

  function clientToFlow(event: PointerEvent | React.PointerEvent) {
    const container = containerRef.current;

    if (!container) {
      return null;
    }

    const rect = container.getBoundingClientRect();
    const point = {
      x: event.clientX - rect.left,
      y: event.clientY - rect.top,
    };

    return screenToFlowPosition(point, viewport);
  }

  function start(event: React.PointerEvent, fromHandle: HandleInfo) {
    event.stopPropagation();
    event.currentTarget.setPointerCapture(event.pointerId);

    const pointer = clientToFlow(event);

    if (!pointer) {
      return;
    }

    sessionRef.current = {
      fromHandle,
      connection: null,
      isValid: null,
    };

    setConnection({
      inProgress: true,
      isValid: null,
      from: { x: fromHandle.x, y: fromHandle.y },
      to: pointer,
      fromHandle,
      toHandle: null,
      connection: null,
    });
  }

  function move(event: React.PointerEvent) {
    const session = sessionRef.current;

    if (!session) {
      return;
    }

    const pointer = clientToFlow(event);

    if (!pointer) {
      return;
    }

    const handles = getAllHandles(nodes);
    const toHandle = getClosestHandle(
      pointer,
      handles,
      connectionRadius,
      session.fromHandle
    );

    const result = toHandle
      ? validateConnection(session.fromHandle, toHandle, isValidConnection)
      : { isValid: false, connection: null };

    session.connection = result.connection;
    session.isValid = result.isValid;

    setConnection({
      inProgress: true,
      isValid: result.isValid,
      from: { x: session.fromHandle.x, y: session.fromHandle.y },
      to: toHandle && result.isValid ? { x: toHandle.x, y: toHandle.y } : pointer,
      fromHandle: session.fromHandle,
      toHandle: result.isValid ? toHandle : null,
      connection: result.connection,
    });
  }

  function end(event: React.PointerEvent) {
    const session = sessionRef.current;

    if (!session) {
      return;
    }

    if (session.connection && session.isValid) {
      onConnect?.(session.connection);
    }

    setConnection(null);
    sessionRef.current = null;
    event.currentTarget.releasePointerCapture(event.pointerId);
  }

  return { start, move, end };
}
```

这个版本把 pointermove / pointerup 绑定在 handle 元素上。

真实 React Flow 会通过 `getHostForElement` 挂到 document 或 shadow root，避免鼠标离开 handle 后事件丢失。证据在：

```txt
packages/system/src/xyhandle/XYHandle.ts:55
packages/system/src/xyhandle/XYHandle.ts:244
```

mini-flow 可以先用 pointer capture。

### 7.8 Handle 组件

```tsx
type HandleProps = {
  node: MiniNode;
  type: HandleType;
  position: HandlePosition;
  id?: string;
  connectionDrag: ReturnType<typeof useConnectionDrag>;
};

function Handle({ node, type, position, id, connectionDrag }: HandleProps) {
  const point = getNodeHandlePosition(node, type, position);
  const handle: HandleInfo = {
    nodeId: node.id,
    handleId: id ?? type,
    type,
    position,
    x: point.x,
    y: point.y,
  };

  return (
    <div
      className={`mini-flow__handle ${type} ${position}`}
      data-nodeid={node.id}
      data-handleid={handle.handleId ?? ''}
      data-handletype={type}
      data-handlepos={position}
      onPointerDown={(event) => connectionDrag.start(event, handle)}
      onPointerMove={connectionDrag.move}
      onPointerUp={connectionDrag.end}
      onPointerCancel={connectionDrag.end}
    />
  );
}
```

真实 Handle 也会用 DOM data attributes 表达 nodeId / handleId / handle type。

mini-flow 的 className 和 data attribute 可以简单一些。

### 7.9 ConnectionLine

```tsx
function ConnectionLine({ connection }: { connection: ConnectionState | null }) {
  if (!connection?.inProgress) {
    return null;
  }

  const path = `M ${connection.from.x} ${connection.from.y} L ${connection.to.x} ${connection.to.y}`;

  return (
    <svg className="mini-flow__connection-line">
      <path
        d={path}
        className={
          connection.isValid
            ? 'mini-flow__connection-path valid'
            : 'mini-flow__connection-path'
        }
      />
    </svg>
  );
}
```

这个组件只消费 connection state。

它不查 nodes，不找 handle，不调用 onConnect。

这就是它和 `useConnectionDrag` 的边界。

### 7.10 MiniFlow 接入

```tsx
function MiniFlow({
  nodes,
  edges,
  onConnect,
  isValidConnection,
  connectionRadius = 24,
}: MiniFlowProps) {
  const containerRef = useRef<HTMLDivElement | null>(null);
  const [connection, setConnection] = useState<ConnectionState | null>(null);
  const [viewport, setViewport] = useState({ x: 0, y: 0, zoom: 1 });

  const connectionDrag = useConnectionDrag({
    nodes,
    viewport,
    containerRef,
    connectionRadius,
    setConnection,
    onConnect,
    isValidConnection,
  });

  return (
    <div ref={containerRef} className="mini-flow">
      <MiniViewport viewport={viewport}>
        <MiniEdgeRenderer edges={edges} nodeLookup={nodeLookup} />
        <ConnectionLine connection={connection} />
        <MiniNodeRenderer
          nodes={nodes}
          connectionDrag={connectionDrag}
        />
      </MiniViewport>
    </div>
  );
}
```

注意 `ConnectionLine` 的位置。

它应该和 edges / nodes 在同一个 viewport 下。

第 6 篇和第 19 篇已经讲过，React Flow 的 `GraphView` 把 `ConnectionLineWrapper` 放在 `EdgeRenderer` 和 `NodeRenderer` 之间：

```txt
EdgeRenderer
ConnectionLineWrapper
NodeRenderer
```

第 22 篇 mini-flow 也沿用这个层级。

---

## 8. 这部分源码的设计取舍

### 8.1 为什么 onConnect 给 Connection，而不是 Edge

因为连接手势只证明了一件事：

```txt
source handle 和 target handle 之间形成了关系
```

它不知道业务上要不要创建 Edge。

也不知道 edge id、edge type、edge data 应该是什么。

所以 React Flow 只交出 `Connection`。

用户可以：

```ts
setEdges((edges) => addEdge(connection, edges));
```

也可以：

```ts
setEdges((edges) => [
  ...edges,
  {
    id: crypto.randomUUID(),
    type: 'workflow',
    data: { createdBy: 'user' },
    ...connection,
  },
]);
```

这个边界让连接系统保持通用。

### 8.2 为什么 ConnectionLine 不负责校验

如果 ConnectionLine 负责校验，它就必须知道：

- nodeLookup。
- handleBounds。
- connectionMode。
- isValidConnection。
- current pointer。
- source handle。
- target handle。

那它就不再是渲染组件，而是交互控制器。

React Flow 把这些放在 `XYHandle`。

`ConnectionLineWrapper` 只消费 connection state。

这让渲染层更稳定，也让自定义 ConnectionLine 更容易。

### 8.3 为什么需要 connection state

连接过程中，很多模块要知道当前连接状态：

- `ConnectionLineWrapper` 要渲染临时线。
- `Handle` 要显示 possible / valid / connecting 样式。
- store 要在 `cancelConnection` 时清理。
- callbacks 要拿到 start / end 状态。

如果 connection 只存在于 `XYHandle` 局部变量里，其他模块无法响应。

所以真实 React Flow 把 connection 放在 store。

第 22 篇 mini-flow 还没有 store，先用 `useState`。

第 23 篇会把它收进统一 store。

### 8.4 为什么找 handle 既看距离，也看鼠标下方 DOM

真实 `isValidHandle` 有一个细节：

```txt
优先使用鼠标下方的 handle
再使用最近距离 handle
```

原因是两个 handle 很近时，几何中心距离不一定符合用户意图。

用户真正指向的 DOM handle 应该优先。

mini-flow 第一版只做距离查找。

但文章要记住这个差异：

```txt
生产级连接系统不能只看距离。
```

### 8.5 为什么 strict / loose mode 是连接系统的一部分

很多图编辑器只允许：

```txt
source -> target
```

但有些场景允许更松的连接规则。

React Flow 把它抽成 `ConnectionMode.Strict` / `ConnectionMode.Loose`。

第 22 篇只实现 strict。

这是合理简化。

因为本篇目标是：

```txt
理解连接系统主链路
```

而不是覆盖所有模式。

---

## 9. 如果我们自己实现，最小版本应该怎么写

这一节把前面的碎片合在一起。

### 9.1 类型

```ts
type HandleType = 'source' | 'target';
type HandlePosition = 'left' | 'right' | 'top' | 'bottom';

type Connection = {
  source: string;
  target: string;
  sourceHandle: string | null;
  targetHandle: string | null;
};

type HandleInfo = {
  nodeId: string;
  handleId: string | null;
  type: HandleType;
  position: HandlePosition;
  x: number;
  y: number;
};

type ConnectionState = {
  inProgress: boolean;
  isValid: boolean | null;
  from: XYPosition;
  to: XYPosition;
  fromHandle: HandleInfo;
  toHandle: HandleInfo | null;
  connection: Connection | null;
};
```

### 9.2 addEdge

```ts
function getEdgeId(connection: Connection) {
  return `mini-edge__${connection.source}${connection.sourceHandle ?? ''}-${connection.target}${connection.targetHandle ?? ''}`;
}

function addEdge(connection: Connection, edges: MiniEdge[]): MiniEdge[] {
  if (!connection.source || !connection.target) {
    return edges;
  }

  const exists = edges.some((edge) => {
    return (
      edge.source === connection.source &&
      edge.target === connection.target &&
      (edge.sourceHandle ?? null) === connection.sourceHandle &&
      (edge.targetHandle ?? null) === connection.targetHandle
    );
  });

  if (exists) {
    return edges;
  }

  return edges.concat({
    id: getEdgeId(connection),
    source: connection.source,
    target: connection.target,
    sourceHandle: connection.sourceHandle,
    targetHandle: connection.targetHandle,
  });
}
```

### 9.3 useConnectionDrag

```tsx
function useConnectionDrag({
  nodes,
  viewport,
  containerRef,
  connectionRadius,
  setConnection,
  onConnect,
  isValidConnection,
}: UseConnectionDragParams) {
  const sessionRef = useRef<{
    fromHandle: HandleInfo;
    connection: Connection | null;
    isValid: boolean | null;
  } | null>(null);

  function clientToFlow(event: PointerEvent | React.PointerEvent) {
    const container = containerRef.current;

    if (!container) {
      return null;
    }

    const rect = container.getBoundingClientRect();

    return screenToFlowPosition(
      {
        x: event.clientX - rect.left,
        y: event.clientY - rect.top,
      },
      viewport
    );
  }

  function start(event: React.PointerEvent, fromHandle: HandleInfo) {
    event.stopPropagation();
    event.currentTarget.setPointerCapture(event.pointerId);

    const pointer = clientToFlow(event);

    if (!pointer) {
      return;
    }

    sessionRef.current = {
      fromHandle,
      connection: null,
      isValid: null,
    };

    setConnection({
      inProgress: true,
      isValid: null,
      from: { x: fromHandle.x, y: fromHandle.y },
      to: pointer,
      fromHandle,
      toHandle: null,
      connection: null,
    });
  }

  function move(event: React.PointerEvent) {
    const session = sessionRef.current;

    if (!session) {
      return;
    }

    const pointer = clientToFlow(event);

    if (!pointer) {
      return;
    }

    const handles = getAllHandles(nodes);
    const toHandle = getClosestHandle(
      pointer,
      handles,
      connectionRadius,
      session.fromHandle
    );

    const result = toHandle
      ? validateConnection(session.fromHandle, toHandle, isValidConnection)
      : { isValid: false, connection: null };

    session.connection = result.connection;
    session.isValid = result.isValid;

    setConnection({
      inProgress: true,
      isValid: result.isValid,
      from: { x: session.fromHandle.x, y: session.fromHandle.y },
      to: toHandle && result.isValid ? { x: toHandle.x, y: toHandle.y } : pointer,
      fromHandle: session.fromHandle,
      toHandle: result.isValid ? toHandle : null,
      connection: result.connection,
    });
  }

  function end(event: React.PointerEvent) {
    const session = sessionRef.current;

    if (!session) {
      return;
    }

    if (session.connection && session.isValid) {
      onConnect?.(session.connection);
    }

    setConnection(null);
    sessionRef.current = null;
    event.currentTarget.releasePointerCapture(event.pointerId);
  }

  return { start, move, end };
}
```

### 9.4 Handle 和 ConnectionLine

```tsx
function Handle({ node, type, position, connectionDrag }: HandleProps) {
  const point = getNodeHandlePosition(node, type, position);
  const handle: HandleInfo = {
    nodeId: node.id,
    handleId: type,
    type,
    position,
    x: point.x,
    y: point.y,
  };

  return (
    <div
      className={`mini-flow__handle ${type} ${position}`}
      data-nodeid={node.id}
      data-handleid={type}
      data-handletype={type}
      data-handlepos={position}
      onPointerDown={(event) => connectionDrag.start(event, handle)}
      onPointerMove={connectionDrag.move}
      onPointerUp={connectionDrag.end}
      onPointerCancel={connectionDrag.end}
    />
  );
}

function ConnectionLine({ connection }: { connection: ConnectionState | null }) {
  if (!connection?.inProgress) {
    return null;
  }

  const path = `M ${connection.from.x} ${connection.from.y} L ${connection.to.x} ${connection.to.y}`;

  return (
    <svg className="mini-flow__connection-line">
      <path
        d={path}
        className={
          connection.isValid
            ? 'mini-flow__connection-path valid'
            : 'mini-flow__connection-path'
        }
      />
    </svg>
  );
}
```

### 9.5 CSS

```css
.mini-flow__handle {
  position: absolute;
  width: 10px;
  height: 10px;
  border: 2px solid #fff;
  border-radius: 50%;
  background: #2563eb;
  transform: translate(-50%, -50%);
  cursor: crosshair;
}

.mini-flow__handle.left {
  left: 0;
  top: 50%;
}

.mini-flow__handle.right {
  right: 0;
  top: 50%;
}

.mini-flow__connection-line {
  position: absolute;
  inset: 0;
  width: 100%;
  height: 100%;
  overflow: visible;
  pointer-events: none;
}

.mini-flow__connection-path {
  fill: none;
  stroke: #94a3b8;
  stroke-width: 2;
  stroke-dasharray: 6 4;
}

.mini-flow__connection-path.valid {
  stroke: #16a34a;
  stroke-dasharray: none;
}
```

这里有一个细节：

`ConnectionLine` 和 `EdgeRenderer` 一样，应该处在 viewport 内。

它的 path 使用 flow 坐标，再由 viewport transform 统一投影。

---

## 10. 本篇总结

这一篇实现了 mini-flow 的连接系统最小闭环：

```txt
Handle
  ↓
connection drag
  ↓
connection state
  ↓
ConnectionLine
  ↓
onConnect
  ↓
addEdge
```

它验证了 React Flow 源码里的几个核心判断：

```txt
1. Handle 是连接点，不是 edge。
2. Connection 是关系描述，不是完整 Edge。
3. 连接过程中必须有 connection in progress 中间态。
4. ConnectionLine 只消费状态，不负责找 handle 或校验。
5. XYHandle 负责 pointer 生命周期、目标查找和合法性判断。
6. pointer up 只有合法连接才触发 onConnect。
7. addEdge 是 Connection -> Edge 的数据工具。
```

这和第 21 篇的节点拖拽形成了完整对照：

```txt
XYDrag:
  pointer gesture -> node position changes

XYHandle:
  pointer gesture -> graph relationship connection
```

到这里，mini-flow 已经有了三条关键机制：

```txt
viewport runtime
node drag runtime
connection runtime
```

但它们现在还分散在 props、局部 state 和组件之间。

下一篇要把这些运行时状态收进统一 store。

---

## 11. 下一篇读什么

下一篇进入：

```txt
第 23 篇：实战：实现 store、Provider 和 hooks
```

前面几篇我们已经有：

```txt
nodes
edges
viewport
connection
onNodesChange
onConnect
```

但它们还不是一个真正的 runtime。

第 23 篇会把这些状态组织成：

```txt
MiniFlowProvider
  ↓
store
  ↓
useMiniFlow
useNodes
useEdges
useViewport
useStore
```

对应 React Flow 源码：

```txt
ReactFlowProvider
StoreUpdater
useStore
useReactFlow
```

也就是说，第 22 篇解决的是“关系怎么被创建”。

第 23 篇解决的是“这些状态应该住在哪里”。
