# my-notes React + Vite 改造计划

> 状态：待实施  
> 日期：2026-07-29  
> 目标站点：https://huangjl-cn.github.io/my-notes/  
> 当前发布方式：GitHub Pages legacy，`main /` 直接发布

## 1. 结论

采用 **React + Vite + TypeScript** 改造首页，继续保留现有独立 HTML 笔记页。

核心选择：

- 使用 React 学习组件、Hooks、Effect 和生态工具。
- 使用 Vite 构建静态 `dist`，不引入服务端运行时。
- 使用 TypeScript 固化笔记 metadata 契约，降低技能写错数据的风险。
- 不引入 React Router。首页只有一个应用入口，笔记仍是真实 HTML 文件，可避免 GitHub Pages 刷新 404。
- 不引入 Redux、Zustand、CSS-in-JS、UI 组件库和 Three.js。
- 保留当前原生 WebGL 光带和 Canvas 点阵，不重新设计视觉效果。
- 将 Pages 发布方式切换为 GitHub Actions，上传 `dist` artifact。
- 将 `html-note` 从“编辑 `index.html`”改为“生成页面并更新结构化 JSON”。

## 2. 成功标准

改造完成必须同时满足：

1. 首页视觉与当前线上版本基本一致，包括字体、卡片、分页、点阵、光带和滚动按钮。
2. 现有 9 条首页笔记均可打开新窗口，10 个 `notes/*.html` 文件全部保留，`trace_e4a45e52.html` 继续不进入首页。
3. Counter-X 的读取、点击递增和失败降级行为不变。
4. 桌面端、移动端和 `prefers-reduced-motion` 行为不退化。
5. `npm run build` 生成可独立部署的 `dist`。
6. GitHub Pages 从 Actions 发布，部署地址仍为 `/my-notes/`。
7. 使用 `html-note` 新增笔记时，不需要修改 React 组件、CSS 或根 `index.html`。
8. 新增笔记后，结构校验、生产构建和链接检查全部通过。
9. 可以通过明确步骤恢复到迁移前的静态版本。

## 3. 非目标

本次不做以下工作：

- 不把详情页改写为 React。
- 不增加搜索、标签页、全文索引、后台管理或在线编辑。
- 不更换 Counter-X 服务或历史计数 namespace。
- 不重做首页设计和动画参数。
- 不引入 SSR、Next.js、数据库或服务端 API。
- 不顺手重构现有笔记页面的 HTML/CSS。

## 4. 目标项目结构

```text
my-notes/
├─ .github/
│  └─ workflows/
│     └─ deploy-pages.yml
├─ docs/
│  └─ react-vite-migration-plan.md
├─ public/
│  ├─ favicon.svg
│  ├─ THIRD_PARTY_NOTICES.md
│  ├─ images/
│  │  └─ *.jpg
│  └─ notes/
│     └─ *.html
├─ scripts/
│  ├─ update-note-index.mjs
│  └─ validate-content.mjs
├─ src/
│  ├─ components/
│  │  ├─ AmbientGrid.tsx
│  │  ├─ AmbientWave.tsx
│  │  ├─ Footer.tsx
│  │  ├─ Header.tsx
│  │  ├─ NoteCard.tsx
│  │  ├─ NoteList.tsx
│  │  ├─ Pagination.tsx
│  │  └─ ScrollNav.tsx
│  ├─ config/
│  │  └─ site.ts
│  ├─ data/
│  │  └─ notes.json
│  ├─ lib/
│  │  ├─ counter.ts
│  │  └─ note-paths.ts
│  ├─ styles/
│  │  └─ index.css
│  ├─ types/
│  │  └─ note.ts
│  ├─ App.tsx
│  └─ main.tsx
├─ tests/
│  ├─ content.test.ts
│  └─ pagination.test.ts
├─ .gitignore
├─ index.html
├─ package.json
├─ package-lock.json
├─ tsconfig.json
├─ tsconfig.app.json
└─ vite.config.ts
```

`.codex/skills/html-note/` 继续是本机技能目录并保持 Git 忽略，不进入 Pages 构建产物。

