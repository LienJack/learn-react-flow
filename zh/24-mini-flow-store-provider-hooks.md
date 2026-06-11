---
title: "第 24 篇：实战：实现 store、Provider 和 hooks"
tags:
  - react-flow
  - xyflow
  - source-code
  - mini-flow
  - store
  - provider
  - hooks
---

# 第 24 篇：实战：实现 store、Provider 和 hooks

第 21 篇，我们实现了 viewport。

第 22 篇，我们实现了节点拖拽。

第 23 篇，我们实现了 Handle、ConnectionLine 和 onConnect。

现在 mini-flow 已经有了一个节点编辑器最核心的几条运行时链路：

```txt
viewport runtime
node drag runtime
connection runtime
```

但问题也随之暴露出来了。

这些状态现在很容易散在不同地方：

```txt
nodes / edges
  可能来自 props，也可能来自组件自己的 useState

viewport
  可能住在 MiniFlow 组件里

connection
  可能住在 ConnectionLine 附近

drag state
  可能住在 useNodeDrag 里

callbacks
  可能从 props 一层层传下去
```

只要示例还小，这样写没什么问题。

但一旦我们想支持这些能力：

```txt
<MiniControls />
<MiniMap />
<Background />
<Panel />
useMiniFlow()
useNodes()
useEdges()
useViewport()
controlled / uncontrolled
```

局部 state 就会开始失控。

这也是 React Flow 源码里 store、Provider、StoreUpdater 和 hooks 这几层存在的原因。

这一篇不只是给 mini-flow 加一个状态管理库。更重要的是，我们要验证第 7 篇、第 14 篇、第 16 篇读到的源码判断：

> React Flow 对外看起来是一个声明式 React 组件，对内其实有一个围绕 store 运行的交互运行时控制面。

mini-flow 到这一篇，才真正开始拥有“运行时”的形状。

更准确地说，store 是 mini-flow 的运行时控制面：它保存当前 graph、viewport、connection、callbacks 和 lookup，也提供改变这些状态的 actions。

本章新增 / 修改：

```txt
src/mini-flow/store.ts                  # createMiniStore / state / actions
src/mini-flow/MiniFlowProvider.tsx
src/mini-flow/MiniFlowWrapper.tsx
src/mini-flow/MiniStoreUpdater.tsx
src/mini-flow/hooks.ts                  # useMiniStore / useMiniFlow / useNodes...
src/mini-flow/MiniFlow.tsx              # 接入 Wrapper / Provider / children
```

本章验收 checklist：

```txt
- MiniFlow 不在已有 Provider 下时，会自动创建 store。
- 外层已经有 MiniFlowProvider 时，MiniFlow 复用已有 store。
- props.nodes 更新会同步到 store，并重建 nodeLookup。
- children 插件可以在 Provider 内调用 useMiniFlow / useMiniStore。
- useNodes 是订阅式，useMiniFlow().getNodes 是命令式快照。
```

本章 store API surface 先保持最小：

| API | 用途 |
| --- | --- |
| `setNodes` / `setEdges` | 同步外部 props 或内部 changes |
| `setViewport` / `fitView` / `zoomIn` / `zoomOut` | viewport 命令 |
| `setConnection` | 连接运行时状态 |
| `triggerNodeChanges` / `triggerEdgeChanges` | controlled / uncontrolled 回流 |
| `getNodes` / `getEdges` / `getViewport` | 命令式读取快照 |

如果同时传 `nodes` 和 `defaultNodes`，mini-flow 采用和 React controlled component 一致的直觉策略：`nodes` 作为受控数据优先，`defaultNodes` 只作为未受控时的初始值。`edges` / `defaultEdges` 同理。

---

## 1. 这一篇要解决的问题

这一篇要解决的问题可以用一句话概括：

> 前面实现出来的 nodes、edges、viewport、connection、callbacks，到底应该住在哪里？

最直觉的答案是：

```tsx
function MiniFlow(props) {
  const [nodes, setNodes] = useState(props.defaultNodes ?? []);
  const [edges, setEdges] = useState(props.defaultEdges ?? []);
  const [viewport, setViewport] = useState({ x: 0, y: 0, zoom: 1 });
  const [connection, setConnection] = useState(null);

  return <GraphView />;
}
```

这能跑。

但它有三个很快会撞上的问题。

第一个问题：深层组件要读状态。

`NodeRenderer` 要读 nodes，`EdgeRenderer` 要读 edges，`ConnectionLine` 要读 connection，`Controls` 要调用 zoom in / zoom out，`MiniMap` 要读 viewport 和 nodes。

如果全靠 props 传递，组件树会变成：

```txt
MiniFlow
  -> GraphView
    -> FlowRenderer
      -> Viewport
        -> EdgeRenderer
        -> NodeRenderer
        -> ConnectionLine
  -> Controls
  -> MiniMap
  -> Background
```

每加一个插件，就要重新穿一次状态和方法。

第二个问题：交互系统需要同步读写最新状态。

节点拖拽不是一次 render 内的纯 UI 计算。它的生命周期是：

```txt
pointer down
  -> 记录起点
pointer move
  -> 高频读取 transform、nodeLookup、snapGrid、nodeExtent
  -> 高频更新 node position
pointer up
  -> 清理 dragging
  -> 触发 onNodesChange / onNodeDragStop
```

这类逻辑如果只依赖闭包里的 state，很容易读到旧值。交互系统需要的是类似：

```ts
store.getState()
store.setState()
store.subscribe()
```

也就是一个可以脱离 React render 节奏、又能把变化通知回 React 组件的状态容器。

第三个问题：受控和非受控要共存。

React Flow 同时支持：

```tsx
// controlled
<ReactFlow
  nodes={nodes}
  edges={edges}
  onNodesChange={onNodesChange}
  onEdgesChange={onEdgesChange}
/>
```

也支持：

```tsx
// uncontrolled
<ReactFlow
  defaultNodes={nodes}
  defaultEdges={edges}
/>
```

这意味着内部 runtime 不能简单地说：

