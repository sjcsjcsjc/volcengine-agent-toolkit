# 查询与诊断

使用 `dbw` 的公开接口。调用前确认目标是 RDS MySQL、地域和数据库，工作台具备访问该实例的权限。示例中的 `$agent_name` 由当前 Agent 明确设置；`$region` 是用户指定地域，实例、库表名称从实际查询确认。用户选了 profile 时，所有请求追加同一 `--profile`。

## 慢日志

从用户的故障时间窗口取开始和结束时间，转换为秒级 Unix 时间戳；不能传毫秒或 ISO 字符串。`RegionId` 是业务参数，须与请求地域一致。RDS MySQL 的 `InstanceType` 取 `MySQL`。

```bash
VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve dbw DescribeSlowLogs \
    --InstanceId "$instance_id" --InstanceType MySQL --RegionId "$region" \
    --StartTime "$start_seconds" --EndTime "$end_seconds" \
    --OrderBy QueryTime --SortBy DESC --PageNumber 1 --PageSize 100 \
    --region "$region" --output json
```

- 指定节点时先查 RDS `DescribeDBInstanceNodes`，再传实际 `NodeId`；不能把代理地址或实例 ID 当作节点 ID。
- 按需使用 `SearchParam.DBs`、`Users`、`SourceIPs`、耗时范围或 `SQLTemplateID` 过滤。分页范围、可用排序字段以帮助为准。
- 需要按 SQL 模板聚合时用 `DescribeAggregateSlowLogs`，先查看其帮助；不要把明细接口的排序字段直接复制到聚合接口。
- 根据实际日志的耗时、锁等待、扫描行数和返回行数判断线索，并核对调用频率、执行计划、索引和时间窗口。慢日志为空只说明本次筛选未返回记录，不能据此判定实例无性能问题。

## 库表与 SQL

下面的接口用 `InstanceID`，与慢日志的 `InstanceId` 不同。不要手动拼连接串或寻找数据库密码来代替工作台请求。

| 任务 | Action | 必填参数 |
| --- | --- | --- |
| 数据库列表 | `ListDatabases` | `InstanceID`、`InstanceType` |
| 表列表 | `ListTables` | `InstanceID`、`InstanceType`、`Database` |
| 表元信息 | `GetTableInfo` | `InstanceID`、`InstanceType`、`Database`、`Table` |
| 执行 SQL | `ExecuteSQL` | `InstanceID`、`InstanceType`、`Database`、`Commands` |

```bash
VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve dbw ListTables --InstanceID "$instance_id" --InstanceType MySQL \
    --Database "$database_name" --PageNumber 1 --PageSize 100 --region "$region" --output json

VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve dbw GetTableInfo --InstanceID "$instance_id" --InstanceType MySQL \
    --Database "$database_name" --Table "$table_name" --region "$region" --output json

VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve dbw ExecuteSQL --InstanceID "$instance_id" --InstanceType MySQL \
    --Database "$database_name" --Commands 'SELECT VERSION();' \
    --TimeOutSeconds 30 --region "$region" --output json
```

查看 SQL 内容并按其实际作用判断授权范围，不能把 `ExecuteSQL` 视作天然只读。只读排查优先元信息、有限结果的 SELECT、SHOW 和普通 EXPLAIN；`EXPLAIN ANALYZE` 会执行查询，不作为默认执行计划检查。大表聚合也可能消耗大量资源，先限定条件和时间窗口。

DDL、DML 和权限变更先给出具体 SQL、目标及影响，遵循用户授权、工作台现有变更流程及 CLI 策略；不得以直接执行 SQL 绕过已要求的审批。不要自动执行慢查询建议中的建索引或参数修改。

## 连接和故障排查

1. 查 RDS 实例详情、节点、终端和代理，确认目标终端用途、地址及端口。不要把只读终端当作写入终端，也不要假设单节点实例有备节点。
2. 查绑定白名单、账号 Host 和库权限，核对客户端所在网络。控制面 API 成功不能证明数据库连通；没有客户端网络观测时，说明这个证据缺口。
3. 参数变更问题查 `DescribeDBInstanceParametersLog`；主备切换问题查 `DescribeFailoverLogs`；操作仍在进行时查 `DescribeTasks`。三者时间格式、分页字段各不相同，不能共用慢日志请求。
4. 事务、锁和会话问题，只有在工作台 SQL 已可用且授权允许时查询所需信息；权限不足时报告缺少的访问能力，不自动提权或终止会话。
5. 工作台不可用时，保留已取得的 RDS 控制面证据，并说明 SQL/库表或日志查询的具体失败。不要编造 `rdsmysqlv2 ExecuteSQL`、慢日志或错误日志 Action，也不要自动启用审计等付费功能。

报告给出目标、时间窗口、证据、判断及仍需核实的项。避免将日志中的 SQL 字面值和业务数据完整复制到对话。