## 5. 现有文件迁移映射

| 当前路径 | 目标路径 | 处理方式 |
| --- | --- | --- |
| `index.html` | `index.html` | 改为 Vite 入口，只保留 metadata、字体、`#root` 和 module script |
| `index.html` 内 CSS | `src/styles/index.css` | 首次迁移原样搬运，暂不拆 CSS Modules |
| `index.html` 内 notes 数组 | `src/data/notes.json` | 转成结构化数据，首页构建时导入 |
| `index.html` 内 WebGL | `src/components/AmbientWave.tsx` | 保持 shader 与参数，加入 React 生命周期清理 |
| `index.html` 内 Canvas 点阵 | `src/components/AmbientGrid.tsx` | 保持绘制和鼠标交互，加入生命周期清理 |
| `index.html` 内列表渲染 | `NoteList.tsx`、`NoteCard.tsx` | 改成声明式 JSX |
| `index.html` 内分页 | `Pagination.tsx` | 使用本地 state，保持每页 4 条 |
| `index.html` 内滚动按钮 | `ScrollNav.tsx` | Effect 注册滚动监听并清理 |
| `notes/*.html` | `public/notes/*.html` | 原样移动，不改页面内容 |
| `images/*.jpg` | `public/images/*.jpg` | 原样移动 |
| `favicon.svg` | `public/favicon.svg` | 原样移动 |
| `THIRD_PARTY_NOTICES.md` | `public/THIRD_PARTY_NOTICES.md` | 让源码仓库和 `dist` 都包含许可声明 |

## 6. React 代码设计

### 6.1 应用入口

`src/main.tsx` 只负责：

- 使用 `createRoot` 挂载 `App`。
- 保留 `StrictMode`，用开发期双重 setup/cleanup 暴露 Effect 清理缺失。
- 引入全局 CSS。

`src/App.tsx` 只组合页面结构，不放 Canvas 实现、Counter 请求或 metadata 编辑逻辑。

```tsx
function App() {
  return (
    <>
      <AmbientWave />
      <AmbientGrid />
      <main className="page">
        <Header />
        <NoteList notes={notes} />
      </main>
      <ScrollNav />
      <Footer />
    </>
  )
}
```

### 6.2 笔记数据契约

`src/data/notes.json` 使用纯 JSON，按日期从新到旧排列：

```json
[
  {
    "slug": "ls-terminal-shell-kernel",
    "topic": "system",
    "category": "System",
    "date": "2026-07-29",
    "title": "ls 命令如何穿过终端、Shell 与内核",
    "keywords": ["终端模拟器", "Shell", "PTY", "系统调用", "ELF"],
    "image": "images/ls-terminal-shell-kernel.jpg"
  }
]
```

约束：

- `slug` 是唯一主键，React 根据它生成详情页链接和 Counter-X 请求路径。
- `topic` 限定为 `cache`、`network`、`agent`、`transformer`、`system`、`note`。
- `image` 无缩略图时为 `null`，不能填不存在的路径。
- `keywords` 为 3 至 5 个字符串，由组件使用 ` · ` 拼接。
- 不在数据中重复保存可推导的 `href` 和 `desc`。
- `trace_e4a45e52` 不写入 JSON。

`src/types/note.ts` 定义 `Note` 类型；`tests/content.test.ts` 和内容校验脚本负责运行时完整性，不能只依赖 TypeScript 对 JSON 的推断。

### 6.3 静态资源与 base

Vite 配置固定使用：

```ts
export default defineConfig({
  base: '/my-notes/',
  plugins: [react()],
})
```

组件通过 `import.meta.env.BASE_URL` 构造图片和笔记 URL，不写以 `/` 开头的站点根路径。

示例：

```ts
const noteUrl = `${import.meta.env.BASE_URL}notes/${note.slug}.html`
const imageUrl = note.image
  ? `${import.meta.env.BASE_URL}${note.image}`
  : undefined
```

详情页中的 `../index.html` 保持不变，因为构建后仍是 `dist/notes/<slug>.html`。

### 6.4 Counter-X

在 `src/config/site.ts` 集中定义：