```txt
拖拽节点 -> 直接改 props.nodes
```

它必须先产生 change objects：

```txt
拖拽节点
  -> NodeChange[]
  -> 如果是 uncontrolled，内部 apply changes
  -> 无论如何，调用 onNodesChange
```

这就是第 14 篇讲过的 controlled / uncontrolled 分界。

所以，第 24 篇要补上的不是一个“全局变量”，而是一套完整的运行时边界：

```txt
MiniFlow props
  ↓
MiniFlowProvider
  ↓
store
  ↓
StoreUpdater
  ↓
renderers / interactions / plugins / hooks
```

---

## 2. 先看用户 API 或运行效果

我们希望 mini-flow 最终支持两种使用方式。

第一种：直接使用 `MiniFlow`，让它自己创建内部 store。

```tsx
export function App() {
  return (
    <MiniFlow
      defaultNodes={initialNodes}
      defaultEdges={initialEdges}
      onConnect={(connection) => {
        console.log("connect", connection);
      }}
    >
      <MiniControls />
      <MiniBackground />
      <MiniMap />
    </MiniFlow>
  );
}
```

这对应 React Flow 源码里的 `Wrapper` 行为：如果外面没有 store，就自动创建一个 `ReactFlowProvider`。

第二种：显式包一层 Provider，让外部插件或面板也能访问同一个 flow runtime。

```tsx
export function App() {
  return (
    <MiniFlowProvider defaultNodes={initialNodes} defaultEdges={initialEdges}>
      <Sidebar />
      <MiniFlow>
        <MiniControls />
      </MiniFlow>
    </MiniFlowProvider>
  );
}

function Sidebar() {
  const nodes = useNodes();
  const { fitView } = useMiniFlow();

  return (
    <aside>
      <div>{nodes.length} nodes</div>
      <button onClick={() => fitView()}>Fit</button>
    </aside>
  );
}
```

这时 `Sidebar` 不在 `GraphView` 里面，但它仍然能读 nodes、调用 `fitView`。

这就是 Provider 的价值：

```txt
它不是为了“省掉 props drilling”这么简单。
它是在声明：这些组件属于同一个 graph runtime。
```

再看受控模式：

```tsx
export function ControlledDemo() {
  const [nodes, setNodes, onNodesChange] = useMiniNodesState(initialNodes);
  const [edges, setEdges, onEdgesChange] = useMiniEdgesState(initialEdges);

  return (
    <MiniFlow
      nodes={nodes}
      edges={edges}
      onNodesChange={onNodesChange}
      onEdgesChange={onEdgesChange}
      onConnect={(connection) => {
        setEdges((eds) => addEdge(connection, eds));
      }}
    />
  );
}
```

这里 mini-flow 不能擅自长期持有最终 nodes。用户传进来的 `nodes` 才是事实来源。

但它仍然需要内部 store。因为交互过程中还要维护：

```txt
transform
connection
nodeLookup
edgeLookup
dragging
selection
options
callbacks
```

这就是容易误解的地方：

> controlled 不等于没有内部 store。controlled 只是说明用户数据的最终提交权在外部。

---

## 3. 核心概念解释

这一篇会出现五个概念。

它们容易混在一起，所以先分清楚。

### 3.1 Store：运行时状态容器

store 保存的是整个 flow runtime 的状态和动作：

```ts
type MiniFlowState = {
  nodes: MiniNode[];
  edges: MiniEdge[];
  nodeLookup: Map<string, MiniNode>;
  edgeLookup: Map<string, MiniEdge>;
  viewport: Viewport;
  connection: MiniConnectionState | null;
  hasDefaultNodes: boolean;
  hasDefaultEdges: boolean;
  onNodesChange?: OnNodesChange;
  onEdgesChange?: OnEdgesChange;
  onConnect?: OnConnect;
  snapGrid: [number, number];
  minZoom: number;
  maxZoom: number;

  setNodes(nodes: MiniNode[]): void;
  setEdges(edges: MiniEdge[]): void;
  setViewport(viewport: Viewport): void;
  updateConnection(connection: MiniConnectionState | null): void;
  triggerNodeChanges(changes: NodeChange[]): void;
  triggerEdgeChanges(changes: EdgeChange[]): void;
};
```

注意这里既有数据，也有 action。

React Flow 的 store 也是这个思路。`packages/react/src/store/index.ts` 里用 `createWithEqualityFn` 创建 Zustand store，然后把 `setNodes`、`setEdges`、`updateNodePositions`、`triggerNodeChanges`、`panBy`、`setCenter`、`cancelConnection`、`updateConnection` 等动作都放在同一个状态容器里。

这说明 store 不是一个“数据袋子”。它是运行时的控制面。

### 3.2 Provider：把同一个 store 交给一棵组件树

Provider 的职责很单纯：

```txt
创建一次 store
  ↓
放进 React Context
  ↓
让子组件通过 hooks 访问
```

React Flow 的 `ReactFlowProvider` 在 `packages/react/src/components/ReactFlowProvider/index.tsx` 里做的就是这件事：

```tsx
const [store] = useState(() =>
  createStore({
    nodes,
    edges,
    defaultNodes,
    defaultEdges,
    width,
    height,
    fitView,
    nodeOrigin,
    nodeExtent,
  })
);

return (
  <Provider value={store}>
    <BatchProvider>{children}</BatchProvider>
  </Provider>
);
```

这里的重点是：

```txt
store 用 useState 初始化
```

这表示它只在 Provider 首次挂载时创建一次，不会因为父组件重新 render 就重建。

一个图编辑器 runtime 最怕的就是“每次 render 都换一个 store”。那会导致订阅、交互状态、panZoom 实例、连接中间态全部丢失。

### 3.3 Wrapper：决定要不要创建 Provider

React Flow 的 `Wrapper` 负责一个很实用的判断：

```txt
如果外层已经有 StoreContext
  -> 直接使用现有 store

如果外层没有 StoreContext
  -> 自动包 ReactFlowProvider
```

源码入口在：

