# Obsidian Logseq 风格块编辑器插件 — 实现计划

## Context

在 Obsidian 中用类 Logseq 的大纲块编辑器**全量替换原生 markdown 编辑器**（超集策略：块模式为默认，原生编辑器随时可切回，文件 100% 标准 markdown）。文件采用 Logseq 风格存储：嵌套 `- ` 列表 + 行首 `TODO/DOING/DONE` marker + 缩进子行 `id:: uuid` 属性 + `((uuid))` 块引用语法。项目从零创建于 `e:\HOME\Code\obsidian\obsidian-logseq`（空目录）。

参考源码：Logseq（ClojureScript）`E:\HOME\Code\logseq\og` — 块树渲染 `components/block.cljs`（DOM 结构：bullet/折叠箭头/.block-content/.block-children-container+左引导线）、键盘分发 `handler/editor.cljs`、slash 菜单 `commands.cljs`、块树构建 `modules/outliner/tree.cljs`。

**用户已确认决策**：① 全量接管所有 .md（可设置排除文件夹，命令切回原生）② Logseq `id::` 格式块引用 ③ V1 = 完整 Logseq 核心体验 ④ 用户有 Obsidian + 开发 vault（部署路径待用户提供）。

## 已验证的关键技术路径

- **接管 .md 视图**：monkey-around patch `WorkspaceLeaf.setViewState`，`state.type === "markdown"` 且不在排除目录时改写为自定义 viewType；卸载时遍历所有 leaf 还原为 markdown。（Kanban 插件验证方案，比 `unregisterExtensions` 更稳）
- **自定义 View**：继承 `TextFileView`（文件 I/O、dirty、保存管线复用原生）
- **聚焦块编辑**：自捆绑 `@codemirror/{state,view,language,commands,autocomplete}` + `@codemirror/lang-markdown`；每个聚焦块一个 CM6 实例，挂在自建 DOM（不与内置实例混用）
- **未聚焦块渲染**：`MarkdownRenderer.render(app, md, el, sourcePath, component)`（View 作 component），渲染序号丢弃旧异步结果防竞态
- **兼容性**：文件经 Vault API 写入 → backlinks/graph/搜索/同步正常

## 项目结构

```
obsidian-logseq/
├─ manifest.json / styles.css / esbuild.mjs / tsconfig.json(strict) / package.json
└─ src/
   ├─ main.ts               入口：setViewState patch、全局命令、卸载恢复
   ├─ types.ts              Block/Marker/插件设置接口
   ├─ view/BlockEditorView.ts   TextFileView 子类：文件 I/O、getState/setState、zoom 状态持久化
   ├─ core/parser.ts        md → Block 树（栈式逐行解析）
   ├─ core/serializer.ts    Block 树 → md（tab 缩进嵌套列表）
   ├─ core/treeOps.ts       纯函数树操作：insert/split/merge/indent/move
   ├─ blocks/renderTree.ts  块树 → DOM（折叠箭头/bullet/内容/children 容器+引导线）
   ├─ editor/focusEditor.ts CM6 挂载/卸载/取值回写
   ├─ editor/extensions.ts  CM6 扩展：keymap/markdown 语言/autocomplete
   ├─ interactions/keymap.ts    Enter/Tab/Backspace/方向键分发
   ├─ interactions/dnd.ts       HTML5 拖拽 + 三档落点计算
   ├─ interactions/collapse.ts  折叠状态管理
   ├─ interactions/zoom.ts      块缩放 + 面包屑
   ├─ features/todo.ts      marker 循环（点 bullet / Ctrl+Enter）
   ├─ features/slashMenu.ts "/" 命令菜单（CM6 autocomplete 源）
   ├─ features/links.ts     [[页面]]/#标签/((uuid)) 补全与引用渲染
   ├─ index/blockIndex.ts   全 vault uuid → 位置索引（metadataCache 不索引正文属性行，需自建）
   ├─ state/undoStack.ts    快照式块级撤销栈
   └─ settings/tab.ts       设置页：排除文件夹、默认视图
```

## 核心数据流

```
patch 拦截 markdown viewState → BlockEditorView.onLoadFile
→ parser.parse → renderTree 渲染 → 聚焦块挂 CM6
→ 编辑事务更新 Block → debounce 800ms
→ serializer.serialize → view.data + requestSave() → vault 落盘
```

```ts
interface Block {
  id: string;                     // uuid，新块即时生成
  marker: 'TODO'|'DOING'|'DONE'|null;
  text: string;                   // 不含 marker/属性
  props: Record<string,string>;   // id::、collapsed::
  children: Block[];
  parent: Block|null;             // 运行时指针，不序列化
  collapsed: boolean;
}
```

