# 衣姿 PoseWear 逆向结论单

> 分析对象：公开站点 [https://posewear.pages.dev/](https://posewear.pages.dev/)（深链 [https://posewear.pages.dev/#pose](https://posewear.pages.dev/#pose)）  
> 分析时间：2026-11-24（Asia/Shanghai）  
> 方法：只读公开 HTML / JS / JSON / HTTP 头 / sitemap，并对照官方开源仓 [atuizz/posewear](https://github.com/atuizz/posewear)（MIT）。  
> 本仓库只放结构结论与字段名，不收录姿势/服装等素材图，不粘贴整段站点源码。

## 一句话定位

**衣姿 PoseWear 是一套部署在 Cloudflare Pages 上的纯前端参考图库 + 浏览器内 AI 试穿工作台**：12 类素材（姿势线稿、深度图、写实/动画服装、人像、表情、发型、历代服饰、场景、光影、道具、家具）以静态 JSON + CDN 图片提供浏览、筛选、收藏与中英文提示词；用户用自己的 API Key（或本机 ComfyUI / sd-cpp）在浏览器里把「人物 + 服装 + 姿势」等参考图送给图像模型生成试穿/出图。无需注册，也没有业务数据库。

它不是独立训练的「一键穿衣模型」，也不是必须安装的手机 App；手机浏览器可直接用，PWA 可安装。线上站点与源码同源，官方主页即 `posewear.pages.dev`。

## 证据（带 URL）

| 判断 | 证据 |
| --- | --- |
| 产品名与定位 | 首页 `<title>` / meta / JSON-LD：`衣姿 PoseWear · 姿势、服装、人像参考图库 \| 中英文提示词 + 在线 AI 试穿`；[首页](https://posewear.pages.dev/) |
| 部署 | 响应头 `server: cloudflare`，HSTS / CSP / NEL；主机名为 `*.pages.dev`；官方 README 与 `package.json` 的 `wrangler pages deploy --project-name posewear` |
| 源码仓 | 页脚与 README 指向 [https://github.com/atuizz/posewear](https://github.com/atuizz/posewear)，主页字段即本站 |
| 技术栈 | 无 React/Vue/Svelte；`src/*.js` + `build.py` 静态构建；vanilla JS 模块；测试与预览用 Node 22 + Playwright + wrangler |
| 图库数据 | 站点 `/assets/*.json` 共 12 份目录，见 [docs/证据摘录.md](docs/证据摘录.md)；首页预加载 `/assets/poses.2383bf449f.json` |
| 图片分发 | 前端 `AH='https://pwimg.0vo.pro/'`；相对路径拼到该域名。CHANGELOG（2026-11-21）写明「图片迁入 Cloudflare R2（pwimg.0vo.pro）」 |
| 热链例外 | CSP `img-src` 另放行 `https://charadb-2fi.pages.dev`、`https://ai-pose-depth-gallery.pages.dev`；发型部分条目的 `t`/`i` 为 CharaDB 绝对 URL |
| 哈希路由 | 首页品牌链到 `#pose`；`route()` 解析 `location.hash` 为 `#<tab>/[id]/[angle]`；工作台是 `#studio`（`/studio/` 线上 404） |
| SEO 路径 | 另有预渲染页，如 [ /pose/full_single_P001/ ](https://posewear.pages.dev/pose/full_single_P001/)、分类页 `/pose/sitting/`、`/en/` 镜像；[sitemap_index.xml](https://posewear.pages.dev/sitemap_index.xml) |
| 生成服务 | 前端直连或经 `/api/proxy` 转发到 OpenAI / Gemini / 火山方舟 Seedream / 自定义兼容接口；默认模型名见证据摘录 |
| 站点代理 | 源码 `functions/api/proxy.js`；线上 `OPTIONS /api/proxy` → 204，无 Origin 的 GET/POST → 403「只接受本站页面发起的请求」 |
| 本机引擎 | 前端请求 `/api/local/status`、`/api/local/generate`（头 `x-posewear-local: 1`）；**线上 Pages 无此 Function**（GET 返回 404 页面）。官方说明需 `npm start` 本地预览 |
| 本地隐私 | 密钥在 `localStorage.wbCfg`；收藏/最近/组合等亦在 localStorage；生成记录在 IndexedDB `lk-studio` / objectStore `h`，上限 200 条（官方 README） |
| 18+ 计数 | 首页 `data-n`（默认可见）与 `data-na`（含成人向）与 JSON 中 `a: 1` 标记一致，见下表 |

## 核心功能与交互流程

### 1. 浏览参考图（`#pose` 等）

1. 打开 [https://posewear.pages.dev/#pose](https://posewear.pages.dev/#pose)（空 hash 时前端也回落到 `pose` 页签）。
2. 顶栏「图库」菜单按四组切换 12 个素材库：姿态（姿势 / 深度姿势）、人物造型（人像 / 表情 / 发型）、服饰（服装 / 动画服装 / 历代服饰）、环境与物件（场景 / 光影 / 道具 / 家具）。
3. 可搜索、按分类/系列/角度（或服装的颜色/单品）筛选、排序、随机一条；点卡片进入灯箱看大图、多角度、中英文（部分含日文）描述。
4. 姿势卡片可在同一机位的**线稿**与**深度图**之间翻面（v1.3.0）；编号点击即切换库。
5. 「＋」把条目放进组合 / 悬浮生图工作台；可复制单条或组合提示词（默认可附参考图直链）。
6. 「我的」：收藏、最近浏览、生成记录。数据在本机。
7. 顶部「18+」：成人向（性感/内衣/双人互动等，JSON 字段 `a`）默认隐藏，首次需确认已满 18 岁（`ageOk`），之后用开关切换（`adult`）。

分享与 SEO 走路径页（例如 `/pose/full_single_P001/`），与哈希 SPA 并存；查询串可见 `?t=costume&q=` 或 `?q=` + hash。

### 2. AI 试穿 / 出图（`#studio` 与悬浮工作台）

可观察闭环（无需本站账号）：

1. **选素材**：从图库加号，或在悬浮工作台内「添加素材」。完整工作台默认展示人物、服装、姿势；表情/发型/场景/光影/道具/家具放在「其他素材」。
2. **人物来源**：选用库内人像，或上传最多 3 张本机照片（`blob:`）。
3. **模式**：试穿（`tryon`，有人物照片 + 服装）或直接文生/图生（`gen`）。可开「保持背景」、选画幅与张数、手改提示词（刷新可恢复草稿）。
4. **提示词组装**（`compose()` 模式概要，不抄模板全文）：
   - 按参考图序号引用「第 N 张 / image N」；
   - 人物、服装、姿势、表情、其他素材（发型/场景/光影/道具/家具）各有角色槽；
   - 可叠加风格预设（`STYLES`，如「照片质感」）与人物预设（`CHARS`，如「东亚女性」）及用户附加句；
   - 默认「减少误拦截」（`wbNeutral`）：发送前把提示词改写成中性电商试穿描述，不改服装与姿势（产品文案与开关可观察；完整改写规则未在本结论单逐句抄录）；
   - 复制到剪贴板时可附公开图直链（`PoseWearReferences.formatBrief`）。
5. **选服务商并填写用户自己的 Key**（只存本机 `wbCfg`）：
   - OpenAI：`https://api.openai.com/v1`，默认 `gpt-image-1`，有参考图走 `images/edits`（multipart），否则 `images/generations`；
   - Gemini：`https://generativelanguage.googleapis.com/v1beta`，默认 `gemini-2.5-flash-image`（产品亦称 Nano Banana），`models/:generateContent`；
   - 豆包 Seedream：`https://ark.cn-beijing.volces.com/api/v3`，线上默认模型为 `doubao-seedream-5-5-261028`（另列出 5.0 flash / 4.5 / 4.0 等）；国内直连常开本站代理以解决跨域；
   - 自定义中转：Base URL + Key + 模型名；可选 OpenAI 兼容的 edits/generations，或 Gemini 协议；
   - 本机：ComfyUI 或 stable-diffusion.cpp，只在 `npm start` 预览里通过 `/api/local/*` 到达回环引擎。
6. **传输路径**：默认浏览器**直连**官方/中转 HTTPS；勾选「通过本站代理」时改为 `POST/GET /api/proxy`，目标写在请求头 `x-target-url`，鉴权头原样转发（`Authorization` / `x-goog-api-key`）。代理有源站校验、路径白名单、同站 Origin/Referer 限制、内存+Cache API 近似限流（每 IP 每分钟 API 20 / 图片 40，源码自称尽力而为）。
7. **结果**：对比前后图、单张或打包 ZIP 下载（前端自实现 store-only ZIP）；写入本机生成记录，可再生成、删除、清空。

本站不代扣费、不保存密钥与生成图（官方声明 + 前端实现一致；代理源码为透传、`cache-control: no-store`）。未在本次分析中用真实 Key 跑通一次付费出图。

## 架构 / 技术栈可观察点

```
[手机/桌面浏览器]  PWA（manifest.webmanifest + sw.js）
        │  hash 路由 #pose/#studio/...
        │  静态 HTML（首页 SPA + 部分 SEO 预渲染页）
        ▼
 Cloudflare Pages  (posewear.pages.dev)
        ├─ /assets/*.js|css|json     构建哈希文件名
        ├─ /api/proxy                Pages Function（生产已启用）
        └─ /api/local/*              仅本地预览，生产 404
        ▼
 图片 CDN  https://pwimg.0vo.pro/     （changelog：Cloudflare R2）
 热链      https://charadb-2fi.pages.dev/  （部分发型）
        ▼
 用户自备的图像 API
   api.openai.com / generativelanguage.googleapis.com
   ark.cn-beijing.volces.com / 自定义 HTTPS 中转
   （本地）ComfyUI 或 sd-cpp，经预览进程回环
```

- **构建**：Python `build.py` 把 `src/` 打到 `dist/`，素材目录 JSON 带内容哈希；`scripts/fetch-assets.py` 从上游图库拉数据（官方仓本身不含图片）。
- **前端**：原生 JS（`app.js` 图库/路由，`studio.js` 工作台，`image-providers.js` 服务商适配，`history.js` 记录，`i18n.js` 中英日，`theme.js` 亮暗色，`zip.js`，`references.js` 提示词附图链；亦可被 MCP 复用）。
- **离线**：SW 版本 `posewear-7fbb77b9f4`，预缓存壳与姿势/深度 JSON；同源 GET 且非 `/api/` 为网络优先再写入缓存。
- **智能体接口**：官方提供只读 MCP（`mcp/server.mjs`）与选图 Skill，用于搜库、选角度、组合提示词；本次未实际启动 MCP。
- **安全头**：CSP 较严（`script-src 'self'` + 一条 inline sha256；`frame-ancestors 'none'`）；但 `connect-src` 为 `https:`，以便浏览器直连各家 API。`robots.txt` 禁止抓 `/api/`、`/o/`。

### 线上库规模（2026-11-24 抓取，与 JSON / 首页 data-n、data-na 核对）

| 库 | 默认可见 `data-n` | 含 18+ `data-na` | JSON 条数 | 备注 |
| --- | ---: | ---: | ---: | --- |
| 姿势 pose | 98 | 288 | 288（`a=1` 共 190） | 角度图合计 1439 |
| 深度姿势 depth | 233 | 288 | 288（`a=1` 共 55） | 角度 1439；图在 pwimg `/d/` |
| 服装 costume | 1068 | 1407 | 1407（`a=1` 共 339） | |
| 动画服装 anime | 1380 | 1760 | 1760（`a=1` 共 380） | sitemap 详情仅 984 条/语种 |
| 人像 portrait | 1499 | 1499 | 1499 | sitemap 似无 `/en/` 详情 |
| 表情 / 发型 / 历代 / 场景 / 光影 / 道具 / 家具 | 95 / 122 / 57 / 64 / 50 / 32 / 41 | 同左 | 同左 | 未见独立 sitemap |

## 可复刻要点（产品层，不抄资源）

若要做「手机端参考库 + 用户自带 Key 出图」，可借鉴的是**产品结构**，不要搬运其姿势/服装图库：

1. **静态优先**：图录做成带哈希的 JSON + CDN 图，站点无账号体系；密钥、收藏、历史只放本机。
2. **双通道路由**：哈希 SPA 负责交互（`#库/id/角度`），路径 SSG 负责可分享详情与搜索收录；两者 ID 对齐。
3. **宿主能力在云、控制在端**：站点只提供目录、提示词拼装与（可选）同源代理；真正推理走用户的 OpenAI/Gemini/Seedream/中转/本机模型。
4. **提示词套路**：素材对象同时存「短标题 `ti` + 中文描述 `zh` + 英文描述 `en` + 检索关键词 `kw`」；出图时按槽位编号引用参考图（人物 → 服装 → 姿势 → 表情 → 场景/光影/道具…），并提供「中性电商试穿改写」降低误审，而不是绕过审核。
5. **工作台分层**：悬浮条随时加减素材；完整台把高频槽（人/衣/姿）与低频槽（其他）分开；步骤可见、可改提示词、可中断、结果可回看。
6. **成人内容默认折叠**：用数据标记（`a`）+ 一次年龄确认 + 随时开关，sitemap 只放默认可见条目。
7. **代理边界写死**：仅同站发起、仅 HTTPS、官方域名+白名单路径、自定义中转需单独登记 origin、拒绝内网/带用户名密码的 URL、鉴权头透传且不落盘。
8. **工具链分离**：源码仓不进素材图；拉数脚本、R2 同步、CSP 白名单由构建校验（`tests/check_csp.py`）守卫热链域名。

## 未知项

- Cloudflare 账号、Pages 项目后台、R2 桶名、`0vo.pro` 域名注册与缓存策略的具体配置（只见到自定义域与 changelog 陈述）。
- 生产环境变量 `POSEWEAR_PROXY_ORIGINS` 是否登记了任何自定义中转源（源码有该开关，值未公开）。
- 控制台是否另为 `/api/proxy` 配置了正式 Rate Limit / WAF（源码注释建议另加；当前可见的是 Function 内近似计数）。
- 「减少误拦截」改写函数的完整句式规则（只确认有开关与产品说明，本仓库不逐句抄模板）。
- 动画服装 sitemap（984）少于默认可见 1380 的构建原因：可能是后并入的 CharaDB 子集未进详情页；未逐条对照 `build.py` / `seo.py`。
- 人像 sitemap 未收录 `/en/portrait/{id}/` 的原因。
- `depths` JSON 里 `pack.source` 仍为 `hotlink`，与 changelog「深度图改为 R2 自存」不完全一致，以线上图片 URL（`pwimg.0vo.pro/d/...`）为准。
- 未用浏览器完整点击一遍所有筛选/导入导出/PWA 安装；未启动官方 MCP；未使用真实 API Key 做一次生成（故端到端成功率、计费与各家审核表现不在本单「已证实」范围）。
- `/o/` 在 `robots.txt` 中被禁止，线上原图路径在 JSON 里写作 `o/p/...` 等并指向图片 CDN；站点自身是否仍托管 `/o/` 未作穷尽探测。

## 复刻边界

- **不要**把 PoseWear 的姿势、服装、人像等图包或完整 JSON 内容拷进自己的产品或本仓库。素材版权归原作者（主要是 @giaaibusi / ギア、@agi_aibusi、CharaDB たき AI、WaveSpeed、Shisei 等），官方仓的 MIT 只覆盖代码与文案。
- **不要**绕过 18+、代理同源校验、服务商安全审核或他人 API 鉴权。
- **不要**把用户 Key 放到自己的服务器「代填」而不作边界说明；原站默认是浏览器直连。
- 可公开复用的是：信息架构、路由与数据字段设计、提示词槽位思路、自带 Key 的服务商适配方式、以及官方 MIT 源码（按其许可，并保留致谢）。
- 本结论单是对公开面的可复核拆解，不是内部实现说明书，也不是攻击指南。

## 参考链接

- 线上站：[https://posewear.pages.dev/](https://posewear.pages.dev/) · [深链 #pose](https://posewear.pages.dev/#pose)
- 官方源码（MIT）：[https://github.com/atuizz/posewear](https://github.com/atuizz/posewear)
- 字段、资产清单、请求形态：[docs/证据摘录.md](docs/证据摘录.md)