```txt
packages/react/src/container/ReactFlow/Wrapper.tsx
```

它先读：

```tsx
const isWrapped = useContext(StoreContext);
```

如果已经有 store，就直接返回 children。否则才创建 `ReactFlowProvider`。

这解释了为什么 React Flow 既可以这样用：

```tsx
<ReactFlow />
```

也可以这样用：

```tsx
<ReactFlowProvider>
  <ReactFlow />
</ReactFlowProvider>
```

mini-flow 也需要这个能力。否则用户一旦想让外部 Sidebar 访问 flow runtime，就会陷入“到底谁创建 store”的问题。

### 3.4 StoreUpdater：把 props 同步进 store

Provider 只负责创建 store。

但 React props 是会变化的。

例如：

```tsx
<ReactFlow nodes={nodes} onNodesChange={onNodesChange} minZoom={0.2} />
```

后续 `nodes`、`onNodesChange`、`minZoom` 都可能变。

如果 store 只在 Provider 初始化时读一次 props，那么它很快就会过期。

所以 React Flow 有一个独立组件：

```txt
packages/react/src/components/StoreUpdater/index.tsx
```

它追踪一组字段：

```txt
nodes
edges
defaultNodes
defaultEdges
onConnect
onNodesChange
onEdgesChange
minZoom
maxZoom
nodeExtent
snapGrid
...
```

当这些 props 变化时，它会调用对应 setter：

```txt
nodes -> setNodes
edges -> setEdges
minZoom -> setMinZoom
maxZoom -> setMaxZoom
其他字段 -> store.setState
```

这层非常关键。

它把两个世界分开了：

```txt
React props 世界
  由外部 render 驱动

React Flow runtime 世界
  由 store 和 interaction 驱动
```

`StoreUpdater` 就是两者之间的同步桥。

### 3.5 Hooks：给不同读者提供不同粒度的入口

React Flow 的 hooks 不是随便堆出来的。

它们分成几种层级。

第一层是底层逃生口：

```ts
useStore(selector, equalityFn)
useStoreApi()
```

`useStore` 从 `StoreContext` 里拿 Zustand store，再调用 `useStoreWithEqualityFn`。它允许用户自己写 selector。

第二层是常用状态读取：

```ts
useNodes()
useEdges()
useViewport()
```

它们只是特定 selector 的包装。例如 `useViewport` 把内部 `transform: [x, y, zoom]` 转成：

```ts
{
  x,
  y,
  zoom,
}
```

第三层是命令式实例：

```ts
useReactFlow()
```

它返回的是一组操作 flow 的 helper：

```txt
getNodes
getEdges
setNodes
setEdges
addNodes
addEdges
getNode
getEdge
fitView
screenToFlowPosition
flowToScreenPosition
...
```

这三层对应三类读者：

```txt
组件只想显示 nodes 数量
  -> useNodes

插件想做 zoom in / fitView
  -> useReactFlow

高级用户想读内部状态切片
  -> useStore(selector)
```

mini-flow 不需要一次做完 React Flow 的全部 hooks，但要把这个分层建立起来。

---

## 4. 源码入口在哪里

这一篇对应的 React Flow 源码入口主要有这些：

```txt
packages/react/src/container/ReactFlow/Wrapper.tsx
packages/react/src/components/ReactFlowProvider/index.tsx
packages/react/src/store/index.ts
packages/react/src/store/initialState.ts
packages/react/src/components/StoreUpdater/index.tsx
packages/react/src/hooks/useStore.ts
packages/react/src/hooks/useNodes.ts
packages/react/src/hooks/useEdges.ts
packages/react/src/hooks/useViewport.ts
packages/react/src/hooks/useReactFlow.ts
packages/react/src/hooks/useNodesEdgesState.ts
```

它们分别回答不同问题。

| 源码文件 | 回答的问题 |
| --- | --- |
| `Wrapper.tsx` | 如果用户没有显式 Provider，ReactFlow 怎么自动创建 store？ |
| `ReactFlowProvider/index.tsx` | store 在哪里创建，怎么通过 Context 传给子树？ |
| `store/initialState.ts` | 初始 nodes、edges、lookup、transform、fitView 状态怎么生成？ |
| `store/index.ts` | store 里有哪些数据和 action？ |
| `StoreUpdater/index.tsx` | 外部 props 变化后，怎么同步进内部 store？ |
| `useStore.ts` | hook 怎么拿到内部 store，怎么订阅状态切片？ |
| `useNodes.ts` / `useEdges.ts` / `useViewport.ts` | 常用 hooks 如何包装 selector？ |
| `useReactFlow.ts` | 命令式实例 API 如何从 store 和 viewport helper 组合出来？ |
| `useNodesEdgesState.ts` | controlled 示例里的 nodes / edges helper 怎么写？ |

读这些文件时，不要把重点放在“Zustand API 怎么用”。

重点应该放在边界：

```txt
谁负责创建 store？
谁负责同步 props？
谁负责读状态？
谁负责写状态？
谁负责把内部变化交还给用户？
```

这几个问题回答清楚了，React Flow 的 runtime 外形就清楚了。

---

## 5. 源码调用链

把这些源码入口串起来，可以得到这条调用链：

```txt
<ReactFlow nodes={nodes} edges={edges} ...>
  ↓
ReactFlow
  ↓
Wrapper
  ↓
如果外面没有 StoreContext：
  ReactFlowProvider
    ↓
    createStore(...)
      ↓
      getInitialState(...)
        ↓
        adoptUserNodes(...)
        updateConnectionLookup(...)
        getViewportForBounds(...) optional
  ↓
StoreUpdater
  ↓
把 props / callbacks / options 同步进 store
  ↓
GraphView / NodeRenderer / EdgeRenderer / ConnectionLine
  ↓
useStore(selector)
  ↓
interaction actions
  ↓
triggerNodeChanges / triggerEdgeChanges / updateConnection / setViewport
  ↓
controlled or uncontrolled branch
```

这条链路最值得注意的是两个方向。

第一个方向是“外部进入内部”：

