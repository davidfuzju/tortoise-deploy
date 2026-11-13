# tortoise-deploy（davidfuzju fork）

[mserajnik/tortoise-deploy](https://github.com/mserajnik/tortoise-deploy) 的 fork，只用来打我们自己的镜像：`ghcr.io/davidfuzju/tortoise-server:zhcn` / `ghcr.io/davidfuzju/tortoise-database:zhcn`。仓 `github.com/davidfuzju/tortoise-deploy`（public），本地嵌套 clone 在 `~/Workspaces/tokendivers/projects/tortoise-deploy/`。部署（compose、配置、Windows 启动器）在 `davidfuzju/tortoise`，不在这里。

## 相对上游的改动

| 位置 | 改动 |
| --- | --- |
| `.github/deploy.yaml` | 只留一个变体 `zhcn`：TortoiseBots + tw-mod-autoscale + tw-mod-zhcn + mod-azeroth-voices + mod-twow-invasions；去掉 base / modules 变体、tw-mod-leech、徽章 |
| `docker/patches/zhcn-*.patch` | BluetomatoBo/tortoise-wow 的三处中文相关核心修复（脚本 NPC 问候、宠物名重载、商城中文名），构建时打到核心上 |
| `docker/patches/modules/<模块目录>/*.patch` | 模块克隆后打的补丁。现有：`mod-azeroth-voices/npc-chatter-getname-concat.patch`（其 b8e7fb5 在本核心上编不过：`GetName()` 返回 `const char*`，不能与字面量直接 `+`） |
| `.github/workflows/notify-build-result.yaml` | 构建失败开 `build-failure` issue，成功后关闭（每日构建无人值守） |
| `docker/server/Dockerfile` | 克隆模块后打上面的模块补丁；模块自己的 CMake 不安装 SQL 时（mod-twow-invasions），构建时把 `data/sql/{character,world}` 拷到安装目录 |

改动处都带 `davidfuzju:` 注释，合并上游时按它找。

## 规则

- `mod-azeroth-voices` 是该模块代码与安装路径要求的目录名（仓库名是 `mod-twow-azeroth-voices`），`modules` 变量里不能改。
- `module-configs` 必须与构建装出的配置文件逐字一致、按 C locale 排序；加减模块要同步改，否则构建失败。部署仓必须挂载这里列出的每个配置文件。
- 新加的模块不登记到 `migration-edits`：改了已发布的迁移只会被再执行一次，不会让数据库停住。
- 模块补丁打不上（模块上游已改到那几行）构建直接失败：看上游是否已修好，修好就删掉补丁，没修就基于新代码重做。
- `zhcn-*.patch` 打不上会让构建失败（`TORTOISE_FAIL_ON_PATCH_ERROR=1`）：到 BluetomatoBo 分支看对应提交是否已更新，基于核心新代码重做 patch。

## 构建与同步上游

- 构建：推到 `master` 且改动在 `.github/deploy.yaml` / `docker/**` 时自动跑；每天 00:00 UTC 定时检查各来源有无新提交，有就重建；也可在 Actions 手动触发 `Build and push Docker images`（勾 `force-rebuild` 强制重建）。
- 构建失败：`notify-build-result.yaml` 开一个 `build-failure` issue（已有就追加评论），下次成功自动关闭。查原因只看错误行，完整日志 70 KB 以上、大半是回显的 `RUN` 命令：`gh run view <id> --log-failed | grep -E 'error:|Error [0-9]|has to install|requires the module|Failed to apply'`
- 合并上游：`git fetch upstream && git merge upstream/master`，冲突基本只在 `.github/deploy.yaml`，按上表保留我们的改动。
