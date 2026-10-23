# 第三方致谢

paper-search-pro 站在一个慷慨的开源生态之上。本文档列出打包进编译产物
`bundle.html`、或通过 Python 运行时引入的主要第三方项目，以及它们的许可证
与主页。

每个 npm 包的完整许可证文本，在 `npm install` 之后位于它自己的
`node_modules/<package>/LICENSE` 里；按照生态惯例，它们不在本仓库中重新
分发。下面每个项目的主页都指向权威的许可证文件。

---

## UI 基础组件

| 包 | 许可证 | 主页 |
|---|:---:|---|
| **shadcn/ui** — `src/components/ui/*.tsx` 下的 41 个组件文件派生自 shadcn/ui 的复制粘贴目录 | MIT | <https://ui.shadcn.com/> |
| **Radix UI** — 27 个 `@radix-ui/react-*` 无样式可访问性基础组件 | MIT | <https://www.radix-ui.com/> |
| **Lucide** — 报告中通篇使用的图标集 | ISC | <https://lucide.dev/> |
| **Sonner** — toast 通知基础组件 | MIT | <https://sonner.emilkowal.ski/> |
| **Vaul** — 抽屉基础组件 | MIT | <https://vaul.emilkowal.ski/> |
| **cmdk** — 命令菜单基础组件 | MIT | <https://cmdk.paco.me/> |
| **Embla Carousel** — 轮播基础组件 | MIT | <https://www.embla-carousel.com/> |
| **React Day Picker** — 日历基础组件 | MIT | <https://react-day-picker.js.org/> |
| **React Resizable Panels** — 可调整尺寸的布局基础组件 | MIT | <https://github.com/bvaughn/react-resizable-panels> |

## 图表与可视化

| 包 | 许可证 | 主页 |
|---|:---:|---|
| **Recharts** — 驱动 Methods 标签页的可组合图表库 | MIT | <https://recharts.org/> |

## 表单与校验

| 包 | 许可证 | 主页 |
|---|:---:|---|
| **React Hook Form** | MIT | <https://react-hook-form.com/> |
| **@hookform/resolvers** | MIT | <https://github.com/react-hook-form/resolvers> |
| **Zod** — 结构校验 | MIT | <https://zod.dev/> |

## 样式

| 包 | 许可证 | 主页 |
|---|:---:|---|
| **Tailwind CSS** | MIT | <https://tailwindcss.com/> |
| **tailwindcss-animate** | MIT | <https://github.com/jamiebuilds/tailwindcss-animate> |
| **class-variance-authority** | Apache 2.0 | <https://cva.style/> |
| **clsx** | MIT | <https://github.com/lukeed/clsx> |
| **tailwind-merge** | MIT | <https://github.com/dcastil/tailwind-merge> |

## 字体（以 base64 内联在 `src/index.css` 中）

| 字体 | 许可证 | 主页 |
|---|:---:|---|
| **Geist Variable** — Vercel 的几何无衬线字体（拉丁） | SIL OFL 1.1 | <https://github.com/vercel/geist-font> |
| **Geist Mono Variable** — Vercel 的等宽字体 | SIL OFL 1.1 | <https://github.com/vercel/geist-font> |
| **Noto Sans SC Variable** — Google 的泛中文无衬线字体（CJK） | SIL OFL 1.1 | <https://fonts.google.com/noto/specimen/Noto+Sans+SC> |

## React 框架与构建工具链

| 包 | 许可证 | 主页 |
|---|:---:|---|
| **React** + **React DOM** | MIT | <https://react.dev/> |
| **Vite** — 打包器 | MIT | <https://vitejs.dev/> |
| **TypeScript** | Apache 2.0 | <https://www.typescriptlang.org/> |
| **next-themes** | MIT | <https://github.com/pacocoursey/next-themes> |
| **date-fns** | MIT | <https://date-fns.org/> |
| **html-inline** — 单文件 HTML 编译步骤 | MIT | <https://github.com/popcorn/html-inline> |
| **PostCSS** + **autoprefixer** | MIT | <https://postcss.org/> |

## Python 运行时（`scripts/requirements.txt`）

`scripts/requirements.lock` 是这些同样的运行时包及其传递依赖的、生成的精确
版本快照。它不引入任何运行时组件；`requirements.txt` 仍然是默认的可移植安装
路径。

| 包 | 许可证 | 主页 |
|---|:---:|---|
| **pyalex** — OpenAlex 客户端 | MIT | <https://github.com/J535D165/pyalex> |
| **semanticscholar** — Semantic Scholar 客户端 | MIT | <https://github.com/danielnsilva/semanticscholar> |
| **biopython** — 经 NCBI Entrez 访问 PubMed | Biopython License（BSD 类） | <https://biopython.org/> |
| **arxiv** — arXiv API 客户端 | MIT | <https://github.com/lukasschwab/arxiv.py> |
| **requests** — HTTP 客户端 | Apache 2.0 | <https://requests.readthedocs.io/> |
| **PyYAML** — YAML 加载器 | MIT | <https://pyyaml.org/> |
| **Jinja2** — 模板引擎 | BSD-3-Clause | <https://palletsprojects.com/p/jinja/> |

## 项目脚手架工具

| 来源 | 许可证 / 状态 | 说明 |
|---|:---:|---|
| **Anthropic Web Artifacts Builder**（Claude Code Skill） | Skill 产出归用户所有 | 为 `assets/webartifacts_app/paper-report/` 提供了最初的 Vite + React + TypeScript + Tailwind + shadcn 脚手架；由此得到的编译模式（`vite build` + `html-inline` → 单文件 `bundle.html`）沿用这个 Skill 推荐的模式 |

---

## 贡献者

| 姓名 | 角色 |
|---|---|
| **Bo** | 原作者 —— Skill 设计、14 步流程、Python 辅助脚本、React 报告渲染器、双语 UI、截图流水线 |
| **paper-qa**（FutureHouse / Edison Scientific） | 上游确定性辅助脚本模式（[`scripts/vendored/`](scripts/vendored/README-vendored.md)） |
| **shadcn** + **shadcn/ui** 社区 | `src/components/ui/` 下的 UI 组件基础组件 |
| **Anthropic** | Claude Code Skill 平台、Web Artifacts Builder Skill 脚手架 |
| **OpenAlex · Semantic Scholar · CrossRef · NCBI / PubMed · arXiv** | 开放的学术元数据 API —— 本 Skill 的全部前提 |

如果你参与贡献并希望被列在这里，请开一个 pull request，写上你的名字 + 角色。

---

*本文件并不穷尽。上表所列包的传递依赖会把更多开源库带进编译产物；每一个都
受其所在 npm 包内自带许可证的约束。*