```txt
props
  -> StoreUpdater
  -> store
```

用户传入的 `nodes`、`edges`、`callbacks`、`options` 不会神奇地自动变成内部状态。它们需要被 `StoreUpdater` 明确同步。

第二个方向是“内部回到外部”：

```txt
interaction
  -> changes
  -> triggerNodeChanges / triggerEdgeChanges
  -> onNodesChange / onEdgesChange
```

拖拽、选择、删除这些交互不会直接修改用户手里的数组。它们先变成 change objects，再交给 controlled / uncontrolled 分支处理。

这就是 React Flow 状态设计里最重要的一条主线：

```txt
外部 props 是输入
store 是运行时
changes 是交互结果
callbacks 是回流出口
```

mini-flow 的实现只要抓住这条线，就不会写散。

---

## 6. 关键数据结构

我们先设计 mini-flow 的 store 类型。

这不是最终生产代码，而是为了把 React Flow 的运行时边界复刻出来。

```ts
export type Viewport = {
  x: number;
  y: number;
  zoom: number;
};

export type MiniNode = {
  id: string;
  position: { x: number; y: number };
  width?: number;
  height?: number;
  selected?: boolean;
  dragging?: boolean;
  data?: unknown;
};

export type MiniEdge = {
  id: string;
  source: string;
  target: string;
  sourceHandle?: string | null;
  targetHandle?: string | null;
};

export type MiniConnection = {
  source: string;
  target: string;
  sourceHandle?: string | null;
  targetHandle?: string | null;
};

export type MiniConnectionState = {
  fromNodeId: string;
  fromHandleId: string | null;
  pointer: { x: number; y: number };
  candidate: MiniConnection | null;
  valid: boolean;
};
```

然后是变化对象。

为了保持文章聚焦，这里只保留最小版本：

```ts
export type NodeChange =
  | { type: "position"; id: string; position: { x: number; y: number }; dragging?: boolean }
  | { type: "select"; id: string; selected: boolean }
  | { type: "remove"; id: string };

export type EdgeChange =
  | { type: "select"; id: string; selected: boolean }
  | { type: "remove"; id: string };
```

再看 store state。

```ts
export type MiniFlowState = {
  nodes: MiniNode[];
  edges: MiniEdge[];

  nodeLookup: Map<string, MiniNode>;
  edgeLookup: Map<string, MiniEdge>;

  viewport: Viewport;
  connection: MiniConnectionState | null;

  hasDefaultNodes: boolean;
  hasDefaultEdges: boolean;

  onNodesChange?: (changes: NodeChange[]) => void;
  onEdgesChange?: (changes: EdgeChange[]) => void;
  onConnect?: (connection: MiniConnection) => void;

  snapGrid: [number, number];
  minZoom: number;
  maxZoom: number;

  setNodes(nodes: MiniNode[]): void;
  setEdges(edges: MiniEdge[]): void;
  setViewport(viewport: Viewport): void;
  updateConnection(connection: MiniConnectionState | null): void;
  triggerNodeChanges(changes: NodeChange[]): void;
  triggerEdgeChanges(changes: EdgeChange[]): void;
};
```

这里有两个 lookup：

```txt
nodes array
  适合对外 API 和 React 渲染列表

nodeLookup map
  适合交互系统按 id 高频查询
```

这和 React Flow 的真实源码一致。

`initialState.ts` 会创建 `nodeLookup`、`edgeLookup`、`connectionLookup`、`parentLookup`，然后把用户节点交给 `adoptUserNodes` 转成内部可操作的节点结构。

mini-flow 暂时不做完整 `InternalNode`，但先保留 lookup 的习惯。

因为从第 22 篇的拖拽开始，我们已经遇到这个问题：

```txt
drag move 时根据 id 找节点
  如果每次 nodes.find(...)
  高频交互里会越来越别扭
```

lookup 不是性能洁癖，而是交互运行时的基础设施。

---

## 7. 关键实现思路

我们分五步实现。

```txt
1. createMiniStore
2. MiniFlowProvider
3. MiniFlowWrapper
4. MiniStoreUpdater
5. hooks
```

### 7.1 createMiniStore：最小外部 store

React Flow 使用 Zustand。

mini-flow 可以直接装 Zustand，但为了看清机制，这里先写一个很小的 external store：

```ts
type Listener = () => void;

export type MiniStoreApi = {
  getState(): MiniFlowState;
  setState(
    partial:
      | Partial<MiniFlowState>
      | ((state: MiniFlowState) => Partial<MiniFlowState>)
  ): void;
  subscribe(listener: Listener): () => void;
};

function createNodeLookup(nodes: MiniNode[]) {
  return new Map(nodes.map((node) => [node.id, node]));
}

function createEdgeLookup(edges: MiniEdge[]) {
  return new Map(edges.map((edge) => [edge.id, edge]));
}

export function createMiniStore(initial: MiniFlowInit): MiniStoreApi {
  const listeners = new Set<Listener>();

  let state: MiniFlowState;

  const api: MiniStoreApi = {
    getState: () => state,
    setState: (partial) => {
      const patch =
        typeof partial === "function" ? partial(state) : partial;

      state = {
        ...state,
        ...patch,
      };

      listeners.forEach((listener) => listener());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };

  const initialNodes = initial.defaultNodes ?? initial.nodes ?? [];
  const initialEdges = initial.defaultEdges ?? initial.edges ?? [];

  state = createInitialState(api, {
    ...initial,
    nodes: initialNodes,
    edges: initialEdges,
    hasDefaultNodes: initial.defaultNodes !== undefined,
    hasDefaultEdges: initial.defaultEdges !== undefined,
  });

  return api;
}
```

这里有一个小细节：

```txt
createInitialState 需要拿到 api
```

因为 action 里要调用 `api.getState()` 和 `api.setState()`。

这和 Zustand 的写法类似：

```ts
createWithEqualityFn((set, get) => ({
  nodes: [],
  setNodes: (nodes) => {
    set({ nodes });
  },
}));
```

