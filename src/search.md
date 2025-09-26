好的，我们来对 prosemirror-search 这个插件进行一次全面、深入的讲解。这个包提供了一套完整的、与编辑器状态紧密集成的查找和替换功能，是学习 ProseMirror 插件如何管理自身状态、如何使用 `Decoration` 以及如何与命令系统交互的绝佳案例。

我们将从核心的状态管理开始，逐步剖析其功能实现。

---

### 1. 核心状态管理：`SearchState` 与插件的 `state` 字段

ProseMirror 插件的一大核心是状态管理。prosemirror-search 将所有与搜索相关的信息都封装在 `SearchState` 类中。

```typescript
class SearchState {
  constructor(
    readonly query: SearchQuery, // 当前的查询对象 (包含搜索词、是否正则等)
    readonly range: { from: number; to: number } | null, // 搜索范围，null 表示整个文档
    readonly deco: DecorationSet // 用于高亮匹配项的 Decoration 集合
  ) {}
}
```

这个 `SearchState` 对象被存储在插件自己的状态字段中，通过 `PluginKey` (`searchKey`) 来唯一标识和访问。

#### `state` 字段的 `init` 和 `apply` 方法

这是插件状态管理的核心逻辑：

- **`init(_config, state)`**:

  - 在编辑器初始化时被调用一次。
  - 它会创建一个初始的 `SearchState`。
  - 它会从 `options` 中获取初始的查询 (`initialQuery`) 和范围 (`initialRange`)。
  - 然后调用 `buildMatchDeco` 来创建初始的高亮装饰。

- **`apply(tr, search, _oldState, state)`**:
  - 在**每次**有 `Transaction` (事务) 被应用时调用。它的职责是根据事务信息计算出**新的** `SearchState`。
  - **分支一：主动更新搜索状态**
    ```typescript
    let set = tr.getMeta(searchKey) as ...
    if (set)
      return new SearchState(set.query, set.range, buildMatchDeco(state, set.query, set.range))
    ```
    - 这是**外部命令**（如用户在搜索框输入新内容）更新搜索状态的方式。
    - 命令会通过 `tr.setMeta(searchKey, ...)` 将新的查询信息附加到事务中。
    - `apply` 方法检测到这个 `meta` 数据后，会用它来创建一个全新的 `SearchState`，并重新计算高亮。
  - **分支二：被动响应文档变更**
    ```typescript
    if (tr.docChanged || tr.selectionSet) {
      // ... 映射 range ...
      search = new SearchState(search.query, range, buildMatchDeco(state, search.query, range))
    }
    ```
    - 当文档内容发生变化 (`tr.docChanged`) 或选区变化 (`tr.selectionSet`) 时，插件需要被动地更新自己。
    - **映射范围 (`range`)**: 如果设置了搜索范围，当文档内容变化时，范围的坐标也需要随之调整。`tr.mapping.map()` 就是用来做这个坐标变换的。
    - **重新计算高亮**: 无论是文档变化还是选区变化，都可能影响高亮。例如，当前选中的匹配项应该有不同的样式 (`ProseMirror-active-search-match`)。因此，需要调用 `buildMatchDeco` 重新生成 `DecorationSet`。
  - **分支三：无变化**
    - 如果事务与搜索无关，`apply` 方法会直接返回旧的 `search` 状态实例，这是一种性能优化，避免了不必要的对象创建和计算。

---

### 2. 视觉反馈：`Decoration` 与 `props`

插件如何将计算出的高亮应用到视图上？答案是 `Decoration`。

- **`buildMatchDeco(...)`**:

  - 这个函数是高亮的核心。
  - 它使用 `query.findNext()` 遍历文档（或指定范围），找到所有匹配项。
  - 对于每个匹配项，它创建一个 `Decoration.inline`。
  - 它会检查匹配项的位置是否与当前选区完全重合，如果是，就赋予一个特殊的 `active` 类名。
  - 最后，它将所有 `Decoration` 放入一个 `DecorationSet` 中返回。`DecorationSet` 是 ProseMirror 用于高效管理和渲染大量装饰的数据结构。

- **`props: { decorations: ... }`**:
  - 这是插件将其状态“注入”到视图的桥梁。
  - `decorations` 是一个插件 `prop`，它接收一个函数，该函数返回一个 `DecorationSet`。
  - `state => searchKey.getState(state)!.deco`: 这个函数从当前的总状态 (`state`) 中，通过 `searchKey` 获取到 `SearchState`，然后返回其 `deco` 属性。
  - ProseMirror 的 `EditorView` 会自动收集所有插件提供的 `decorations`，并将它们高效地渲染到 DOM 上。

