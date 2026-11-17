# RSSHub 二次开发与维护指南

新增网站通常只需在 `lib/routes/` 下增加路由，复用现有请求、缓存和输出机制。先完成真实站点抓取、生产构建和镜像验收，再增加共享采集层或独立任务系统。

本文基于提交 `38638e6947212e02b465d2bacb9c60caf36fcb92`，核查日期为 2026-11-17。环境安装、订阅方式和部署配置见[使用指南](usage.md)。长期维护分支 `custom-main` 已在本地和远程创建；示例路由、部署模板和发布流程仍是后续操作指导，尚未部署或运行验收。具体状态见[维护记录](maintenance.md)。

## 1. 仓库关联和分支约定

本项目的远程仓库约定为：

| 名称 | 地址或用途 |
| --- | --- |
| `origin` | `https://github.com/Shonee/RSSHub.git`，自有 fork |
| `upstream` | `https://github.com/DIYgod/RSSHub.git`，官方上游 |
| 本地 `master` | 跟踪 `origin/master`，保留真实的远程提交基线 |
| `custom-main` | 长期自定义维护分支，已创建并跟踪 `origin/custom-main` |
| `feat/*`、`fix/*` | 具体新增站点或修复分支 |
| `chore/sync-*` | 上游升级和兼容性验收分支 |

本地目录原先没有 Git 元数据。已从 `origin/master` 获取真实提交，以浅仓库恢复本地 `master` 和跟踪关系，未覆盖原有源码。随后从该基线创建并切换到 `custom-main`，在 GitHub 创建同名分支并建立跟踪关系；`master` 仍跟踪 `origin/master`。使用指南、二次开发指南和维护记录随文档初始化提交纳入版本控制。

检查当前状态：

```bash
git remote -v
git branch -vv
git status --short
git rev-parse --is-shallow-repository
```

浅仓库仍可以正常开发和提交，但历史只覆盖已取得的范围。需要完整历史或合并上游前，在 Git 网络正常后执行：

```bash
git fetch --unshallow origin
git fetch upstream master
```

只有 `--is-shallow-repository` 返回 `true` 才使用 `--unshallow`；完整仓库直接正常 fetch。添加远程和读取公开仓库不代表具有推送权限，推送仍需要 GitHub 身份凭据。

本仓库的维护分支已初始化，不再重复创建。日常开发先提交或妥善保存当前改动，再从维护分支创建功能分支，例如：

```bash
git switch custom-main
git pull --ff-only origin custom-main
git switch -c feat/my-site
```

功能修改通过 PR 合并到 `custom-main`，不要以 `master` 为自定义功能的合并目标。`custom-main` 的推送目标已显式设置为 `origin`。GitHub 默认分支仍为 `master`，本次未修改默认分支。

Fork 包含 [sync-fork.yml](../../.github/workflows/sync-fork.yml)，会尝试同步 fork 默认分支；包含工作流不表示 GitHub 上已经启用。默认分支承载自定义改动前，应检查同步策略。现有 [docker-release.yml](../../.github/workflows/docker-release.yml)围绕 `master` 发布，并有 `DOCKER_USERNAME` 等条件，不应假定 fork 后推送就会得到自己的镜像。

## 2. 理解代码执行路径

```text
lib/index.ts
  → lib/app.ts / lib/app-bootstrap.tsx
  → 访问控制、日志、参数、缓存等中间件
  → lib/registry.ts / lib/registry-helpers.ts
  → lib/routes/<namespace>/<route>.ts 的 handler
  → 返回 Data，经参数处理和模板转换
  → RSS / Atom / JSON Feed
```

中间件在进入路由和路由返回后分别执行；路由返回标准数据，由框架统一输出，无需手工拼接 XML。

| 位置 | 修改时机 |
| --- | --- |
| `lib/routes/<site>/namespace.ts` | 网站名称、域名、命名空间说明 |
| `lib/routes/<site>/<route>.ts` | 路由路径、参数、元数据、采集与 Feed 映射 |
| `lib/routes/<site>/utils.ts` 等辅助模块 | 同一网站确有共享逻辑时复用 |
| `lib/utils/` | 确实适用于多个网站的公共工具 |
| `lib/config.ts` | 需要新增正式环境变量和类型定义时 |
| `lib/middleware/`、`lib/registry*` | 有明确框架层需求时；新增网站通常不涉及 |
| `scripts/workflow/build-routes.ts` | 生成注册表、Radar 和路径类型，不手改构建产物 |
| `lib/pkg.ts` | 以库方式使用的 `init`、`request`、`registerRoute` |