只不过 mini-flow 用更朴素的方式把 `set` / `get` 展开了。

### 7.2 createInitialState：初始化数据和 action

```ts
type MiniFlowInit = {
  nodes?: MiniNode[];
  edges?: MiniEdge[];
  defaultNodes?: MiniNode[];
  defaultEdges?: MiniEdge[];
  onNodesChange?: (changes: NodeChange[]) => void;
  onEdgesChange?: (changes: EdgeChange[]) => void;
  onConnect?: (connection: MiniConnection) => void;
  snapGrid?: [number, number];
  minZoom?: number;
  maxZoom?: number;
};

function createInitialState(
  api: MiniStoreApi,
  init: MiniFlowInit & {
    nodes: MiniNode[];
    edges: MiniEdge[];
    hasDefaultNodes: boolean;
    hasDefaultEdges: boolean;
  }
): MiniFlowState {
  const setNodes = (nodes: MiniNode[]) => {
    api.setState({
      nodes,
      nodeLookup: createNodeLookup(nodes),
    });
  };

  const setEdges = (edges: MiniEdge[]) => {
    api.setState({
      edges,
      edgeLookup: createEdgeLookup(edges),
    });
  };

  const triggerNodeChanges = (changes: NodeChange[]) => {
    const state = api.getState();

    if (state.hasDefaultNodes) {
      setNodes(applyNodeChanges(changes, state.nodes));
    }

    state.onNodesChange?.(changes);
  };

  const triggerEdgeChanges = (changes: EdgeChange[]) => {
    const state = api.getState();

    if (state.hasDefaultEdges) {
      setEdges(applyEdgeChanges(changes, state.edges));
    }

    state.onEdgesChange?.(changes);
  };

  return {
    nodes: init.nodes,
    edges: init.edges,
    nodeLookup: createNodeLookup(init.nodes),
    edgeLookup: createEdgeLookup(init.edges),
    viewport: { x: 0, y: 0, zoom: 1 },
    connection: null,
    hasDefaultNodes: init.hasDefaultNodes,
    hasDefaultEdges: init.hasDefaultEdges,
    onNodesChange: init.onNodesChange,
    onEdgesChange: init.onEdgesChange,
    onConnect: init.onConnect,
    snapGrid: init.snapGrid ?? [1, 1],
    minZoom: init.minZoom ?? 0.5,
    maxZoom: init.maxZoom ?? 2,
    setNodes,
    setEdges,
    setViewport: (viewport) => api.setState({ viewport }),
    updateConnection: (connection) => api.setState({ connection }),
    triggerNodeChanges,
    triggerEdgeChanges,
  };
}
```

这段代码有一个很重要的分支：

```ts
if (state.hasDefaultNodes) {
  setNodes(applyNodeChanges(changes, state.nodes));
}

state.onNodesChange?.(changes);
```

这就是 uncontrolled 和 controlled 的分界。

如果用户传的是 `defaultNodes`，mini-flow 可以内部应用 changes。

如果用户传的是 `nodes`，mini-flow 只触发 `onNodesChange`，由用户决定怎么更新外部 nodes。

React Flow 的 `triggerNodeChanges` 也是这个思路：当存在 default nodes 时，内部先应用 changes；随后仍然调用用户传入的 `onNodesChange`。

这让两种模式共享同一套交互系统：

```txt
XYDrag / selection / delete
  -> NodeChange[]
  -> triggerNodeChanges
  -> controlled or uncontrolled
```

### 7.3 MiniFlowProvider：稳定创建一次 store

接下来实现 Provider。

```tsx
const MiniStoreContext = createContext<MiniStoreApi | null>(null);

type MiniFlowProviderProps = MiniFlowInit & {
  children: ReactNode;
};

export function MiniFlowProvider({
  children,
  ...initial
}: MiniFlowProviderProps) {
  const [store] = useState(() => createMiniStore(initial));

  return (
    <MiniStoreContext.Provider value={store}>
      {children}
    </MiniStoreContext.Provider>
  );
}
```

这里不要写成：

```tsx
const store = createMiniStore(initial);
```

因为这会导致每次 render 都创建新 store。

React Flow 的 `ReactFlowProvider` 用 `useState(() => createStore(...))` 正是为了避免这个问题。

Provider 的职责很克制：

```txt
只创建 store
只提供 Context
不在这里同步后续 props
```

后续 props 同步交给 `MiniStoreUpdater`。

### 7.4 MiniFlowWrapper：自动 Provider 包装

为了让用户既可以显式写 Provider，也可以直接用 `MiniFlow`，我们加一个 Wrapper：

```tsx
function MiniFlowWrapper({
  children,
  ...props
}: MiniFlowProviderProps) {
  const existingStore = useContext(MiniStoreContext);

  if (existingStore) {
    return <>{children}</>;
  }

  return (
    <MiniFlowProvider {...props}>
      {children}
    </MiniFlowProvider>
  );
}
```

然后 `MiniFlow` 内部结构就可以写成：

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

这对应 React Flow 的主组件结构：

```txt
ReactFlow
  -> Wrapper
    -> StoreUpdater
    -> GraphView
    -> children
```

注意 `children` 的位置。

插件组件必须在 Provider 内部，否则它们读不到 store。

这就是为什么第 17 篇说插件组件不是魔法：

```txt
Controls / MiniMap / Background / Panel
  本质上都是读取同一个 runtime store 的子组件
```

### 7.5 MiniStoreUpdater：把 props 同步到 store

Provider 初始化只发生一次。

所以 `MiniFlow` 的 props 更新，需要另一个组件负责同步。

