# RSSHub 使用与运维指南

RSSHub 把网站 API、HTML 页面或浏览器采集结果转换为 RSS、Atom 和 JSON Feed。将路由地址添加到 RSS 阅读器后，由阅读器定期请求订阅源；RSSHub 按请求生成内容并缓存结果，不会自动保存完整历史文章。

本文面向自托管实例的使用者和维护者。源码基线为 `38638e6947212e02b465d2bacb9c60caf36fcb92`，核查日期为 2026-11-17。新增站点和修改程序见[二次开发指南](development.md)，微博、Twitter/X、抖音、小红书、快手和 Telegram 的采集方式与监控边界见[六平台采集链路](social-platforms.md)。

文中的运行、部署命令供实际操作使用；本次已核对源码、文档链接与示例语法，并通过 Compose 配置静态校验，未安装依赖、启动服务或执行镜像部署。

## 1. 选择运行方式

| 方式 | 适用场景 | 更新方式 |
| --- | --- | --- |
| 源码开发模式 | 调试和修改站点路由 | 修改源码，由 `tsx watch` 重启 |
| 源码生产模式 | 在有 Node 环境的机器上运行 | 重新构建，再重启进程 |
| 自建镜像 + Redis + Browserless | 持续维护自定义站点 | 验收镜像后按 digest 更新 |

普通 API、HTML 路由不需要浏览器。需要执行 JavaScript 或读取动态页面的路由才需要本机 Chromium、远程 Browserless 或相应的 Worker 浏览器绑定。Redis 是生产缓存的建议选择，默认的内存缓存也能运行，但多个进程之间不共享。

仓库根目录的 [docker-compose.yml](../../docker-compose.yml) 使用官方 `diygod/rsshub` 镜像。直接启动它不会包含本地 TypeScript 改动；自定义程序需要构建自己的镜像。

## 2. 从源码启动

### 准备环境

[package.json](../../package.json) 声明 Node.js `^24.15.0 || ^26.13.0`、pnpm `12.18.0`。先安装满足范围的 Node，再准备对应 pnpm：

```bash
node --version
npm install --global pnpm@12.18.0
pnpm --version
```

本次核查机器上的 Node 为 `26.7.0`，低于当前源码要求的 Node 26 最低版本。不能因为主版本同为 26 就认定兼容。升级源码后也应重新检查版本声明。

在仓库根目录安装依赖。API、HTML 路由或使用远程 Browserless 时，可以跳过本机浏览器下载：

```bash
PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 pnpm install --frozen-lockfile
PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1 pnpm rebuild
```

开发环境需要开发依赖，不加 `--prod`。保留锁文件，依赖构建脚本的允许项以 [pnpm-workspace.yaml](../../pnpm-workspace.yaml) 为准。

### 开发模式

在根目录创建 `.env`；已有文件时只调整对应配置，保留站点凭据：

```dotenv
PORT=1201
LISTEN_INADDR_ANY=false
CACHE_TYPE=
LOGGER_LEVEL=debug
NO_LOGFILES=true
```

`CACHE_TYPE=` 显式关闭缓存，便于看到解析逻辑的最新结果。`pnpm dev` 本身不会关闭缓存。不设置该变量时，默认仍为内存缓存。

```bash
pnpm dev
```

另开终端检查服务，预期得到 `ok`：

```bash
curl -fsS http://127.0.0.1:1201/healthz
```

开发脚本启用了 Node inspector，仅用于开发环境。生产运行使用构建产物。

### 生产模式

停止开发进程后，在根目录运行：

```bash
pnpm build
CACHE_TYPE=memory pnpm start
```

`build` 生成路由注册表并编译，`start` 执行 `dist/index.mjs`。修改 TypeScript 后必须重新构建。根目录 `.env` 仍会参与配置读取；上面命令显式覆盖其中的空缓存设置。

## 3. 订阅和查看内容

路由地址由实例地址、站点命名空间和路由参数组成。例如已有 GitHub Issues 路由：

```text
http://127.0.0.1:1201/github/issue/DIYgod/RSSHub/open
```

这是源码中已有的路由示例，实际抓取是否成功还取决于 GitHub 可达性、限频和站点配置。将完整 URL 添加到 RSS 阅读器即可订阅。

调试时可直接检查 JSON Feed：

```bash
curl -fsS 'http://127.0.0.1:1201/github/issue/DIYgod/RSSHub/open?format=json&limit=5'
```

| 通用参数 | 示例 | 用途 |
| --- | --- | --- |
| `format` | `format=atom`、`format=json` | 选择输出格式，默认 RSS |
| `limit` | `limit=5` | 限制输出条数；不保证每条路由都只抓取这么多源站数据 |
| `filter_title` | `filter_title=RSSHub` | 保留标题匹配的条目 |
| `filterout_title` | `filterout_title=广告` | 排除标题匹配的条目 |
| `fulltext` | `fulltext=true` | 尝试提取全文，可能增加请求和耗时 |

