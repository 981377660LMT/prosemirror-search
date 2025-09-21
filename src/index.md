好的，我们来深入代码，详细分析和讲解 prosemirror-search 这个模块。它是一个功能完备的库，提供了在 ProseMirror 编辑器中进行查找和替换所需的所有底层逻辑和上层命令。

这个模块的设计非常清晰，可以分为两个主要部分：

1.  **`query.ts` - 搜索引擎**: 这是模块的核心。它定义了什么是“查询”，并实现了在 ProseMirror 文档中查找匹配项的纯逻辑，完全与编辑器 UI 和状态管理解耦。
2.  **`search.ts` - 编辑器集成**: 这部分将 query.ts 提供的搜索引擎与 ProseMirror 的插件系统、命令和视图（通过 `Decoration`）连接起来，为用户提供完整的查找、高亮和替换功能。

我们将按照这个逻辑顺序来深入讲解。

---

### 第一部分：`query.ts` - 搜索引擎的核心

这个文件定义了如何执行一个搜索。

#### 1. `SearchQuery` 类 - 查询的定义

这是所有搜索操作的起点。它是一个配置对象，封装了用户发起一次搜索的所有意图。

```typescript
export class SearchQuery {
  // ...
  constructor(config: {
    search: string
    caseSensitive?: boolean
    literal?: boolean
    regexp?: boolean
    replace?: string
    wholeWord?: boolean
    // ...
  }) {
    // ...
    this.impl = !this.valid
      ? nullQuery
      : this.regexp
      ? new RegExpQuery(this)
      : new StringQuery(this)
  }
  // ...
}
```

- **构造函数**: 接收一个配置对象，包含 `search` 字符串、是否 `caseSensitive`、是否是 `regexp`（正则表达式）等。
- **`impl` 属性**: 这是设计的关键，采用了**策略模式 (Strategy Pattern)**。根据 `regexp` 标志，`SearchQuery` 会实例化一个具体的查询实现 (`RegExpQuery` 或 `StringQuery`) 并赋值给 `impl`。所有实际的查找操作（如 `findNext`）都会被委托给这个 `impl` 对象。这使得 `SearchQuery` 的上层 API 无需关心底层是字符串匹配还是正则匹配。

#### 2. `StringQuery` 和 `RegExpQuery` - 具体的查询策略

这两个类都实现了 `QueryImpl` 接口，该接口要求它们提供 `findNext` 和 `findPrev` 方法。

- **`StringQuery`**: 用于简单的字符串查找。
- **`RegExpQuery`**: 用于正则表达式查找。

它们的实现都依赖于一个核心的辅助函数：`scanTextblocks`。

#### 3. `scanTextblocks` 和 `textContent` - 在结构化文档中查找文本

ProseMirror 的文档是一个树状结构，而不是一个扁平的字符串。直接在上面做文本搜索是低效且复杂的。prosemirror-search 的解决方案是：

1.  **`textContent(node)`**: 这个函数递归地遍历一个节点及其所有子节点，将它们的文本内容拼接成一个**扁平的字符串**。

    - 它用一个空格来分隔不同的块级节点。
    - 它用一个特殊字符 `\ufffc` (Object Replacement Character) 来代表叶子节点（如图片），以确保即使它们不产生文本，也能在计算位置时占据一个字符，保证位置映射的准确性。
    - 为了性能，它使用 `WeakMap` (`TextContentCache`) 来缓存已计算过的节点的文本内容。

2.  **`scanTextblocks(node, from, to, f)`**: 这个函数递归地遍历文档，但它很聪明，只在包含文本的块级节点（textblock）上执行真正的搜索。
    - 它接收一个回调函数 `f`。
    - 当它进入一个 textblock 时，它调用 `f`，并将这个 textblock 节点和它在文档中的起始位置传递给 `f`。
    - `StringQuery` 和 `RegExpQuery` 的 `findNext`/`findPrev` 方法就是将它们各自的匹配逻辑作为回调 `f` 传递给 `scanTextblocks`。

**例如，`StringQuery.findNext` 的工作流程**:

1.  调用 `scanTextblocks` 遍历文档。
2.  `scanTextblocks` 进入一个段落（textblock）。
3.  `findNext` 的回调函数被执行。
4.  回调函数内部调用 `textContent(paragraphNode)` 获取该段落的扁平文本。
5.  然后在这个扁平文本上执行 `indexOf` 来查找匹配项。
6.  如果找到，根据匹配的索引和段落的起始位置，计算出匹配项在**整个文档**中的 `from` 和 `to` 坐标，并返回 `SearchResult`。

#### 4. `getReplacements` 和 `parseReplacement` - 处理替换逻辑

当需要替换一个匹配项时，`getReplacements` 方法被调用。它支持 `$` 语法（如 `$1`, `$&`），这在正则表达式替换中很常见。

- **`parseReplacement(text)`**: 这个函数解析替换字符串（例如 `"Found: $1"`），将其分解成一个数组，包含普通字符串和代表捕获组的对象（例如 `[{group: 1, copy: false}]`）。
- **`getReplacements(state, result)`**:
  1.  调用 `parseReplacement` 解析替换文本。
  2.  遍历解析出的部分。
  3.  如果是普通字符串，就创建一个包含该文本的 `Slice`。
  4.  如果是捕获组，就从 `SearchResult` 的 `match` 数组中找到对应的原始文本，并创建一个包含该原始文本的 `Slice`。
  5.  最终，它返回一个范围数组 `[{from, to, insert}]`，描述了如何通过一系列替换操作来完成整个替换。这个设计非常精巧，因为它允许替换操作跳过（保留）匹配项中的某些部分。