开发模式按命名空间延迟导入，生产模式读取 `assets/build/routes.js` 等构建结果。构建会加载路由模块，所以源站请求、浏览器启动和必须依赖凭据的运行校验放在处理函数里，避免模块顶层产生网络副作用。

参考维护流程提出的 `lib/custom/collectors/`、`deploy/` 和自有 Actions 是可选组织方式，目前没有为此创建目录骨架或自动化。简单网站先使用现有站点目录；只有复杂采集逻辑需要独立复用时再拆分。

## 3. 开发一条路由

### 明确输入与输出

开始前确定真实来源网页、用户或栏目参数、API/HTML/浏览器取数方式、站点凭据、文章链接、发布时间和异常情况。先找稳定 API，再考虑 HTML，最后使用浏览器。RSS 只取第一批内容，不设计翻页参数。

环境准备见[源码启动](usage.md#2-从源码启动)。生成器由当前 Node 直接运行，查看帮助不需要安装项目依赖：

```bash
node scripts/generate-route.mjs --help
```

安装依赖后，可以使用：

```bash
pnpm new:route
```

生成器会询问命名空间、文件名、展示名、来源网页、GitHub 维护者、分类和 `api`、`html`、`browser` 模板；会校验维护者和来源地址，并拒绝覆盖已有路由。其检查需要外网可达。生成骨架后仍需完成 API 字段或 HTML 选择器。

### 示例：将 GitHub Releases 转为订阅源

以下是完整的教学示例，使用真实 API 地址；未作为业务路由加入仓库，未执行线上抓取验收。生产中也可以直接使用 GitHub 提供的官方 Releases Atom。示例用于说明 RSSHub 的文件组织、响应校验和字段映射。

文件 `lib/routes/my-github/namespace.ts`：

```typescript
import type { Namespace } from '@/types';

export const namespace: Namespace = {
    name: 'My GitHub',
    url: 'github.com',
};
```

文件 `lib/routes/my-github/releases.ts`：

```typescript
import MarkdownIt from 'markdown-it';
import { z } from 'zod';

import type { Route } from '@/types';
import ofetch from '@/utils/ofetch';
import { parseDate } from '@/utils/parse-date';

const markdown = new MarkdownIt();
const releasesSchema = z.array(
    z.object({
        name: z.string().nullable(),
        tag_name: z.string(),
        html_url: z.url(),
        body: z.string().nullable(),
        published_at: z.string().nullable(),
        author: z.object({ login: z.string() }),
    })
);

export const route: Route = {
    path: '/releases',
    name: 'Releases',
    example: '/my-github/releases',
    categories: ['programming'],
    maintainers: ['Shonee'],
    handler,
};

async function handler() {
    const response = await ofetch('https://api.github.com/repos/DIYgod/RSSHub/releases', {
        timeout: 10000,
    });
    const releases = releasesSchema.parse(response);

    return {
        title: 'RSSHub Releases',
        link: 'https://github.com/DIYgod/RSSHub/releases',
        item: releases.map((release) => ({
            title: release.name || release.tag_name,
            link: release.html_url,
            description: markdown.render(release.body || ''),
            author: release.author.login,
            pubDate: release.published_at ? parseDate(release.published_at) : undefined,
        })),
    };
}
```

`Shonee` 是已核验存在的远程仓库所有者；实际提交路由应填写真正维护者。接口原始字段 `tag_name` 等保持源站格式，本地变量使用 camelCase。示例没有详情请求，因此不需要人为加一层详情缓存；路由结果仍由中间件缓存。未登录 GitHub API 有限频，持续运行时按现有 GitHub 路由的配置方式处理认证。

命名空间加上 `route.path` 得到 `/my-github/releases`，一般不需要手工编辑中央路由表。新增顶层目录后开发服务需要重启；生产环境则重新构建。

### 字段与元数据约束

以 [lib/types.ts](../../lib/types.ts)和 [AGENTS.md](../../AGENTS.md)为准：

- `example` 是以 `/` 开头的 RSSHub 路径，参数名必须与 `path` 一致。
- `name` 不重复命名空间名称，`categories` 只填一个分类。
- `namespace.url`、`radar.source` 不带协议；Radar 直接写在路由元数据，不另建 `radar.ts`。
- Feed 的 `link` 指向人可阅读的来源网页，每条 `item.link` 唯一并指向具体内容。
- `description` 只放正文，作者、日期、分类分别放到专用字段，不人工截断标题。
- 来源提供日期时用 `parseDate`；没有日期就省略，不使用当前时间伪造发布时间。
- 仅使用类型定义支持的条目属性；附件必须是可下载媒体地址，并填写实际 MIME 类型。
- 站点配置走路径参数，输出数量和过滤使用框架通用参数，不增加自定义筛选或翻页接口。

## 4. 控制采集成本并处理失败

### 请求、详情缓存和并发

优先复用 [ofetch](../../lib/utils/ofetch.ts)，它会自动解析 JSON，并支持响应 schema。不要再次 `JSON.parse` 一个已经解析的对象。框架封装没有在此文件里给所有请求直接设置统一 `timeout`，关键请求应显式指定超时。

获取列表后还需要逐篇请求详情时，使用 `cache.tryGet(articleUrl, async () => ...)` 缓存详情，复用 `p-map` 并限制并发。默认详情有效期由 `CACHE_CONTENT_EXPIRE` 决定。不要用不唯一的列表页链接作为缓存键。

需要在抓详情前参考 `limit` 时，复用 [getRouteLimit](../../lib/utils/route-limit.ts)。通用过滤会影响候选池大小，不能直接随意截断列表，否则可能没有足够条目供后续过滤。只获取源站第一批结果。

自托管可以使用现有主机限频配置：

```dotenv
REQUEST_RATE_LIMITS={"www.zhihu.com":{"points":1,"duration":5}}
```

该示例表示每 5 秒允许 1 次请求。主机名精确匹配，API 子域名需要单独配置；限制按进程或 Worker isolate 生效，不是跨副本共享限流。说明见 [CONTRIBUTING.md](../../CONTRIBUTING.md)。

### HTML 和浏览器

HTML 内容只调用一次 `cheerio.load()` 并复用结果。浏览器路由使用项目 [getPlaywrightPage](../../lib/utils/playwright.ts)，在 `onBeforeLoad` 内安装请求资源类型白名单，只放行实际需要的 `document`、`script`、`xhr`、`fetch` 等类型，其余中止。

等待明确选择器或响应，不使用固定延迟。成功和异常路径都在 `finally` 中 `await destroy()`；只有确保自行清理时才设置 `closeTimeout: 0`。不要在详情并发循环中反复创建浏览器会话。路由确实使用项目浏览器工具时才标记 `features.requirePuppeteer: true`。

### 凭据与错误

Cookie、Token 通过运行配置读取。需要增加正式配置时，一并维护 `lib/config.ts` 中的环境变量类型、配置类型和映射，按项目已有方式添加相关配置测试。不要将凭据写入源码或错误信息。

缺少必需配置时使用已有 `ConfigNotFoundError`；源站响应结构变化、状态错误、超时应能被识别。没有条目时检查真实来源，不用自定义空数组消息绕过框架空内容检查。只在来源确实允许为空且符合现有约定时处理正常空列表。

## 5. 验证修改

在关闭缓存的开发实例访问新路由；示例文件加入后可运行：

```bash
curl -fsS 'http://127.0.0.1:1201/my-github/releases?format=json&limit=5'
```

对照源站检查标题、正文、真实日期、唯一链接、图片和附件，并覆盖参数错误、凭据缺失、源站改版或超时。开发实例启用 `ACCESS_KEY` 时还需提供认证。

安装满足版本要求的依赖后，在根目录依次执行：

```bash
pnpm format:check
pnpm typecheck
pnpm build
```

修改公共工具、配置或中间件时，运行已有相关 Vitest，例如缓存改动运行：

```bash
pnpm run vitest lib/utils/cache/index.test.ts
```

具体测试可能依赖 Redis、浏览器或构建产物，应查看对应 setup 和现有 CI。不要在 `lib/routes/` 下新增路由测试文件；真实站点示例由路由测试流程验证。可复用的解析逻辑若已放到公共工具中，再按项目测试习惯补有意义的回归用例。

停止开发服务，再以 `CACHE_TYPE=memory pnpm start` 验证生产构建。最后在实际候选镜像、独立缓存中验证普通路由、浏览器路由及资源释放，不能用开发模式成功替代镜像验收。

## 6. 构建和发布镜像

沿用当前 [Dockerfile](../../Dockerfile)。默认 `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1` 对应外接浏览器的方式；内置浏览器构建使用 `0`。镜像构建会生成程序、移除源码并裁剪依赖，运行中的源码挂载不能代替重新构建。

本机尝试构建，不推送远程：

```bash
docker build --build-arg PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 -t rsshub-local:dev .
```

正式发布前先提交经过验证的改动并确认工作区干净。以下命令需要先登录 GHCR 并拥有 `ghcr.io/shonee/rsshub` 的写权限；这是预期的自有发布位置，本次没有创建包或发布镜像。服务器为 AMD64 时：

```bash
RSSHUB_REV="$(git rev-parse HEAD)"
RSSHUB_IMAGE="ghcr.io/shonee/rsshub:sha-${RSSHUB_REV}"

docker buildx build \
  --platform linux/amd64 \
  --build-arg PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 \
  --label "org.opencontainers.image.revision=${RSSHUB_REV}" \
  --tag "${RSSHUB_IMAGE}" \
  --push \
  .
```

ARM 服务器改为 `linux/arm64`，也可在具备相应构建能力后发布多架构镜像。SHA 标签可能被覆盖，相同源码重建时基础镜像也可能变化；以实际发布的 digest 作为验收、生产部署和回滚依据，保留旧镜像。

参考维护流程建议两条自有流水线：PR 上做环境固定、安装、格式检查、类型检查、构建和相关验证；已验收提交通过标签或手动触发发布镜像并记录 digest。它们目前属于后续建设建议，没有在本次文档任务中新增工作流或配置 GitHub Secrets。

## 7. 同步上游并控制升级范围

先保存文档及业务改动，确保工作区干净；浅仓库先补齐所需历史。在维护分支上执行：

```bash
git fetch origin
git fetch upstream master
git switch custom-main
git pull --ff-only origin custom-main
git switch -c chore/sync-upstream-20261117
git merge upstream/master
```

分支日期按实际升级日期命名。处理冲突后检查 Node/pnpm 版本、锁文件、请求工具、配置类型、浏览器封装和构建行为，再运行第 5 节的检查与真实路由验收。Git 没有冲突不代表行为兼容。

每次核心改动记录原因、受影响路由和升级检查项。生产采用[使用指南的更新与回滚流程](usage.md#6-更新回滚和日常维护)，按同一候选 digest 验收和发布，不在生产机器直接修改编译产物。

## 8. 可选：加载外部私有路由

当前源码已经支持 `USER_ROUTES_PATH`，Node/Docker 可挂载独立 `.mjs` 文件。每个文件定义一个新命名空间，导出 `namespace` 与非空 `routes` 数组：

```javascript
export const namespace = { name: 'Personal', url: 'example.com' };
export const routes = [
    {
        path: '/status',
        name: 'Status',
        maintainers: [],
        handler: async () => ({
            title: 'Personal status',
            link: 'https://example.com',
            item: [{ title: 'Configured', link: 'https://example.com/#status' }],
        }),
    },
];
```

这是只演示模块接口的静态示例，不表示源站有采集 API。保存为挂载目录中的 `personal.mjs`，设置 `USER_ROUTES_PATH` 为该目录的容器内绝对路径，重启后注册 `/personal/status`。文件名只能使用受支持的小写命名空间格式，不能覆盖内置或保留命名空间。

独立模块不支持 TypeScript 和仓库 `@/` 别名；修改后重启，也不自动获得访问隔离，需要正常配置 RSSHub 访问控制。目录中的模块是可信运营者提供的可执行代码。Workers 没有此运行时文件系统扩展方式，应在构建阶段包含路由。实际加载约束见 [lib/user-routes.ts](../../lib/user-routes.ts)。

## 9. 资料与本次验证范围

主要依据：[项目约束](../../AGENTS.md)、[贡献说明](../../CONTRIBUTING.md)、[数据类型](../../lib/types.ts)、[开发注册表](../../lib/registry-dev.ts)、[注册与处理函数](../../lib/registry-helpers.ts)、[生成器](../../scripts/generate-route.mjs)、[构建脚本](../../scripts/workflow/build-routes.ts)及用户提供的自有 RSSHub 维护流程。参考流程中的目录、分支和流水线建议已与当前源码能力区分。

已核验基线的 7,241 个受版本控制文件与远程内容一致，Git 对象连接检查通过。两份文档的本地链接、章节锚点及 24 个命令、代码和配置块已通过静态校验；Compose 配置解析和生成器 `--help` 也已通过。示例接口和生成代码尚未执行真实请求、完整类型检查或镜像验收；这份文档没有代替实际功能发布。官方扩展资料见[开发入门](https://docs.rsshub.app/joinus/)和[基线源码](https://github.com/Shonee/RSSHub/tree/38638e6947212e02b465d2bacb9c60caf36fcb92)。