站点自己的用户 ID、栏目等配置使用各路由定义的路径参数。更多参数见[官方参数说明](https://docs.rsshub.app/guide/parameters)。

服务还提供 `/api/reference` 和 `/api/openapi.json`，用于查看命名空间、路由等管理 API。它们不是订阅地址。启用访问控制时，这些地址同样需要认证。

## 4. 配置缓存、浏览器和访问控制

默认值依据 [lib/config.ts](../../lib/config.ts)，不同提交可能变化。

| 环境变量 | 当前默认值 | 作用 |
| --- | --- | --- |
| `PORT` | `1200` | 应用监听端口 |
| `LISTEN_INADDR_ANY` | `true` | 是否监听所有地址；本机源码调试建议 `false` |
| `CACHE_TYPE` | `memory` | `memory`、`redis`、`http`；空字符串关闭缓存 |
| `REDIS_URL` | `redis://localhost:6379/` | Redis 连接地址，容器中使用服务名 |
| `CACHE_EXPIRE` | `300` 秒 | 路由结果缓存时间 |
| `CACHE_CONTENT_EXPIRE` | `3600` 秒 | 详情等内容缓存的默认时间 |
| `CACHE_REQUEST_TIMEOUT` | `60` 秒 | 路由请求协调控制键的有效期，不是源站 HTTP 超时 |
| `REQUEST_TIMEOUT` | `30000` 毫秒 | 请求工具使用的超时配置；具体路由可能另设超时 |
| `REQUEST_RETRY` | `2` | 请求工具使用的重试配置 |
| `PLAYWRIGHT_CDP_ENDPOINT` | 未设置 | Chromium CDP 远程连接地址 |
| `ACCESS_KEY` | 未设置 | 访问密钥，未设置时不要求此项认证 |
| `NO_LOGFILES` | `false` | 设置为 `true` 后由容器或进程平台收集日志 |
| `DEBUG_INFO` | `true` | 调试信息控制；生产可设为 `false` |
| `USER_ROUTES_PATH` | 未设置 | 外部私有 `.mjs` 路由目录，详见二次开发指南 |

本机浏览器需要单独安装：

```bash
pnpm exec patchright install chromium
```

Linux 还需要满足 Chromium 的系统依赖。远程 Browserless 则填写完整端点，例如宿主机访问已发布到本机的 Browserless 使用 `ws://127.0.0.1:3000?token=实际令牌`，同一 Compose 网络内使用 `ws://browserless:3000?token=实际令牌`。根目录 `.env` 不应假定支持变量之间的自动展开。

当前浏览器连接优先级为 CDP、Playwright WS、本机 Chromium，依据 [playwright.ts](../../lib/utils/playwright.ts)。Browserless 的会话时限与 RSSHub 中的页面导航、选择器等待超时需要分别配置。

启用 `ACCESS_KEY` 后，可通过 `?key=密钥` 访问。查询参数 `code` 是 `MD5(请求路径 + 密钥)`，不是直接填写密钥。`/healthz` 也受保护，因此健康检查必须携带有效凭据。实现见[访问控制中间件](../../lib/middleware/access-control.ts)和[凭据校验](../../lib/utils/weibo-oauth.ts)。

`.env` 已被 Git 忽略。其他自定义文件名如 `.env.production` 应另行检查忽略规则。订阅 URL 中的凭据也可能进入阅读器和代理日志，应按凭据管理。

## 5. 使用自己的镜像部署

以下是根据仓库 Compose 和参考维护流程整理的部署起点，尚未在本机运行验收。先按[二次开发指南的镜像发布步骤](development.md#6-构建和发布镜像)准备可拉取的镜像，再在独立部署目录保存下面的 `compose.yaml` 与 `.env`。

```yaml
services:
  rsshub:
    image: "${RSSHUB_IMAGE:?Set RSSHUB_IMAGE}"
    restart: unless-stopped
    env_file: .env
    environment:
      NODE_ENV: production
      PORT: "1200"
      CACHE_TYPE: redis
      REDIS_URL: redis://redis:6379/0
      PLAYWRIGHT_CDP_ENDPOINT: "ws://browserless:3000?token=${BROWSERLESS_TOKEN:?Set BROWSERLESS_TOKEN}"
      ACCESS_KEY: "${ACCESS_KEY:?Set ACCESS_KEY}"
      DEBUG_INFO: "false"
      NO_LOGFILES: "true"
    ports:
      - "127.0.0.1:${RSSHUB_PORT:-1200}:1200"
    depends_on:
      redis:
        condition: service_healthy
      browserless:
        condition: service_healthy
    healthcheck:
      test: [CMD-SHELL, 'curl -fsS --get --data-urlencode "key=$$ACCESS_KEY" http://127.0.0.1:1200/healthz > /dev/null']
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    logging:
      driver: local
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: "${REDIS_IMAGE:?Set REDIS_IMAGE}"
    restart: unless-stopped
    volumes:
      - redis-data:/data
    healthcheck:
      test: [CMD, redis-cli, ping]
      interval: 30s
      timeout: 5s
      retries: 3

  browserless:
    image: "${BROWSERLESS_IMAGE:?Set BROWSERLESS_IMAGE}"
    restart: unless-stopped
    environment:
      TOKEN: "${BROWSERLESS_TOKEN:?Set BROWSERLESS_TOKEN}"
    shm_size: 1g
    healthcheck:
      test: [CMD-SHELL, 'curl -fsS --get --data-urlencode "token=$$TOKEN" http://127.0.0.1:3000/pressure > /dev/null']
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

volumes:
  redis-data:
```

配套 `.env` 的三个镜像引用必须先填写，使用已经验收的固定版本或 digest。访问密钥与 Browserless Token 分别用 `openssl rand -hex 24` 生成，不共用：

```dotenv
RSSHUB_IMAGE=
REDIS_IMAGE=
BROWSERLESS_IMAGE=
ACCESS_KEY=
BROWSERLESS_TOKEN=
RSSHUB_PORT=1200
```

RSSHub 镜像来源应为自己的发布仓库，例如 `ghcr.io/shonee/rsshub`；本文未发布该镜像，也未核验这个包是否已存在。Redis、Browserless 的具体版本由部署者选定并验证，不提供虚构 digest。使用的 Browserless 镜像应兼容上述 `/pressure` 健康检查并包含 `curl`，其余并发参数按该镜像版本配置。

Compose 中 `${...}` 在宿主机展开，`$$...` 留给容器内 shell。`env_file` 也会将站点 Cookie、Token 等配置注入 RSSHub，`environment` 显式配置优先。这个模板没有将 Redis 和 Browserless 端口发布到宿主机；RSSHub 仅绑定回环地址，跨设备访问应配置 HTTPS 反向代理。不要将源码开发 `.env` 中的 `LISTEN_INADDR_ANY=false` 复制到此容器配置，否则端口转发无法访问容器内的回环监听。

在部署目录运行；私有镜像需要先登录对应镜像仓库：

```bash
docker compose config -q
docker compose pull
docker compose up -d --wait
docker compose ps
docker compose logs --tail=100 rsshub
```

`config -q` 只校验，不输出包含凭据的完整配置。验收至少包括一条 API/HTML 路由、一条实际使用的浏览器路由和一次缓存命中。`/healthz` 正常只说明应用可以响应，不能证明采集成功。

## 6. 更新、回滚和日常维护

更新前保存旧镜像 digest 和对应配置，在独立验收环境运行候选镜像。验收环境使用独立 Compose 项目名、宿主机端口和 Redis 数据卷，避免旧缓存掩盖抓取失败。

将部署 `.env` 的 `RSSHUB_IMAGE` 改成验收通过的引用，再执行：

```bash
docker compose pull rsshub
docker compose up -d --no-deps --wait rsshub
```

需要回滚时，恢复旧镜像 digest 和匹配的配置，执行相同命令并复查真实路由。`restart` 不会应用新镜像和环境变量；单实例重建可能短暂停机。日常路由更新只替换 RSSHub，Redis、Browserless 的版本另行验收。保留数据卷，不使用 `down -v` 作为普通更新步骤。

分支初始化状态已记入[维护记录](maintenance.md)。实际发布时继续追加以下信息，只有完成镜像发布和验收后才填写相应结果：

| 记录项 | 内容 |
| --- | --- |
| 源码与上游 | 自有 commit、对应上游基线 |
| 镜像 | RSSHub digest、Redis 与 Browserless 引用 |
| 配置 | 配置版本及变更项，不记录真实密钥 |
| 验收 | 关键路由、冷缓存结果、浏览器释放情况 |
| 回滚 | 上一个可用镜像与配置位置 |

| 现象 | 检查与处理 |
| --- | --- |
| 改了代码但结果没变化 | 开发环境关闭缓存；生产环境重新构建并确认镜像已更换 |
| 健康检查认证失败 | 检查 `ACCESS_KEY` 和健康检查的 `key` 参数 |
| 浏览器启动失败 | 检查浏览器安装、CDP 端点、Token、容器服务名和系统依赖 |
| `/healthz` 正常但路由报错 | 检查源站可达性、站点凭据、限频、页面结构和应用日志 |
| 多实例仍触发源站限频 | 内置出站限频按进程或 Worker isolate 生效，集群总量需要共享限流 |
| 新增路由返回 404 | 开发时重启；生产时重新构建自己的镜像，确认未使用官方旧镜像 |

## 7. 依据与验证边界

本文吸收了参考维护流程中的分支、三服务部署、固定镜像和回滚建议；实际行为以当前 [package.json](../../package.json)、[配置](../../lib/config.ts)、[路由注册](../../lib/registry.ts)、[输出模板](../../lib/middleware/template.tsx)、[Dockerfile](../../Dockerfile)和[原有 Compose](../../docker-compose.yml)为依据。

官方资料入口：[使用文档](https://docs.rsshub.app/)、[部署文档](https://docs.rsshub.app/deploy/)、[当前源码基线](https://github.com/Shonee/RSSHub/tree/38638e6947212e02b465d2bacb9c60caf36fcb92)。运行步骤尚需在满足 Node 版本的环境中验证；上线应以同一镜像 digest 的真实路由验收结果为准。
