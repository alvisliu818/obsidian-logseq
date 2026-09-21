# 项目：obsidian-logseq（Obsidian 插件，Logseq 式块编辑器）

## 工程约定
- 只用 npm（禁用 pnpm）。
- 改代码后的验证链：`npx tsc -noEmit -skipLibCheck` → `npx vitest run` → `npm run build`（= tsc + esbuild production，产物 `main.js`，已 gitignore）。
- **部署**：构建后把 `main.js` + `manifest.json` + `styles.css` 复制到 `E:\HOME\BaiduSyncdisk\.obsidian\plugins\obsidian-logseq`，**不要覆盖 `data.json`**（用户设置），复制后用 `cmp` 校验。
- 单测放 `tests/`，用 vitest；被单测的模块必须不 import `obsidian`（否则 node 下无法加载）。纯逻辑优先抽成独立模块（如 `src/features/copyFormats.ts`）再写测试。
- `src/core/*`（parser/serializer/treeOps）与 `src/types.ts` 保持无依赖。

## 关键语法
- 块引用：`((uuid))`（渲染成行内 chip）→ `features/links.ts` 的 `enhanceBlockRefs`
- 块嵌入：`{{embed ((uuid))}}`（渲染成嵌入块，显示整棵子树）→ `enhanceEmbeds`
- **渲染顺序铁律**：`enhanceEmbeds` 必须在 `enhanceBlockRefs` 之前跑——否则 `((uuid))` 先被 chip 化，embed 语法失效。
- 嵌入块：点击 body 行 = 原地编辑该行对应块（CM6，跨文件也支持，按 child-index path 定位），面包屑 = 跳转源块；提交（Enter/Esc/blur）后写回源块并重新渲染 box。写回同名文件走 model，跨文件优先走已打开的 BlockEditorView，否则 `vault.modify`。
- **嵌入框必须吞掉自己所有的点击**（`stopPropagation` 放在最前 + `wireContentEvents` 里 `.block-embed` 兜底 return），否则冒泡到宿主块会把它切成编辑态、露出 `{{embed ((id))}}` 源码并干掉原地编辑器。
- 块 id 存在 `id:: uuid` 属性里（`ensureId` 惰性分配；分配后需经 `host.mutate` 落盘）

## 交互现状（2026-09 起）
- 圆点：单击 = 聚焦(zoom)；Shift/Ctrl+单击 = 多选；右键 = 块菜单顶部带「Copy block embed / Copy block as markdown」
- 折叠 = caret，TODO 循环 = marker 复选框 / Ctrl+Enter
- 文档注释/菜单文案用英文
