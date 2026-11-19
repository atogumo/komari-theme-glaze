# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

Komari 探针的前端主题 Komari Glaze，Vue 3 + Vite 单页应用，交付物是 Komari 后台能导入的主题压缩包。

- 基线是 Komari server 1.4.3 和 agent 1.5.11，接口字段的含义以这两个版本的源码为准。
- 当前阶段只做问题修复和优化，不新增大特性。
- 变更用 OpenSpec 管理（`openspec/`），相关文档用中文写。

## 命令

```bash
npm ci               # 按锁文件安装
npm run dev          # Vite 开发服务器
npm run build        # 类型检查 + 打包 + 生成主题压缩包
npm run type-check   # 只做类型检查
```

没有测试，也没有 lint。验证方式是 `npm run build` 通过，并在浏览器里对照实际面板核对。

## 提交与推送

- 没有用户的明确授权，不允许提交或推送。
- 提交和推送必须拆成两个步骤，不能合在一次操作里完成。
- 提交信息统一用下面的格式，具体改动以 Markdown 列表逐条写出：

  ```
  <类型>(<范围>): <一句话概述>

  - 具体改动一
  - 具体改动二
  ```

  - 类型取 `feat`、`fix`、`perf`、`refactor`、`docs`、`build`、`ci`、`chore` 之一。
  - 范围必填，写受影响的模块，用小写英文，例如 `ping`、`chart`、`home`、`detail`、`globe`、`store`、`rpc`、`settings`、`release`；涉及整个仓库时写 `repo`。
  - 概述和列表用中文，概述末尾不加句号。
- 提交信息里不允许携带协作者信息，不加 `Co-Authored-By` 之类的署名行，也不加工具生成的标记。

## 版本与发布

- 版本号只有一个来源：`komari-theme.json` 的 `version`。
- 按语义化版本递增：

  | 变化 | 版本位 |
  | --- | --- |
  | 修复错误，不改变用法 | 补丁位 |
  | 行为有变化，或新增设置项 | 次版本 |
  | 不兼容：设置项改名、`short` 变更、基线版本变更 | 主版本 |

- 每个版本在 `CHANGELOG.md` 顶部加一条记录。
- 推送不会发版。小改动可以先累积在 `main` 上，测试稳定后再改版本号、统一发一个版本。
- 发版由用户在 GitHub 的 Actions 页面手动运行 `.github/workflows/release.yml`：它构建主题，为当前的 `version` 打 `v<version>` 标签并发布 Release。如果该标签已经存在于别的提交上，流程会跳过，所以发版前必须先改版本号。

## 改动时要注意的约定

- `komari-theme.json` 的 `short`（`Glaze`）是主题的唯一标识，改了等于换一个主题，用户设置会丢失。
- 新增或修改主题设置项，要同时改 `komari-theme.json` 的 `configuration.data` 和 `src/stores/app.ts` 里的解析。
- `stores/nodes.ts` 对节点状态是逐字段就地赋值的，不要改成整体替换对象，否则所有卡片每轮轮询都会重渲染。
- `public/images/` 下的文件名由代码在运行时拼出，改名要同步修改 `utils/regionHelper.ts` 和 `utils/osImageHelper.ts`。
- 压缩包文件名 `komari-theme-glaze-build-<短提交号>.zip` 在 `vite.config.ts` 和发布流程里各写了一次，要一起改。

## 对外请求

除了 Komari 自身的接口，页面还会从访客的浏览器直接请求第三方服务。新增这类请求，或改动下面这些代码时，要考虑隐私影响：

- IP 归属地：`utils/ipGeoHelper.ts`，把节点 IP 发给四个公共查询服务（`api.ip.sb`、`ipinfo.io`、`ipwho.is`、`ipapi.co`）。
- 访客信息卡片：`components/VisitorInfo.vue`，请求 `ipapi.co`，对方能看到访客 IP。
- 汇率：`utils/financeHelper.ts`，请求 `open.er-api.com` 和 `api.frankfurter.app`。
- 图标：`@iconify/vue` 在运行时从 Iconify 公共接口获取，没有打包进产物。

## 服务端字段的易错点（1.4.3）

以下含义从主题代码看不出来，已对照服务端源码确认：

- `connections`：实时状态里是 TCP + UDP 总数，历史记录里是纯 TCP；`connections_udp` 两处都是 UDP。
- 历史记录的 `ram_total`、`swap_total`、`disk_total` 恒为 0，总量取节点基本信息。
- 实时状态的 `gpu`、`temp` 写死为 0。
- `expired_at` 可以是 `null`，`public_remark` 为空时不返回。
- 离线节点仍返回最后一次上报的状态，`online` 为 `false`。
- Ping 记录每个节点、每个任务每分钟一个点，丢包为负值。
- `common:getRecords` 的点数上限参数是 `maxCount`，默认 4000，超过后抽样；取负载记录时 `records` 是按节点 ID 分组的对象。
- 实时状态带有每个 Ping 任务最近 1 小时的汇总（`ping` 字段），其中 `latest` 是最近一次成功的延迟，全丢包时为 -1。