```ts
export const COUNTER_NAMESPACE = 'hjl-2005.github.io'
export const REPOSITORY_URL = 'https://github.com/Huangjl-cn/my-notes'
```

注意：仓库所有者已经是 `Huangjl-cn`，但 Counter-X namespace 暂时保留 `hjl-2005.github.io`，否则会丢失历史点击数。

`src/lib/counter.ts` 提供两个函数：

- `getCount(slug, signal)`：读取点击数，失败返回可识别的 unavailable 状态。
- `hitCount(slug)`：使用 `keepalive: true` 递增。

`NoteCard` 负责展示 loading、成功和失败状态，并在链接点击时调用 `hitCount`。读取请求必须支持 `AbortController`，避免翻页卸载后更新旧组件。

### 6.5 分页

保持现有行为：

- `PAGE_SIZE = 4`。
- 当前 9 条索引数据产生 3 页。
- 切页按钮使用 React state，不写入 URL。
- 上一页、下一页在边界禁用。
- 保留当前 180ms 淡出和逐项进入效果。
- 定时器必须在组件卸载或快速重复切页时清理。

### 6.6 WebGL 光带

`AmbientWave.tsx` 使用 `canvasRef` 和单个 Effect：

1. 获取 WebGL context。
2. 编译 shader、创建 program 和 buffer。
3. 注册 resize、visibilitychange。
4. 启动 `requestAnimationFrame`。
5. 在 cleanup 中取消 RAF、移除监听、删除 buffer/program/shader，并释放 context。

必须保留：

- 当前 `#cc785c` 铜色光带。
- 左上与右下区域的亮度遮罩。
- 只增加亮度、不生成黑点的 grain 算法。
- 正弦往复运动。
- `150vh` 底部锚定位置。
- 预乘 Alpha 混合方式。
- Vue Bits 版权注释和第三方许可文件。

组件在 WebGL 不可用时静默降级为空背景，不能阻止笔记列表渲染。

### 6.7 Canvas 点阵

`AmbientGrid.tsx` 保留现有 `Path2D` 分桶绘制与鼠标弹性位移。

Effect cleanup 必须覆盖：

- resize、pointermove、visibilitychange 监听器。
- 进行中的 RAF。
- 点阵状态数组和 Canvas context 引用。

触摸设备和 `prefers-reduced-motion` 下只绘制静态点阵。

### 6.8 样式迁移

第一阶段把当前 CSS 原样迁移到 `src/styles/index.css`，只处理 JSX 必需差异：

- `class` 改为 `className`。
- 依赖动态 style 的缩略图改为 React style object。
- 保持现有层级、响应式断点、颜色和动画参数。
- 删除确实无法再引用的旧 DOM 选择器，但不顺带重新设计。

首次上线达到视觉一致后，再决定是否按组件拆 CSS。本次不拆。

## 7. `html-note` 技能改造

### 7.1 改造目标

技能生成一篇新笔记时，只允许修改：

- `public/notes/<slug>.html`
- `public/images/<slug>.jpg`，如果有缩略图
- `src/data/notes.json`

技能默认禁止修改：

- `src/components/**`
- `src/lib/**`
- `src/styles/**`
- 根 `index.html`
- `.github/workflows/**`
- `vite.config.ts`

这样新增内容不会影响 React 首页、光带、点阵和部署配置。

### 7.2 `SKILL.md` 修改

将现有步骤更新为：

1. 输出详情页到 `public/notes/<slug>.html`。
2. 图片压缩到 `public/images/<slug>.jpg`。
3. 生成临时 metadata JSON。
4. 调用 `node scripts/update-note-index.mjs --input <metadata.json>`。
5. 运行详情页结构校验。
6. 运行 `npm run validate:content`。
7. 运行 `npm run build`。

删除或改写以下旧约定：

- 删除“在 `index.html` notes 数组顶部插入”。
- 删除“必要时同步修改 `index.html` CSS”。未知 topic 统一降级为 `note`，技能不再新增 CSS class。
- 将 `buildRow` 保护规则改为“不修改 React 组件和 Counter service”。
- 将路径从 `notes/`、`images/` 改成 `public/notes/`、`public/images/`。
- 保留 topbar 的 `../index.html`。