```tsx
function MiniStoreUpdater(props: MiniFlowProps) {
  const store = useMiniStoreApi();

  useEffect(() => {
    if (props.nodes) {
      store.getState().setNodes(props.nodes);
    }
  }, [props.nodes, store]);

  useEffect(() => {
    if (props.edges) {
      store.getState().setEdges(props.edges);
    }
  }, [props.edges, store]);

  useEffect(() => {
    store.setState({
      onNodesChange: props.onNodesChange,
      onEdgesChange: props.onEdgesChange,
      onConnect: props.onConnect,
      snapGrid: props.snapGrid ?? [1, 1],
      minZoom: props.minZoom ?? 0.5,
      maxZoom: props.maxZoom ?? 2,
    });
  }, [
    props.onNodesChange,
    props.onEdgesChange,
    props.onConnect,
    props.snapGrid,
    props.minZoom,
    props.maxZoom,
    store,
  ]);

  return null;
}
```

这只是最小版本。

React Flow 的 `StoreUpdater` 更细：

```txt
它维护一组 reactFlowFieldsToTrack
它记录 previousFields
它按字段选择专用 setter 或 store.setState
它在 mount 时处理 defaultNodes / defaultEdges
它在 unmount 时 reset
```

mini-flow 不需要立刻复刻所有细节，但要保留这个设计位置。

因为没有 `StoreUpdater` 的话，Provider 很容易变成一个什么都管的组件：

```txt
创建 store
同步 props
处理 callbacks
处理 options
处理 controlled/uncontrolled
```

这会让边界重新混乱。

Provider 负责“有一个 runtime”。

StoreUpdater 负责“这个 runtime 接收当前 props”。

### 7.6 useStoreApi：直接拿 store API

先实现底层 API。

```tsx
export function useMiniStoreApi() {
  const store = useContext(MiniStoreContext);

  if (!store) {
    throw new Error(
      "useMiniStoreApi must be used inside <MiniFlowProvider /> or <MiniFlow />."
    );
  }

  return store;
}
```

这对应 React Flow 的 `useStoreApi()`。

React Flow 返回的是：

```ts
{
  getState: store.getState,
  setState: store.setState,
  subscribe: store.subscribe,
}
```

mini-flow 可以直接返回 store 本身。

### 7.7 useStore：订阅状态切片

再实现 selector hook。

```tsx
export function useMiniStore<T>(
  selector: (state: MiniFlowState) => T
): T {
  const store = useMiniStoreApi();

  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState())
  );
}
```

这个版本有一个限制：

```txt
它没有 equalityFn
```

也就是说，只要 store 通知订阅者，selector 返回的新对象就可能导致组件重新 render。

React Flow 的 `useStore` 接收 `equalityFn`，内部调用 Zustand 的 `useStoreWithEqualityFn`：

```ts
useStore(selector, equalityFn)
```

这也是为什么 `useNodes`、`useEdges`、`useViewport` 会配合 `shallow` 使用。

mini-flow 暂时保留最小版本，下一步如果要优化性能，可以把 API 扩展成：

```ts
useMiniStore(selector, equalityFn)
```

或者直接改用 Zustand。

重点不是“自己造一个更好的 Zustand”，而是理解：

```txt
hooks 订阅的不是整个 store
hooks 订阅的是 selector 选择出的状态切片
```

这正是第 18 篇讲性能设计时反复强调的事情。

### 7.8 useNodes、useEdges、useViewport：常用 selector 包装

有了 `useMiniStore`，常用 hooks 就很薄了。

```tsx
export function useNodes() {
  return useMiniStore((state) => state.nodes);
}

export function useEdges() {
  return useMiniStore((state) => state.edges);
}

export function useViewport() {
  return useMiniStore((state) => state.viewport);
}
```

这对应 React Flow 的：

```txt
packages/react/src/hooks/useNodes.ts
packages/react/src/hooks/useEdges.ts
packages/react/src/hooks/useViewport.ts
```

真实源码里 `useViewport` 不是直接读 `viewport` 字段，而是从 `transform` 派生：

```ts
const viewportSelector = (state) => ({
  x: state.transform[0],
  y: state.transform[1],
  zoom: state.transform[2],
});
```

这说明内部数据结构和公开 hook 结果不必完全一样。

内部为了渲染层方便，可以存 `[x, y, zoom]`。

公开 API 为了用户理解，可以返回 `{ x, y, zoom }`。

mini-flow 暂时内部也用 `{ x, y, zoom }`，所以 hook 更简单。

### 7.9 useMiniFlow：命令式实例

最后实现类似 `useReactFlow()` 的 hook。

```tsx
export function useMiniFlow() {
  const store = useMiniStoreApi();

  return useMemo(
    () => ({
      getNodes: () => store.getState().nodes.map((node) => ({ ...node })),
      getEdges: () => store.getState().edges.map((edge) => ({ ...edge })),
      getViewport: () => store.getState().viewport,

      setNodes: (nodes: MiniNode[]) => {
        store.getState().setNodes(nodes);
      },

      setEdges: (edges: MiniEdge[]) => {
        store.getState().setEdges(edges);
      },

      setViewport: (viewport: Viewport) => {
        store.getState().setViewport(viewport);
      },

      addEdges: (edgesToAdd: MiniEdge | MiniEdge[]) => {
        const nextEdges = Array.isArray(edgesToAdd)
          ? edgesToAdd
          : [edgesToAdd];

        const state = store.getState();
        state.setEdges([...state.edges, ...nextEdges]);
      },

      screenToFlowPosition: (point: { x: number; y: number }) => {
        const { viewport } = store.getState();

        return {
          x: (point.x - viewport.x) / viewport.zoom,
          y: (point.y - viewport.y) / viewport.zoom,
        };
      },

      flowToScreenPosition: (point: { x: number; y: number }) => {
        const { viewport } = store.getState();

        return {
          x: point.x * viewport.zoom + viewport.x,
          y: point.y * viewport.zoom + viewport.y,
        };
      },
    }),
    [store]
  );
}
```

这个 hook 不是为了渲染，而是为了“操作 flow”。

例如 Controls 可以这样写：

```tsx
function MiniControls() {
  const { getViewport, setViewport } = useMiniFlow();

  return (
    <div className="mini-flow__controls">
      <button
        onClick={() => {
          const viewport = getViewport();
          setViewport({ ...viewport, zoom: viewport.zoom * 1.2 });
        }}
      >
        +
      </button>
      <button
        onClick={() => {
          const viewport = getViewport();
          setViewport({ ...viewport, zoom: viewport.zoom / 1.2 });
        }}
      >
        -
      </button>
    </div>
  );
}
```

