好的，我们来对 prosemirror-search 中的 query.ts 文件进行一次全面、深入的讲解。如果说 `search.ts` 是搜索功能的“骨架”和“神经系统”（负责状态管理和与编辑器的集成），那么 query.ts 就是其“大脑”和“肌肉”——它包含了所有关于**如何查找**和**如何替换**的纯粹算法和逻辑。

这个文件完全独立于 ProseMirror 的插件系统和视图层，使其逻辑非常清晰和可测试。

---

### 1. 核心类：`SearchQuery`

这是你与搜索逻辑交互的唯一入口。它是一个不可变的对象，封装了一个完整的查询请求的所有信息和能力。

#### a. 构造函数与配置

当你创建一个 `SearchQuery` 实例时，你提供一个配置对象，它包含了：

- `search`: 搜索的字符串。
- `caseSensitive`: 是否区分大小写。
- `regexp`: 是否将 `search` 字符串解释为正则表达式。
- `literal`: 是否按字面量解析 `search` 字符串中的 `\n`, `\r`, `\t`。
- `wholeWord`: 是否开启全词匹配。
- `replace`: 替换文本。
- `filter`: 一个可选的过滤函数，可以用来忽略某些匹配结果。

#### b. 核心设计：策略模式 (`QueryImpl`)

`SearchQuery` 最精妙的设计在于其内部的**策略模式**。它不亲自执行搜索，而是根据配置，将任务委托给一个实现了 `QueryImpl` 接口的内部对象。

```typescript
// ...existing code...
  /// @internal
  impl: QueryImpl

  /// Create a query object.
  constructor(/*...*/) {
// ...existing code...
    this.impl = !this.valid ? nullQuery : this.regexp ? new RegExpQuery(this) : new StringQuery(this)
  }
// ...existing code...
```

- 如果查询无效（例如，一个空的或语法错误的正则表达式），它使用 `nullQuery`，这个实现的所有方法都直接返回 `null`。
- 如果 `regexp` 为 `true`，它创建一个 `RegExpQuery` 实例。
- 否则，它创建一个 `StringQuery` 实例。

这种设计将高层的查询配置（`SearchQuery`）与底层的搜索算法（`StringQuery`, `RegExpQuery`）完全解耦，使得代码结构非常清晰。

#### c. `findNext` 和 `findPrev` 方法

这两个方法是查找的入口。它们本身不执行查找，而是：

1.  调用 `this.impl.findNext(...)` 或 `this.impl.findPrev(...)` 来获取一个原始的匹配结果。
2.  如果找到了结果，就调用 `this.checkResult(...)` 对其进行**后处理**。
3.  `checkResult` 会检查是否满足 `wholeWord`（全词匹配）和 `filter`（自定义过滤）的条件。
4.  如果后处理失败，它会从失败的位置继续向后/向前搜索，直到找到一个有效的匹配或搜索完整个范围。

#### d. `getReplacements` 方法：复杂的替换逻辑

这是 query.ts 中最复杂但功能最强大的部分。它负责计算出替换操作需要执行的具体步骤。它不仅仅是简单的文本替换，还支持正则表达式中的捕获组引用（如 `$1`, `$&`）。

它的工作流程是：

1.  **解析替换字符串**: 调用 `parseReplacement` 将 `replace` 字符串（如 `"Found: $1"`）解析成一个指令数组，例如 `["Found: ", {group: 1, copy: false}]`。
2.  **获取捕获组**: 从 `SearchResult` 中获取正则表达式匹配到的所有捕获组的位置信息。
3.  **构建替换片段**: 遍历指令数组。
    - 如果指令是普通字符串，就创建一个文本节点。
    - 如果指令是捕获组引用（如 `$1`），它会从原始文档中**切片 (`slice`)** 出该捕获组对应的内容，并将其追加到正在构建的 `Fragment` 中。
4.  **计算替换范围**: 通过巧妙地追踪位置，它能计算出多个不连续的替换范围。例如，对于 `"$1"` 这样的替换，它会生成两个 `range`：一个是从匹配开始到 `$1` 开始，用空内容替换（删除）；另一个是从 `$1` 结束到匹配结束，也用空内容替换（删除）。中间 `$1` 的部分被保留了下来。
5.  最终返回一个 `{from, to, insert: Slice}` 对象数组，`search.ts` 中的 `replace` 命令会精确地执行这些操作。