---

### 3. 用户交互：命令系统 (`find`, `replace`)

插件提供了一系列命令，供用户绑定到快捷键或菜单按钮上，以执行查找和替换操作。

#### `findCommand(wrap, dir)`: 查找命令的工厂

这是一个高阶函数，用于创建 `findNext` 和 `findPrev` 等命令。

- 它返回一个标准的 ProseMirror `Command` 函数 `(state, dispatch) => boolean`。
- **逻辑**:
  1.  获取当前的 `SearchState`。
  2.  根据方向 `dir` 和是否循环 `wrap`，调用 `nextMatch` 或 `prevMatch` 辅助函数来找到下一个匹配结果的位置。
  3.  如果找到了匹配项 (`next`)，并且 `dispatch` 函数存在（意味着不只是检查命令是否可用，而是要实际执行），则：
      - 创建一个新的 `TextSelection` 来选中匹配项。
      - 将这个选区设置到一个新的事务中。
      - 调用 `tr.scrollIntoView()` 确保选中的匹配项在视口中可见。
      - 通过 `dispatch(tr)` 应用这个事务。

#### `replaceCommand(wrap, moveForward)`: 替换命令的工厂

这是创建 `replaceNext` 和 `replaceCurrent` 的工厂函数，逻辑更复杂。

- **逻辑**:
  1.  找到下一个匹配项 `next`。
  2.  **情况一：当前已选中一个匹配项** (`state.selection.from == next.from`)
      - 获取替换文本/节点 (`query.getReplacements`)。
      - 创建一个事务 `tr`，并使用 `tr.replace()` 执行替换操作。
      - **如果 `moveForward` 为 `true`**，则继续查找下一个匹配项 `after`，并将选区移动到 `after` 的位置。注意这里需要用 `tr.mapping.map()` 来计算 `after` 在**替换后**的新坐标。
      - **如果 `moveForward` 为 `false`** (`replaceCurrent`)，则将选区设置在刚刚被替换内容之后的位置。
        . `dispatch` 这个包含了替换和选区移动的事务。
  3.  **情况二：当前未选中匹配项**
      - 如果 `moveForward` 为 `true`，则只执行查找操作，将选区移动到下一个匹配项，而不进行替换。
      - 如果 `moveForward` 为 `false`，则什么都不做。

#### `replaceAll`: 替换所有

- 这个命令的逻辑相对直接：
  1.  循环调用 `query.findNext()` 找到**所有**的匹配项，并将它们存入一个数组。
  2.  创建一个事务 `tr`。
  3.  **从后往前**遍历匹配项数组，并对每个匹配项执行 `tr.replace()`。从后往前是为了避免替换操作影响到尚未处理的匹配项的坐标。
  4.  `dispatch` 这个包含了所有替换操作的事务。

---

### 4. 外部 API

插件还导出了一些辅助函数，方便其他代码与搜索状态交互：

- **`getSearchState(state)`**: 安全地获取当前搜索状态。
- **`setSearchState(tr, query, range)`**: 这是更新搜索状态的**唯一正确方式**。它不是直接修改状态，而是将更新意图作为 `meta` 数据附加到事务上，由插件的 `apply` 方法来处理。这保证了数据流的单向和可预测性。

### 总结

prosemirror-search 是一个设计精良的 ProseMirror 插件，它完美地展示了：

1.  **状态封装**: 将所有相关状态（查询、范围、高亮）封装在 `SearchState` 中，并通过插件的 `state` 字段进行管理。
2.  **单向数据流**: 状态的变更只能通过派发一个带有 `meta` 数据的事务来触发，`apply` 方法根据事务信息计算新状态，视图根据新状态进行渲染。
3.  **声明式视图**: 使用 `Decoration` 来“声明”你想要的高亮效果，而不是手动操作 DOM。ProseMirror 负责高效地将这些声明渲染出来。
4.  **命令集成**: 将核心逻辑封装成标准的 ProseMirror `Command`，可以轻松地与快捷键、菜单等 UI 组件集成。
5.  **坐标映射**: 在处理文档变更时，熟练地使用 `Transaction.mapping` 来正确地更新位置信息，这是编写健壮的 ProseMirror 插件的关键技能。