React Flow 的 `useReactFlow()` 更复杂。

它会组合：

```txt
store api
viewport helper
batch context
viewport initialized state
```

源码里 `setNodes` 和 `setEdges` 不是直接 `store.setState`，而是进入 batch queue。`fitView` 也不是马上计算完，而是设置 `fitViewQueued`，再推动 nodes 队列触发后续处理。

mini-flow 先不实现 batch，但要知道这是生产级库需要补上的层。

因为真实场景里，用户可能连续调用：

```ts
reactFlow.setNodes(...)
reactFlow.setEdges(...)
reactFlow.fitView()
```

如果每次都同步重算布局、测量、viewport，代价会很高。

---

## 8. 这部分源码的设计取舍

这一层的设计看起来“工程味”很重，但它解决的是节点编辑器库的根问题。

### 8.1 为什么不只用 React state？

React state 适合组件自己的 UI 状态。

但 React Flow 的状态不是普通 UI 状态。

它要被这些对象共享：

```txt
React components
hooks
plugins
drag controller
panzoom controller
connection controller
keyboard listener
selection listener
```

其中很多逻辑不是简单的“render 时读一次”。

它们需要在事件回调里同步读取最新状态。

所以 React Flow 选择 Zustand 这类 external store，是合理的。

收益是：

```txt
交互系统可以 getState / setState
React 组件可以 selector subscribe
插件可以共享同一 runtime
controlled/uncontrolled 可以统一回流
```

代价是：

```txt
状态边界更复杂
内部 action 更多
用户可能通过 useStore 依赖内部结构
需要小心 selector 和 equalityFn，否则容易重渲染
```

这就是为什么 React Flow 文档和源码都会鼓励：

```txt
常规场景用 useNodes / useEdges / useViewport / useReactFlow
高级场景再用 useStore
```

### 8.2 为什么 Provider 和 StoreUpdater 要分开？

因为它们处理的是两个不同时间点的问题。

Provider 处理：

```txt
组件树首次进入 flow runtime 时，创建 store。
```

StoreUpdater 处理：

```txt
React props 后续变化时，同步到已有 store。
```

如果混在一起，很容易写出这种代码：

```tsx
function MiniFlowProvider(props) {
  const [store] = useState(() => createMiniStore(props));

  useEffect(() => {
    store.setState(props);
  }, [props]);

  return <Provider value={store}>{props.children}</Provider>;
}
```

表面上也能跑，但边界变模糊了。

更糟的是，Provider 可能被用在没有 `MiniFlow` props 的外层：

```tsx
<MiniFlowProvider>
  <Sidebar />
  <MiniFlow nodes={nodes} />
</MiniFlowProvider>
```

这时真正的 props 来自 `MiniFlow`，而不是 Provider。

所以 React Flow 让 `Wrapper`、`Provider`、`StoreUpdater` 各守一段边界：

```txt
Wrapper
  决定是否需要 Provider

Provider
  创建并提供 store

StoreUpdater
  把当前 ReactFlow props 同步进 store
```

这就是工程里很值得借鉴的地方：不是所有逻辑都塞进“最上层组件”。

### 8.3 为什么 hooks 要分层？

如果只暴露一个：

```ts
useStore()
```

用户当然什么都能做。

但这会把内部结构直接暴露给所有人。

如果只暴露：

```ts
useReactFlow()
```

用户又很难写轻量组件，比如只显示当前 zoom。

React Flow 的做法是分层：

```txt
useNodes / useEdges / useViewport
  面向常见读取

useReactFlow
  面向命令式操作

useStore / useStoreApi
  面向高级逃生口
```

这个设计的好处是：

```txt
新手不必理解内部 store
插件作者能拿到足够能力
高级用户仍然有底层入口
```

代价是 API 面会变宽。

库作者要持续维护这些 hook 的语义边界。

### 8.4 为什么 controlled helper 不放进 store？

`useNodesState` 和 `useEdgesState` 看起来像 store 的一部分，但它们其实不是。

它们只是用户侧的便捷 helper：

```tsx
const [nodes, setNodes, onNodesChange] = useNodesState(initialNodes);
```

源码里它基本就是：

```ts
const [nodes, setNodes] = useState(initialNodes);
const onNodesChange = useCallback(
  (changes) => setNodes((nds) => applyNodeChanges(changes, nds)),
  []
);
```

这说明 controlled 模式的事实来源在用户组件里。

React Flow 内部 store 负责产生 changes。

用户侧 helper 负责应用 changes。

这条边界不要反过来。

如果把 controlled helper 塞进内部 store，就会把“用户拥有数据”这件事稀释掉。

---

## 9. 如果我们自己实现，最小版本应该怎么写

这一节把前面的片段合成一个更完整的 mini-flow 运行时骨架。

### 9.1 runtime store

```tsx
import {
  createContext,
  useContext,
  useEffect,
  useMemo,
  useState,
  useSyncExternalStore,
  type ReactNode,
} from "react";

const MiniStoreContext = createContext<MiniStoreApi | null>(null);

export function MiniFlowProvider({
  children,
  ...initial
}: MiniFlowInit & { children: ReactNode }) {
  const [store] = useState(() => createMiniStore(initial));

  return (
    <MiniStoreContext.Provider value={store}>
      {children}
    </MiniStoreContext.Provider>
  );
}

export function useMiniStoreApi() {
  const store = useContext(MiniStoreContext);

  if (!store) {
    throw new Error("MiniFlow hooks must be used inside a MiniFlow runtime.");
  }

  return store;
}

export function useMiniStore<T>(
  selector: (state: MiniFlowState) => T
) {
  const store = useMiniStoreApi();

  return useSyncExternalStore(
    store.subscribe,
    () => selector(store.getState()),
    () => selector(store.getState())
  );
}
```

这个骨架先解决最核心的问题：