根据 `skill-creator` 规范，实施时检查 frontmatter，只保留运行时真正支持且需要的字段；至少确保 `name` 和 `description` 清晰覆盖“生成独立 HTML、更新 React/Vite 内容索引”的触发场景。

### 7.3 确定性索引脚本

新增仓库脚本 `scripts/update-note-index.mjs`，不要让技能使用字符串替换编辑 JSON。

输入为单个 metadata JSON 文件。脚本职责：

- 使用 JSON parser 读取输入和 `src/data/notes.json`。
- 校验必填字段、slug、日期、topic、keywords 数量。
- 拒绝重复 slug、重复 title 和重复详情页路径。
- 校验详情页已经存在。
- `image` 非空时校验图片存在。
- 将新条目插入数组顶部。
- 使用 UTF-8、两空格缩进和结尾换行原子写回。
- 失败时不改原文件并返回非零退出码。

脚本不负责生成正文或图片，只负责 metadata 数据完整性。

### 7.4 校验器改造

现有 `.codex/skills/html-note/scripts/validate.py`：

- 增加 `--root`，不再把仓库绝对路径写死在代码中。
- 详情页位置改为 `<root>/public/notes/<slug>.html`。
- 图片检查位置改为 `<root>/public/images/`。
- 保留 HTML 结构、可访问性、tab、链接安全和 `&amp;` 检查。
- 增加“slug 已进入 `src/data/notes.json`”检查。
- PowerShell 版本与 Python 版本二选一作为主实现，避免长期维护两份等价逻辑；Windows 环境优先保留 Python 主实现，PowerShell 仅做薄包装。

新增仓库脚本 `scripts/validate-content.mjs`：

- 校验 notes JSON schema。
- 校验 slug、title 唯一。
- 校验日期降序。
- 校验每个条目对应的 HTML 和可选图片存在。
- 校验所有路径是相对路径且不逃出 `public`。
- 确认 `trace_e4a45e52` 未进入首页数据。
- 确认首页至少有一条数据。

### 7.5 参考文档修改

`references/design-index.md`：

- Canonical project 首页参考从单个 `index.html` 改成 `src/styles/index.css`、`src/components/NoteCard.tsx` 和线上首页。
- 详情页参考路径全部增加 `public/` 前缀。

`references/rules-guide.md`：

- Metadata 示例改成 JSON schema。
- “首页 Row 规则”改成 `src/data/notes.json` 规则。
- 删除 `buildRow` 和直接编辑首页的说明。
- 增加禁止修改 React 实现文件的边界。
- 最终清单增加 content validation 和 production build。

`references/house-catalog.md`、`references/plain-catalog.md`：

- 只更新示例文件路径。
- topbar 的 `../index.html` 不变。
- 不改变现有详情页视觉规范。

### 7.6 技能验证

实施完成后执行：

```powershell
python "C:\Users\hjl15\.codex\skills\.system\skill-creator\scripts\quick_validate.py" `
  "D:\Code\github\my-notes\.codex\skills\html-note"
```

再做三次前向测试：

1. 有缩略图的新笔记，确认页面、图片、JSON、构建全部成功。
2. 无缩略图的新笔记，确认 `image: null` 且卡片正常。
3. 重复 slug，确认脚本拒绝并且 JSON 没有部分写入。

`.codex/` 当前被 `.gitignore` 忽略，因此技能改造需要单独备份或同步到个人技能仓库；它不会随站点提交自动分发。

## 8. package scripts

目标 `package.json` 至少提供：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "typecheck": "tsc -b --pretty false",
    "test": "vitest run",
    "validate:content": "node scripts/validate-content.mjs",
    "check": "npm run validate:content && npm run typecheck && npm run test && npm run build"
  }
}
```

依赖保持最小：

- runtime：`react`、`react-dom`
- build：`vite`、`@vitejs/plugin-react`、`typescript`
- types：`@types/react`、`@types/react-dom`
- test：`vitest`

