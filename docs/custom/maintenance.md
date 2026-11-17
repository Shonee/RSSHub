# RSSHub 自有分支维护记录

长期维护分支为 `custom-main`，跟踪 `origin/custom-main`。`master` 跟踪 `origin/master`，用于保留 fork 的同步基线。新增站点和修复从 `custom-main` 建立功能分支，PR 的目标分支也选择 `custom-main`。

开发步骤见[二次开发指南](development.md)，实例部署与回滚见[使用指南](usage.md)。

## 2026-11-17：初始化维护分支

| 项目 | 已确认状态 |
| --- | --- |
| 自有仓库 | [Shonee/RSSHub](https://github.com/Shonee/RSSHub) |
| 上游仓库 | [DIYgod/RSSHub](https://github.com/DIYgod/RSSHub) |
| 初始提交 | `38638e6947212e02b465d2bacb9c60caf36fcb92` |
| 本地分支 | 已创建并切换到 `custom-main` |
| 远程分支 | 已创建 [custom-main](https://github.com/Shonee/RSSHub/tree/custom-main) |
| 跟踪与推送 | 跟踪 `origin/custom-main`，推送远程显式设置为 `origin` |
| GitHub 默认分支 | `master`，保持不变 |
| 历史范围 | 浅仓库，尚未补齐历史或合并新的上游提交 |
| 源码 | 原有 7,241 个受版本控制文件保持不变 |
| 文档 | 使用指南、二次开发指南和本记录纳入 `custom-main` 的文档初始化提交 |
| 发布与部署 | 尚未构建或发布自有镜像，尚未部署服务 |

Git HTTPS 查询在本次操作中超时，因此通过已登录的 GitHub CLI 调用 GitHub API 创建远程分支，再配置本地跟踪关系。远程分支创建时指向上述真实提交，没有改写提交历史。

## 维护约定

- 新增路由优先放在 `lib/routes/<site>/`；确有复用需求时再拆分辅助模块。
- 每个功能使用独立分支；合并前检查真实路由、格式、类型和生产构建。公共工具改动补充适当回归验证，不在 `lib/routes/` 新增路由测试。
- 上游更新通过独立 `chore/sync-*` 分支合并、验收，再进入 `custom-main`。浅仓库需要先补齐合并所需历史。
- 现有 `sync-fork.yml` 操作默认分支，默认分支维持 `master`；若将来改成 `custom-main`，先调整同步策略。
- 现有镜像发布工作流围绕 `master`，并要求 Docker Hub 配置。`custom-main` 目前使用二次开发指南中的手动构建流程；推送维护分支不表示已发布自有镜像。
- 发布时记录源码提交、上游基线、镜像 digest、配置版本、真实路由验收结果和回滚目标；不记录密钥。

本次仅初始化分支及维护文档，没有启用定时维护、修改 GitHub 工作流或执行生产升级。开始开发前需将本机 Node `26.7.0` 切换到项目要求的 `^24.15.0 || ^26.13.0`，并安装 `pnpm@12.18.0`。