```txt
所有 runtime 状态从同一个 store 读写
React 组件通过 Context 找到 store
hook 通过 selector 订阅 store 切片
```

### 9.2 hooks

```tsx
export function useNodes() {
  return useMiniStore((state) => state.nodes);
}

export function useEdges() {
  return useMiniStore((state) => state.edges);
}

export function useViewport() {
  return useMiniStore((state) => state.viewport);
}

export function useMiniFlow() {
  const store = useMiniStoreApi();

  return useMemo(
    () => ({
      getNodes: () => store.getState().nodes.map((node) => ({ ...node })),
      getEdges: () => store.getState().edges.map((edge) => ({ ...edge })),
      getViewport: () => store.getState().viewport,
      setNodes: (nodes: MiniNode[]) => store.getState().setNodes(nodes),
      setEdges: (edges: MiniEdge[]) => store.getState().setEdges(edges),
      setViewport: (viewport: Viewport) =>
        store.getState().setViewport(viewport),
    }),
    [store]
  );
}
```

注意：

```txt
useNodes / useEdges / useViewport
  是订阅式读取

useMiniFlow
  是命令式操作入口
```

这两个不要混为一谈。

如果组件要随着 zoom 变化重新 render，用 `useViewport`。

如果按钮点击时临时读取当前 zoom，用 `useMiniFlow().getViewport()`。

### 9.3 MiniFlow 组件结构

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

function MiniFlowWrapper({
  children,
  ...props
}: MiniFlowProps & { children?: ReactNode }) {
  const existingStore = useContext(MiniStoreContext);

  if (existingStore) {
    return <>{children}</>;
  }

  return (
    <MiniFlowProvider
      nodes={props.nodes}
      edges={props.edges}
      defaultNodes={props.defaultNodes}
      defaultEdges={props.defaultEdges}
      onNodesChange={props.onNodesChange}
      onEdgesChange={props.onEdgesChange}
      onConnect={props.onConnect}
    >
      {children}
    </MiniFlowProvider>
  );
}
```

这时 mini-flow 的结构就接近 React Flow：

```txt
MiniFlow
  -> MiniFlowWrapper
    -> MiniFlowProvider optional
      -> MiniStoreUpdater
      -> MiniGraphView
      -> children plugins
```

这里的关键不是 JSX 长什么样。

关键是运行时归属变清楚了：

```txt
MiniGraphView
MiniControls
MiniMap
Background
Sidebar

只要在同一个 Provider 下
就属于同一个 mini-flow runtime
```

### 9.4 controlled helper

最后补上用户侧 helper。

```tsx
export function useMiniNodesState(initialNodes: MiniNode[]) {
  const [nodes, setNodes] = useState(initialNodes);

  const onNodesChange = useCallback((changes: NodeChange[]) => {
    setNodes((currentNodes) =>
      applyNodeChanges(changes, currentNodes)
    );
  }, []);

  return [nodes, setNodes, onNodesChange] as const;
}

export function useMiniEdgesState(initialEdges: MiniEdge[]) {
  const [edges, setEdges] = useState(initialEdges);

  const onEdgesChange = useCallback((changes: EdgeChange[]) => {
    setEdges((currentEdges) =>
      applyEdgeChanges(changes, currentEdges)
    );
  }, []);

  return [edges, setEdges, onEdgesChange] as const;
}
```

这对应 React Flow 的 `useNodesState` 和 `useEdgesState`。

它们不是运行时 store 的一部分，而是用户在 controlled 模式下管理外部数组的便利工具。

到这里，mini-flow 已经具备了一个清晰的状态闭环：

```txt
用户传入 nodes / defaultNodes
  ↓
MiniFlowProvider 初始化 store
  ↓
MiniStoreUpdater 同步后续 props
  ↓
GraphView / interaction / plugin 读写 store
  ↓
triggerNodeChanges / triggerEdgeChanges
  ↓
controlled: 用户 onChange 更新外部数据
uncontrolled: 内部 apply changes
```

这就是第 7 篇、第 14 篇、第 16 篇合在一起的实战验证。

---

## 10. 本篇总结

这一篇做的事情，看起来只是“加 store”。

但它真正补上的是 mini-flow 的运行时边界。

在这之前，mini-flow 已经能做一些事：

```txt
渲染节点和边
移动 viewport
拖拽节点
从 handle 创建连接
```

但这些能力还像散落的零件。

第 24 篇把它们收进一个统一结构：

```txt
MiniFlowProvider
  ↓
store
  ↓
StoreUpdater
  ↓
hooks
  ↓
renderers / interactions / plugins
```

对应 React Flow 源码，就是：

```txt
Wrapper
ReactFlowProvider
createStore / initialState
StoreUpdater
useStore / useStoreApi
useNodes / useEdges / useViewport
useReactFlow
useNodesState / useEdgesState
```

这一层最值得带走的不是某个 API，而是这条设计原则：

> 图编辑器库的 store 不只是保存数据，它是连接外部 props、内部交互、插件组件和用户回调的运行时控制面。

如果只把 store 当成“全局 useState”，就读不懂 React Flow。

如果把它当成交互运行时的中心，后面的 children 插件模型、性能优化和受控状态都会顺很多。

---

## 11. 下一篇读什么

下一篇进入：

```txt
第 25 篇：实战：实现 Controls、Background、MiniMap
```

有了第 24 篇的 store 和 hooks，插件组件就不再需要特殊待遇。

它们会变成同一种模式：

```txt
组件挂在 MiniFlow children 里
  ↓
通过 hooks 读取 store
  ↓
通过 useMiniFlow 调用 runtime 方法
  ↓
渲染自己的 UI
```

第 25 篇会用三个插件验证这件事：

```txt
Controls
  验证命令式 viewport 操作

Background
  验证 viewport transform 下的辅助渲染

MiniMap
  验证 nodes + viewport 的二级视图
```

也就是说，第 24 篇解决的是：

```txt
状态住在哪里？
```

第 25 篇解决的是：

```txt
插件怎么共享这套运行时？
```