---

### 第二部分：`search.ts` - 与编辑器的集成

这个文件负责将 query.ts 的能力展现给用户。

#### 1. `search()` 插件和 `SearchState`

这是功能的核心入口。`search()` 函数创建了一个 ProseMirror 插件。

- **`SearchState`**: 插件的状态对象，存储了：

  - `query: SearchQuery`: 当前的查询对象。
  - `range: {from, to} | null`: 当前的搜索范围，如果为 `null` 则为整个文档。
  - `deco: DecorationSet`: 一个 `DecorationSet`，包含了所有匹配项的高亮装饰。

- **插件逻辑**:
  - **`state.init`**: 初始化 `SearchState`。
  - **`state.apply`**: 监听 `Transaction`。
    - 如果 `Transaction` 带有 `searchKey` 的元数据（由 `setSearchState` 设置），就用新的查询和范围更新整个 `SearchState`。
    - 如果文档发生变化，它会重新计算高亮装饰 (`buildMatchDeco`)。
  - **`props.decorations`**: 将 `SearchState` 中的 `deco` 提供给 `EditorView`，视图会自动渲染这些高亮。

#### 2. `buildMatchDeco` - 创建高亮装饰

这个函数是实现实时高亮的关键。

```typescript
function buildMatchDeco(state: EditorState, query: SearchQuery, range: {from: number, to: number} | null) {
  // ...
  for (let pos = ...; ...;) {
    let next = query.findNext(state, pos, end) // 1. 使用 query 引擎查找
    if (!next) break
    // 2. 为当前选中的匹配项添加特殊 class
    let cls = next.from == sel.from && next.to == sel.to ? "ProseMirror-active-search-match" : "ProseMirror-search-match"
    // 3. 创建 Decoration
    deco.push(Decoration.inline(next.from, next.to, {class: cls}))
    pos = next.to
  }
  return DecorationSet.create(state.doc, deco)
}
```

它在一个循环中不断调用 `query.findNext`，遍历所有匹配项，并为每一个匹配项创建一个 `Decoration.inline`。

#### 3. `setSearchState` 和 `getSearchState` - 状态管理 API

- `setSearchState(tr, query)`: 这是**更新**搜索查询的唯一正确方式。它将新的 `SearchQuery` 对象作为元数据附加到 `Transaction` 上。
- `getSearchState(state)`: 用于从外部获取当前的搜索状态。

#### 4. `find...` 和 `replace...` 命令

这些是导出的、可以直接绑定到快捷键或按钮上的 `Command` 函数。

- **`findCommand(wrap, dir)`**: 一个命令工厂，用于创建 `findNext` 和 `findPrev`。

  - 它从 `SearchState` 中获取当前查询。
  - 调用 `nextMatch` 或 `prevMatch` 辅助函数来查找下一个匹配项（这两个函数内部处理了是否“循环搜索”的逻辑）。
  - 如果找到，就创建一个新的 `TextSelection` 来选中匹配项，并派发一个更新选区的 `Transaction`。

- **`replaceCommand(wrap, moveForward)`**: 一个更复杂的命令工厂，用于创建 `replaceNext` 和 `replaceCurrent`。

  - 它首先查找下一个匹配项。
  - **核心逻辑**: 如果当前用户的选区**正好**与找到的匹配项重合，它就执行替换操作：
    1.  调用 `search.query.getReplacements()` 获取替换所需的 `Slice`。
    2.  从后往前地将这些替换应用到 `Transaction` 中（从后往前是为了避免位置映射的麻烦）。
    3.  如果 `moveForward` 为 `true`，它会继续查找下一个匹配项并选中它。
  - 如果当前选区不匹配，它就只执行查找和选中的操作。

- **`replaceAll`**: 这个命令最直接。它循环调用 `query.findNext` 找到所有匹配项，然后在一个 `Transaction` 中从后往前地应用所有替换。

### 总结与回顾

1.  **配置与策略**: 用户通过 `new SearchQuery(...)` 创建一个查询配置。该对象内部根据配置选择 `StringQuery` 或 `RegExpQuery` 策略。
2.  **扁平化搜索**: 具体的查询策略使用 `scanTextblocks` 和 `textContent` 将结构化的 ProseMirror 文档“暂时”扁平化为字符串，以便进行高效的 `indexOf` 或 `regexp.exec` 搜索。
3.  **插件管理状态**: `search()` 插件在 `EditorState` 中维护一个 `SearchState`，包含当前的 `query` 和高亮 `DecorationSet`。
4.  **状态更新**: 外部通过 `setSearchState` 发起一个带元数据的 `Transaction` 来更新查询，插件的 `apply` 方法捕获这个元数据并更新 `SearchState`。
5.  **渲染高亮**: `SearchState` 更新后，`buildMatchDeco` 会被调用，它使用 `query.findNext` 遍历所有匹配项，并创建 `Decoration`。插件通过 `props` 将这些 `Decoration` 提供给视图进行渲染。
6.  **用户交互**: `find...` 和 `replace...` 命令将所有这些逻辑串联起来，响应用户的操作，创建并派发包含选区变化或文档修改的 `Transaction`。

这个库完美地展示了 ProseMirror 的模块化和分层设计思想：一个纯粹的、可测试的逻辑核心 (`query.ts`)，加上一个与编辑器生态系统（插件、命令、视图）无缝集成的适配层 (`search.ts`)。
