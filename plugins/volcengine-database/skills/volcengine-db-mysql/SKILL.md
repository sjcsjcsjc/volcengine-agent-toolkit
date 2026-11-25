---
name: volcengine-db-mysql
description: Operate Volcengine RDS MySQL with ve. Use for instances, accounts, databases, parameters, backups, slow logs, and SQL via DBW. Do not use for self-hosted MySQL, veDB, Redis, or Supabase.
---

# 火山引擎 RDS MySQL

使用公开的 `ve` CLI 操作云数据库 MySQL 版。复用 `volcengine-cli` 的安装、身份校验、地域、profile、安全策略和错误处理流程；按 skill 名称加载，不依赖跨插件的相对路径。缺少该 skill 时，通过 `volcengine-find-skills` 按名称查找。

## 入口与准备

- 实例、节点、连接地址、代理、账号、数据库、参数、备份和白名单使用 `ve rdsmysqlv2 <Action>`，对应 API `2022-01-01`。旧 `rdsmysql` / `rds_mysql` 的 `2018-01-01` API 已下线，不要套用其 Action 或参数。
- 慢日志、库表元信息和 SQL 使用 `ve dbw <Action>`。RDS 控制面权限不代表工作台或数据库访问权限；先核实目标实例是否可通过工作台访问。
- 先运行 `ve version`，按 `volcengine-cli` 要求检查版本；再查看 `ve rdsmysqlv2 --help` 和本次 Action 的 `--help`。变更前加 `--detail`，以当前帮助为准。本文命令使用 `ve 2.0.2` 的帮助核实。
- 每次执行（包括帮助、重试和轮询）都设置 `VE_CALLER_TYPE=ai_agent`、`VE_CALLER_NAME=<当前 Agent 产品名>`、`VE_SKILL_NAME=volcengine-db-mysql`。
- 实际请求前按 `volcengine-cli` 检查安全策略和 `ve sts GetCallerIdentity`。凭据交给 CLI 解析，不读取配置或登录缓存。只在用户指定地域或 profile 时传 `--region` / `--profile`，后续保持一致。

## 工作流程

1. 明确用户要查询还是变更、目标地域和实例；已有实例先查询，避免重复创建。
2. 用 `DescribeDBInstances` 定位实例 ID，再查 `DescribeDBInstanceDetail`、`DescribeDBInstanceNodes`、`DescribeDBInstanceEndpoints`。区分本地盘/云盘、单/双/多节点、是否有代理和终端的用途，不假定所有实例有相同拓扑或公网地址。
3. 按任务加载下表中的参考，只调用本次需要的 API。请求参数大小写以帮助为准；数组使用帮助支持的编号字段或 JSON body，不同时传 body 和业务参数。
4. 保留完整 JSON 检查 `ResponseMetadata.Error`；依据实际 `Result` 字段展示结果，不猜测返回路径。列表按接口的分页字段读取，遇到空列表、缺失字段或失败先处理再继续。
5. 变更遵循用户已有授权及 CLI 安全策略。完成后以对应只读 API 验证；异步操作设置期限并轮询，失败时停止。未经用户请求，不因诊断结果自动重启、切换主备、恢复数据或修改参数。

| 任务 | 读取 |
| --- | --- |
| 实例、连接、账号、数据库、参数、备份和白名单 | [资源操作](references/operations.md) |
| 慢查询、库表结构、SQL 和连接排查 | [查询与诊断](references/queries.md) |

## 关键区别

- `rdsmysqlv2` 多数接口用 `InstanceId`；`dbw ExecuteSQL`、`ListDatabases`、`ListTables`、`GetTableInfo` 用 `InstanceID`。`dbw DescribeSlowLogs` 又用 `InstanceId`，不能统一替换大小写。
- 慢日志的 `StartTime` / `EndTime` 是秒级 Unix 时间戳，且 `RegionId` 是必填业务参数；系统选项 `--region` 不能替代它。参数变更历史使用 UTC 时间字符串。
- 创建账号和账号授权的 `DryRun` 返回预览 SQL；它不是通用接口的 `DryRunOperation` 错误模式。不要将预览当作变更成功。
- `ExecuteSQL` 的影响由 SQL 内容决定；只读任务只执行已检查的查询。DDL、DML、权限变更及有副作用的语句需要用户授权，并遵循目标工作台的变更流程和 CLI 策略。

未覆盖的 API 先查看服务帮助和[官方文档](https://www.volcengine.com/docs/6313)，不能把 Supabase 的工作区、分支、SQL 或 Auth 命令套到 MySQL 上。遇到工作台不支持、权限缺失或网络不通，报告具体阻塞；不要自动开公网、放宽白名单或绕过审批。