---

### 2. 底层实现：`StringQuery` 和 `RegExpQuery`

这两个类是实际执行搜索的“工人”。

#### `StringQuery`

- **原理**: 非常直接。它使用 `textContent` 获取文本块的内容，然后调用 JavaScript 的 `indexOf` 或 `lastIndexOf` 方法来查找字符串。
- **大小写不敏感**: 如果 `caseSensitive` 为 `false`，它会将搜索字符串和文本块内容都转换为小写再进行比较。

#### `RegExpQuery`

- **原理**: 它使用 `textContent` 获取文本块内容，然后调用 `regexp.exec()` 来执行匹配。
- **状态管理**: 正则表达式的 `lastIndex` 属性在这里至关重要。每次调用 `exec` 后，`lastIndex` 会自动更新，使得下一次调用可以从上一次匹配结束的地方继续。
- **`findPrev` 的挑战**: 正则表达式本身不支持反向搜索。因此，`findPrev` 的实现方式是：在一个文本块内，从头到尾循环执行 `exec()`，不断用新的匹配结果覆盖旧的，直到循环结束。最后记录的那个 `match` 就是该文本块中的最后一个匹配，也就是反向搜索找到的第一个。

---

### 3. 与 ProseMirror 模型交互的“黑魔法”：`scanTextblocks` 和 `textContent`

这是 query.ts 中最底层、也是与 ProseMirror 耦合最紧密的部分。由于 ProseMirror 的文档是一个树状结构，而不是一个扁平的字符串，因此不能直接在整个文档上进行字符串搜索。

#### `textContent(node)`

- **目的**: 将一个 ProseMirror 节点（及其所有子孙）的文本内容**近似地**转换成一个扁平的字符串，以便进行搜索。
- **实现**:
  - 它递归地遍历节点树。
  - 对于文本节点，直接追加其内容。
  - 对于叶子节点（如 `image`），它插入一个特殊字符 `\ufffc` (Object Replacement Character) 作为占位符。
  - 对于块级节点之间的边界，它插入空格来模拟段落间的分隔。
  - **缓存**: 它使用 `WeakMap` 来缓存已计算过的节点内容，极大地提高了性能，避免了对同一节点反复进行成本高昂的文本提取。

#### `scanTextblocks(node, from, to, f)`

- **目的**: 这是一个智能的遍历器。它递归地遍历 ProseMirror 文档树，但**只会在包含内联内容（即可以被视为文本块）的节点上**执行你提供的回调函数 `f`。
- **工作方式**:
  - 它会跳过那些只包含块级子节点的容器节点（如 `doc`, `blockquote`）。
  - 当它找到一个 `inlineContent` 为 `true` 的节点（如 `paragraph`, `heading`）时，它会调用 `f(node, nodeStart)`，并将该节点和它在整个文档中的起始位置传递给你。
  - `StringQuery` 和 `RegExpQuery` 的 `findNext`/`findPrev` 方法就是将它们各自的搜索逻辑作为回调 `f` 传递给 `scanTextblocks`。

### 总结

query.ts 是一个展示了如何在复杂的树状结构上实现高效文本操作的杰作。

1.  **分层与解耦**: 通过 `SearchQuery` (接口层)、`QueryImpl` (策略层) 和 `scanTextblocks` (遍历层) 的设计，将复杂的逻辑清晰地分层。
2.  **模型抽象**: `textContent` 函数巧妙地在“忠于原始结构”和“便于字符串处理”之间取得了平衡，创建了一个可供搜索的虚拟文本视图。
3.  **性能优化**: `WeakMap` 缓存 `textContent` 的结果，是避免重复计算、保证搜索性能的关键。
4.  **强大的替换能力**: `getReplacements` 的设计远超简单的文本替换，通过解析替换字符串和引用捕获组，实现了与现代代码编辑器相媲美的替换功能。

深入理解 query.ts，你就能掌握在任何树状数据结构（不仅仅是 ProseMirror）上实现高级查找和替换功能的核心思想。