Playwright 继续用于发布前浏览器验收；实施时再决定作为本地 dev dependency 固化，还是沿用当前缓存命令。不要为了迁移引入完整 UI 测试框架。

## 9. GitHub Pages 改造

### 9.1 当前状态

- URL：`https://huangjl-cn.github.io/my-notes/`
- `build_type`：`legacy`
- Source：`main /`
- HTTPS：已启用
- 自定义域名：无

### 9.2 Actions 工作流

新增 `.github/workflows/deploy-pages.yml`：

1. push 到 `main` 或手动触发。
2. checkout。
3. setup Node LTS，并启用 npm cache。
4. `npm ci`。
5. `npm run check`。
6. 上传 `./dist` 为 Pages artifact。
7. 使用 `actions/deploy-pages` 发布。

权限限定为：

```yaml
permissions:
  contents: read
  pages: write
  id-token: write
```

使用 `github-pages` environment 和单一部署 concurrency group。Actions 版本在实施时按 Vite 官方 Pages 模板固定到明确 SHA，避免浮动版本。

### 9.3 Git 忽略

`.gitignore` 增加：

```gitignore
node_modules/
dist/
coverage/
playwright-report/
test-results/
```

`dist` 只存在于本机或 Actions 临时 runner，不提交到 `main`。

### 9.4 切换发布源

上线前在 GitHub：

1. Settings -> Pages。
2. Source 从 `Deploy from a branch` 改为 `GitHub Actions`。
3. 合并迁移分支到 `main`。
4. 等待 deploy job 成功。
5. 用 Pages API 确认部署 commit 与 `main` HEAD 一致。

旧 Pages artifact 在新部署成功前保持可访问。不要在本地构建通过前改 Pages Source。

## 10. 分阶段实施

### 阶段 0：基线与回滚点

工作：

- 从当前 `a9ee812` 创建 `pre-react-migration` tag。
- 保存桌面和移动端基线截图。
- 记录当前 9 条首页数据、10 个 HTML、9 张 JPG。
- 记录 Counter-X namespace 和 Pages 配置。

验收：当前线上站点与 tag 对应，工作区干净。

### 阶段 1：Vite 骨架与静态资产

工作：

- 创建 React + TypeScript Vite 配置。
- 移动 `notes`、`images`、favicon 和第三方声明到 `public`。
- 建立最小 `main.tsx`、`App.tsx` 和全局 CSS。
- 配置 `/my-notes/` base。

验收：`npm run build` 后 `dist/notes`、`dist/images` 和 favicon 完整，所有详情页返回链接有效。

### 阶段 2：首页数据与组件

工作：

- 将 9 条 notes 数据迁移到 JSON。
- 实现 Header、NoteCard、NoteList、Pagination、Footer、ScrollNav。
- 抽取 Counter-X service 和 site config。
- 修正 Footer 中旧的 GitHub 仓库链接。

验收：分页、点击数、链接、失败降级、键盘焦点与移动端行为和当前版本一致。

### 阶段 3：背景效果迁移

工作：

- 实现 AmbientWave 和 AmbientGrid。
- 完整实现 Effect cleanup。
- 保留 reduced-motion、visibility pause 和 WebGL fallback。

验收：React StrictMode 下无重复监听和重复 RAF；桌面/移动截图无明显视觉回归；Canvas 像素非空且不拦截 pointer events。

### 阶段 4：内容工具和技能

工作：

- 新增 metadata 更新和内容校验脚本。
- 更新 `html-note` 的 SKILL、references 和 validator。
- 完成三类技能前向测试。

验收：新增一篇测试笔记只修改允许的三个内容位置，React 组件 diff 为空，完整构建成功。

### 阶段 5：CI 与 Pages 切换

工作：

- 增加 Pages workflow。
- 在迁移分支完成 `npm ci`、`npm run check`。
- 切换 Pages Source 并合并到 `main`。
- 等待 GitHub Pages 状态为 `built`。

验收：线上 URL、笔记 URL、图片、Counter-X、背景动画和移动端全部正常。

### 阶段 6：清理

工作：

