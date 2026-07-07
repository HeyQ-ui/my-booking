# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

纯静态、纯前端的中山大学体育场馆预约**演示页面**。没有构建系统、没有依赖管理、没有后端。每个页面是一个独立的 HTML 文件，用 `<script>` 内联原生 JavaScript（ES5/ES6 混用，无框架、无打包）实现逻辑。页面之间通过 **URL 查询参数**传递状态，用 `window.location.href` 跳转。

没有 `package.json`、没有测试、没有 lint 配置。「运行」= 用浏览器打开 HTML 文件；「测试」= 手动点击走完流程。

## 文件结构与用途

| 文件 | 用途 |
|------|------|
| `index.html` | 流程第 1 步：选择预约日期。手写的简单表单页，点击后跳转 `booking.html?date=YYYY-MM-DD`。 |
| `booking.html` | 流程第 2 步：选择场次。**场次数据硬编码在页内 `sessions` 数组**，按 `campusGroups` 分校区折叠展示，支持搜索。选中后跳转 `我的预约.html?date=..&venue=..&time=..&type=..`。 |
| `我的预约.html` | 流程第 3 步：预约结果页 + 详情悬浮层。**特殊：由 SingleFile 保存的中山大学官方 Vuetify 页面**（`gym.sysu.edu.cn`），我们在末尾追加了两段自定义 `<script>` 和一段 `<style>` 来「篡改」页面内容并加交互。 |
| `我的预约（点击1）.html` | **参考素材，非运行文件**。同样是 SingleFile 保存的官方页面，但抓取时官方的预约详情悬浮层处于打开状态。悬浮层的精确 HTML 结构与 Vuetify CSS 规则从这里提取，用于复刻到 `我的预约.html`。修改悬浮层外观时应以此文件为「标准答案」比对。**被 `.gitignore` 排除，不纳入仓库**（见下）。 |
| `场馆预约信息.xlsx` | 场次数据的原始来源（`booking.html` 里的 `sessions` 数组据此录入）。 |
| `.gitignore` | 排除参考素材 / 标准化快照文件（`*（点击*）.html`、`*（标准*）.html` 等），使它们不被 git 跟踪。 |

## 参考素材 / 标准化文件不纳入版本控制

`我的预约（点击1）.html` 这类文件是官方页面的 SingleFile 抓取快照，**仅作复刻悬浮层、比对样式的「标准答案」用**，体积大且非运行必需，因此通过 `.gitignore` 排除，不提交进仓库。命名带「（点击N）」「（标准…）」等后缀的对照文件都遵循此约定——需要时本地留存即可，不要 `git add`。

## 关键架构点

### 页面间数据流
状态全部走 URL 查询参数，没有 localStorage、没有全局状态：
```
index.html ──date──► booking.html ──date,venue,time,type──► 我的预约.html
```
读取参数统一用 `new URLSearchParams(window.location.search)`。日期用本地时区手动拼 `YYYY-MM-DD`（见各文件的 `todayLocalStr()`），**刻意避开 `valueAsDate` 的 UTC 偏移坑**——改日期逻辑时注意保持这一点。

### `我的预约.html` 是「改造过的官方页面」，不是手写页面
这是本仓库最反直觉的地方：

- 文件约 1MB，主体是 SingleFile 内联的 Vuetify 2.7 应用（含 base64 图片、压缩 CSS），**几乎全部内容在少数几行超长行里**（尤其第 20 行）。用 `Read` 直接读会命中大小限制；用 `grep -oP` 提取具体片段，而非通读。
- 我们自己的代码只在文件**末尾**：一段 `<style>`（第 23 行起）+ 两段 `<script>`。第一段 script 根据 URL 参数改写官方预约卡片（`.card-item`）的标题/日期/时段/场地；第二段 script 实现「点击卡片弹出悬浮层」。
- **该文件缺失部分 Vuetify CSS 规则**（`.v-dialog`、`.v-dialog__content`、`.v-card` 白底、`.v-btn:not(.v-btn--outlined).primary{color:#fff}` 等），这些规则只存在于 `我的预约（点击1）.html`。所以复刻悬浮层时，除了拷贝 HTML，还必须把缺的 CSS 规则一并补进我们的 `<style>` 块（已作用域限定在 `.booking-detail-*` 前缀下，避免污染官方样式）。

### DOM 选择器约定（官方页面结构，勿改）
悬浮层交互依赖官方页面里的这些 class，改动时保持一致：
- `.card-list` → 卡片容器；`.card-item` → 单张预约卡片
- 卡片内：`.booking-title`（场馆名）、`.booking-id`（`#RB-... 运动时 ¥5`）、`.card-details > div`（图标分隔的 日期/时段/场地 文本节点）、`.card-actions button`（原「取消」按钮，已被脚本移除）
- 悬浮层：`.v-overlay`（遮罩）+ `.v-dialog__content > .v-dialog > .v-card`，官方组件带 `data-v-578881b0` 属性

悬浮层内容是**运行时从被点击卡片的 DOM 解析**（`parseCard()`），不是硬编码，从而保证悬浮层与卡片显示一致。

## 处理超大 HTML 文件的注意事项

`我的预约.html` 和 `我的预约（点击1）.html` 都是 1MB 级、含超长单行的文件：

- 用 `grep`/`grep -oP` 提取需要的片段，不要整文件 `Read`。
- 编写正则时**避免灾难性回溯**：不要用 `[^{};]*something[^{]*\{[^}]*\}` 这类在超长行上会挂起的模式（本项目已踩过坑）。优先用固定、有界的模式（如 `\.booking-title\{[^}]{0,150}\}`）。
- 修改我们自己的代码时，用末尾行号定位（`grep -n` 找锚点），用 `Edit` 精确替换，不要重写整文件。

## 常用操作

- 预览：浏览器打开 `index.html`，或 `python -m http.server 8000` 后访问。
- 新增/修改场次：编辑 `booking.html` 里的 `sessions` 数组与 `campusGroups` 分组。
- 改悬浮层外观：以 `我的预约（点击1）.html` 为准提取结构/CSS，改 `我的预约.html` 末尾的 `<style>` 与第二段 `<script>`。
- 提交信息：本仓库使用中文 commit message。