**解析规则**（栈式逐行，parse∘serialize 往返不变式为单测核心）：
- `/^(\t*)- (TODO |DOING |DONE )?(.*)$/`，tab 深度对比栈顶定父子
- 比块内容更深缩进的 `key:: value` 子行 → 当前块 props
- 非列表行（标题/段落）包为顶层块保留原文；frontmatter 原样前置

## 关键技术方案

1. **CM6 生命周期**：点击未聚焦块 → 在其 .block-content 内建 EditorView 替换静态 HTML；blur → doc 写回 Block → destroy → MarkdownRenderer 重渲染（带渲染序号防竞态）
2. **跨块光标**：ArrowUp/Down 由 CM6 keymap 拦截；已在首/末视觉行 → preventDefault → 聚焦相邻块，光标置 end/start
3. **撤销**：自建快照栈——块级事务前 push 全文 serialize()（上限 100 步）；聚焦时块内 Ctrl+Z 归 CM6 history，非聚焦弹栈恢复重解析
4. **保存与冲突**：事务置 dirty → debounce → requestSave；自身写入排除；外部变更无 dirty → 静默重载，有 dirty → confirm 覆盖/重载
5. **拖拽落点**：dragover 按目标块内容区 Y 三档——上 25% 插前 / 下 25% 插后（同层）/ 中 50% 作 first-child；禁止拖入自身子孙
6. **Zoom**：view 持有 zoomedBlockId，渲染只输出该块子树；面包屑为根到该块路径；getState/setState 持久化
7. **块引用**：自建索引 Map<uuid, {path, block}>，启动全量扫描 + modify/rename/delete 增量维护；渲染时 ((uuid)) 替换为被引块 markdown 再交 MarkdownRenderer；点击跳转定位
8. **补全**：CM6 autocompletion override 源——`[[`（复用 metadataCache.getLinkSuggestions()）、`#` 标签、`/` slash 分组菜单

## 实现里程碑

- **M1 架构跑通**：main.ts + view/ + parser + serializer + renderTree + styles.css（Logseq 视觉：bullet/引导线/children 容器）
  验收：所有 .md 以块树打开（只读）；排除目录生效；切回原生/卸载恢复正常；真实 Logseq 导出文件解析无损
- **M2 编辑内核**：focusEditor + keymap + treeOps + undoStack + 保存链路
  验收：Enter 新建/拆分、Tab/Shift+Tab 缩进、Backspace 块首合并、方向键跨块、自动落盘、外部改动重载、Ctrl+Z
- **M3 交互增强**：collapse + dnd + todo + zoom
  验收：折叠持久（collapsed:: 写盘）、拖拽三档落点、bullet 点击/Ctrl+Enter 循环 marker、zoom 进出 + 面包屑 + 状态恢复
- **M4 补全与菜单**：slashMenu + links + blockIndex
  验收：[[ ]]、#、((uuid)) 补全；引用内联渲染可跳转；索引编辑后即时更新

## 验证方案

- **vitest 纯逻辑单测**：parser 往返不变式与边界（空文件/纯段落/深层嵌套/混合属性行/frontmatter）；treeOps（indent 越界、merge 跨层、move 入子孙拒绝）；serializer 输出稳定性
- **开发 vault 手动验证**：M1 用真实 Logseq 导出文件对照渲染；M2 外部编辑器改文件触发冲突路径；M3 长文档（500+ 块）滚动与拖拽性能；M4 块/原生编辑器互切往返内容零丢失
- **部署**：构建产物（main.js/manifest.json/styles.css）复制到开发 vault 的 `.obsidian/plugins/<id>/`（具体路径待用户提供后写入 esbuild 部署脚本/文档）

## 参考文件（实现时对照）

- Logseq 块 DOM 结构：`E:\HOME\Code\logseq\og\src\main\frontend\components\block.cljs`（L1743-1816 block-control：bullet/折叠/拖拽手柄；L1930-2020 build-block-title：内容组装）
- Logseq 块样式：`E:\HOME\Code\logseq\og\src\main\frontend\components\block.css`（L166-207 嵌套引导线结构）
- Logseq 键盘分发：`E:\HOME\Code\logseq\og\src\main\frontend\components\editor.cljs`（L485-507 on-key-down → editor-handler）
- Logseq slash 命令：`E:\HOME\Code\logseq\og\src\main\frontend\commands.cljs`
- Kanban 接管方案：patch `WorkspaceLeaf.setViewState`（community-archive/obsidian-kanban main.ts）