- 删除不再使用的旧内联首页实现。
- 确认没有重复资产和临时测试文件。
- 更新 README 的开发、构建和部署命令。

验收：`git status` 干净，`dist` 未被跟踪，README 能让新环境从零启动。

## 11. 验证矩阵

| 范围 | 检查 |
| --- | --- |
| 数据 | JSON schema、唯一 slug/title、日期顺序、文件和图片存在 |
| 类型 | `npm run typecheck` |
| 单元 | 分页边界、URL 生成、Counter-X fallback |
| 构建 | `npm run build`，无 warning/error，`dist` 内容完整 |
| 桌面 | 1440x1000，4 条/页，背景非空，无重叠或横向滚动 |
| 移动 | 390x844，文本不溢出，分页可用，Canvas 正确铺满 |
| 动效 | 光带往复、点阵鼠标响应、隐藏 tab 暂停 |
| 无障碍 | reduced motion 静态降级、focus visible、按钮 aria-label |
| 链接 | 笔记新窗口、`noopener noreferrer`、返回首页、Footer 仓库链接 |
| 部署 | Pages artifact 对应 main HEAD，线上资源无 404 |
| 技能 | 有图、无图、重复 slug 三个场景 |

## 12. 风险与控制

| 风险 | 控制措施 |
| --- | --- |
| Vite base 错误导致资源 404 | 固定 `/my-notes/`，通过 `import.meta.env.BASE_URL` 生成动态路径 |
| React StrictMode 启动两套动画 | 所有 Effect 完整 cleanup，专门测试 setup -> cleanup -> setup |
| Counter-X 历史计数丢失 | namespace 保持 `hjl-2005.github.io`，只修 Footer 仓库 URL |
| 技能误改组件 | JSON + 确定性脚本，SKILL 明确禁止实现层修改 |
| JSON 与 HTML 不一致 | `validate-content.mjs` 同时检查 metadata、页面、图片 |
| 独立笔记被 SPA 路由吞掉 | 不使用客户端 Router，保留真实 `public/notes/*.html` |
| 迁移期间 Pages 中断 | 功能分支完成全量检查后再切 Actions；保留旧 artifact 和回滚 tag |
| 第三方许可未进入 dist | 将 `THIRD_PARTY_NOTICES.md` 放在 `public` 并检查构建产物 |
| `.codex` 被忽略导致技能丢失 | 技能单独备份或同步到个人技能仓库 |

## 13. 回滚方案

如果新站上线后出现阻断问题：

1. 基于 `pre-react-migration` 创建 revert commit，恢复静态根目录结构。
2. GitHub Pages Source 改回 `Deploy from a branch`。
3. 选择 `main` 和 `/(root)`。
4. 等待 legacy Pages build 完成。
5. 验证首页和所有笔记 URL。

不要使用 `git reset --hard` 或强推回滚线上分支。

如果只是 React 实现问题但静态笔记正常，优先修复并重新运行 Actions，不切换发布模式。

## 14. 实施顺序检查表

- [ ] 建立 `pre-react-migration` tag 和视觉基线
- [ ] 创建 React + Vite + TypeScript 骨架
- [ ] 配置 `base: '/my-notes/'`
- [ ] 移动静态笔记、图片、favicon、第三方声明到 `public`
- [ ] 建立 `notes.json` 和 Note 类型
- [ ] 迁移首页组件、分页和 Counter-X
- [ ] 迁移光带与点阵，完成 Effect cleanup
- [ ] 增加内容更新和校验脚本
- [ ] 改造并验证 `html-note` 技能
- [ ] 添加 Pages Actions workflow
- [ ] 通过 `npm run check` 和 Playwright 验收
- [ ] 切换 Pages Source 并部署
- [ ] 核对线上 commit 与 Pages build
- [ ] 更新 README 并清理旧代码

## 15. 参考资料

- React `useEffect` 与外部系统清理：https://react.dev/reference/react/useEffect
- Vite public 目录：https://vite.dev/guide/assets.html#the-public-directory
- Vite GitHub Pages 部署：https://vite.dev/guide/static-deploy.html#github-pages
- GitHub Pages 自定义 Actions：https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages

