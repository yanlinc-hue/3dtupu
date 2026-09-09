# 部署说明（中文）

本文件说明 2026-09-09 0.5.0 版本的实际结构。普通使用者直接访问 [3dtupu.com](https://3dtupu.com)，不需要部署。

## 先确认拿到的是哪份仓库

| 仓库／目录 | 用途 | 能否直接开发 |
| --- | --- | --- |
| 公开 `yanlinc-hue/3dtupu` | GitHub Pages 编译产物、中文文档、虚构示例 | 不能，里面不是完整源码 |
| 私有 `yanlinc-hue/3dtupu-core/dealgraph` | React / Vite 前端及 Cloudflare Worker 源码 | 取得授权后可以 |
| 仓库根 `.openai/hosting.json` | 旧私有 Sites 配置 | 不是本次公开站点的发布方式 |

不要把私有源码改成公开仓库，也不要把客户文件放进任一发布目录。部署需要：私有源码权限、Node.js 24 与 npm、目标 Pages 仓库权限、Cloudflare 账号和域名区域的 Worker / 路由权限。自己的新站应使用自己的账号、域名和仓库；不要沿用维护者的账号 ID 或区域 ID。

## 1. 本地运行与构建

已取得私有仓库权限后：

```sh
git clone git@github.com:yanlinc-hue/3dtupu-core.git
cd 3dtupu-core/dealgraph
node --version
npm ci
npm run test:hosted
npm run test:graph
npm run build
npm run dev
```

打开 `http://127.0.0.1:5174`。当前 `npm run dev` 使用 5174，含同源本地 API 桥接；请用这个精确地址，不要换成局域网 IP 或对外监听。本地桥接调用同一份 Worker 处理逻辑，再访问固定的模型服务商接口，**不把密钥转交公开网站**。

需要模型时，在页面临时输入自己的 API 密钥并同意上传。不要把密钥放进 `.env`、命令行、配置文件或 Git。`npm run preview` 只是构建后的静态预览，未挂载完整模型 API，不能拿它代替上述开发服务。

只有在本机已有可信代理且直连受阻时，才显式使用本地配置，例如：

```sh
DEALGRAPH_LOCAL_PROXY=http://127.0.0.1:12334 npm run dev
```

端口必须是自己已启用的代理；这不是必需步骤。该设置只作用于本地预览，不进入浏览器或生产 Worker，不关闭 TLS 校验。

## 2. 理解线上两部分

```text
浏览器访问自己的 HTTPS 域名
  → Cloudflare Worker
      /api/relationships/* → 本次用户选定的模型服务商
      其余路径             → GitHub Pages 静态文件
```

**只上传 Pages，网页可以显示，但不能独立调用模型。** 本轮涉及 Worker 逻辑更新，必须一起发布前端与 Worker；不能照旧说明写成“后端未修改，无需发布”。

## 3. 校准域名与配置

现有站点使用 `https://3dtupu.com`。部署到其他域名时，以下位置必须一致：

| 位置 | 要检查什么 |
| --- | --- |
| `wrangler.jsonc` | 自己的 `name`、`account_id`、路由 `pattern`、`zone_id`；路由覆盖目标域名 `/*` |
| `server/worker.ts` | `ORIGIN`、HTTP → HTTPS 跳转中的域名、相关域名提示 |
| `server/local-preview.ts` | 内部构造 Worker Request 的域名和 Origin，与 Worker 保持一致 |
| Pages 设置及公开根目录 `CNAME` | 相同自定义域名；Pages 发布根目录与实际发布分支匹配 |
| Cloudflare DNS / TLS | 该域名指向 Pages 源、由 Cloudflare 代理并能命中 Worker 路由；HTTPS 证书正常 |
| `tests/` 与中文文档 | 修改测试用 Origin 与网址，不放宽为任意来源 |

前端通过同源 `/api/relationships/*` 调用，不能只把 API 换成另一域名再打开通配 CORS。`wrangler.jsonc` 的 `API_LIMIT`、`GLOBAL_LIMIT` 限流绑定也必须部署；缺少绑定时服务会拒绝模型操作。保留经验证的兼容日期与取消信号设置，变更后重新测试。

## 4. 发布静态文件与 Worker

在 GitHub 的 Settings → Pages 中使用 **Deploy from a branch → main → /(root)**；自定义域名也需在 Pages 设置中配置，不能只写 CNAME。保留 `.nojekyll`。本次现有站点已是该方式。

先完成测试和构建，再将以下白名单内容发布到公开 Pages 仓库根目录：

- `dist/index.html`、`dist/assets/`。
- 审核过的中文文档、全虚构 `examples/`、演示 `demo/`。
- 正确的 `CNAME`、既有 Pages 必需配置和第三方许可声明。

不要发布整个私有仓库或 `dealgraph/`；不要带入 `node_modules`、`.env`、`.dev.vars`、源码 map、密钥、客户文件、真实微信材料、测试原始响应或本地日志。以新提交更新版本，保留回滚记录。Worker 先发布，再发布配套页面；接口契约为 `commercial-relations-v3`，旧页面会在付费调用前收到版本拒绝，请先加密保存再刷新。

在私有源码的 `dealgraph/` 使用本轮固定的 Wrangler **4.92.0**：

```sh
npx --yes wrangler@4.92.0 --version
npx --yes wrangler@4.92.0 deploy --dry-run --config wrangler.jsonc
npx --yes wrangler@4.92.0 deploy --config wrangler.jsonc
```

当前 `package.json` 不安装 Wrangler；上述命令使用指定版本，不使用未经验证的 `latest`。首次执行会按需取得该版本。首次部署先运行 `npx --yes wrangler@4.92.0 login`，在浏览器中用自己的 Cloudflare 账号授权。Cloudflare 登录凭据仅用于部署，不是模型服务商密钥。

无需部署站长的 `OPENAI_API_KEY`，也不要添加共享密钥：生产接口是 BYOK，每个用户仅在本次请求提交自己的所选服务商 key。

若页面嵌入演示视频，须同时核对 `server/worker.ts` 的响应 CSP 与 `vite.config.ts` 注入的 CSP，允许同源媒体；只改其中一处仍可能无法播放。不要为视频放开任意脚本、任意连接地址或任意媒体域名。

HTML 响应同时保留 `Cache-Control: no-store, no-transform`，禁止 CDN 改写页面或自动插入统计脚本；公网验收应检查实际 HTML，不只检查仓库源码。这个行为见 [Cloudflare Web Analytics FAQ](https://developers.cloudflare.com/web-analytics/faq/)。不要为消除浏览器告警而放宽 `script-src` 或 `connect-src`。

## 5. 发布后验收与回滚

1. HTTPS 首页、四个中文入口、静态资源、文档、示例和中文字幕视频可用。
2. `/api/relationships/status` 返回本站 JSON，而非 Pages HTML；配置密钥之前不会自动调用模型。
   同时检查页面与后端接口契约匹配；若页面提示版本不兼容，先补齐匹配的前后端发布，不绕过检查。
3. 在页面检查服务；未配置密钥、未同意上传、错误来源均被拒绝。连接诊断不验证密钥或余额，不代表模型已成功。
4. 用全虚构小样本检查导入、范围过滤、名单确认、取消、加密保存和恢复。真实模型测试必须另获用户同意；产生的费用由其 key 账户承担。
5. 记录私有源码提交、公开产物提交、Worker 版本、测试时间与未完成项，才标记“本轮已上线”。

回滚应协调恢复匹配的前端和 Worker 版本，保留域名、Pages 配置与原有路由。仅修改本项目范围，不删除整个仓库、Cloudflare 区域或其他站点配置。

## 官方参考

[GitHub Pages 发布来源](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site)说明分支、根目录与发布任务；[Cloudflare Worker Routes](https://developers.cloudflare.com/workers/configuration/routing/routes/)说明现有源站前的路由部署；[Wrangler 命令](https://developers.cloudflare.com/workers/wrangler/commands/)列出部署命令。
