# 资源操作

下列 Action 属于 `rdsmysqlv2`。每次执行前按主 skill 和 `volcengine-cli` 完成准备，并查看本次 Action 的帮助。示例中的 `$agent_name` 由当前 Agent 明确设为自己的产品名；`$region` 是用户指定的地域，`$instance_id` 是查询确认的目标，不要原样运行未填写变量的示例。用户选择了 profile 时，为每次请求追加同一个 `--profile`。

## 定位和只读检查

```bash
VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve rdsmysqlv2 DescribeDBInstances --PageNumber 1 --PageSize 100 --region "$region" --output json

VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve rdsmysqlv2 DescribeDBInstanceDetail --InstanceId "$instance_id" --region "$region" --output json

VE_CALLER_TYPE=ai_agent VE_CALLER_NAME="$agent_name" VE_SKILL_NAME=volcengine-db-mysql \
  ve rdsmysqlv2 DescribeDBInstanceEndpoints --InstanceId "$instance_id" --region "$region" --output json
```

不要把首个列表项作为目标。名称有歧义时，用实例 ID、项目、VPC、可用区进一步定位；实例列表为空不等于所有地域都没有实例。

| 检查 | Action 与必要参数 |
| --- | --- |
| 节点和代理 | `DescribeDBInstanceNodes`、`DescribeDBProxy`，`InstanceId` |
| 账号 | `DescribeDBAccounts`，`InstanceId`；支持 `AccountName`、`PageNumber`、`PageSize` |
| 数据库 | `DescribeDatabases`，`InstanceId`；支持 `DBName`、`PageNumber`、`PageSize` |
| 参数 | `DescribeDBInstanceParameters`，`InstanceId`；可指定 `NodeId`、`ParameterName` |
| 参数变更历史 | `DescribeDBInstanceParametersLog`，`InstanceId`、`StartTime`、`EndTime`；时间为 UTC `yyyy-MM-ddTHH:mm:ssZ` |
| 备份和备份策略 | `DescribeBackups`、`DescribeBackupPolicy`；为当前任务显式指定 `InstanceId` |
| 白名单 | `DescribeAllowLists`，指定 `InstanceId`、`QueryInstance=true`、`RegionId`；需要 IP 明细时设 `WithIpList=true` |
| 白名单详情 | `DescribeAllowListDetail`，`AllowListId` |
| 主备切换历史 | `DescribeFailoverLogs`，`InstanceId`；时间为 UTC `yyyy-MM-dd HH:mm:ss`，分页用 `Offset` / `Limit` |
| 操作任务 | `DescribeTasks`，指定 `InstanceId`；`PageNumber`、`PageSize` 必填 |

## 创建和变更

先核实目标及当前状态，再查看 `--help --detail`。以下列出调用依据，不代表已获准执行。

| 需求 | Action | 执行前核实 | 验证 |
| --- | --- | --- | --- |
| 创建实例 | `CreateDBInstance` | 地域、可用区、规格、存储、拓扑、VPC/子网、计费方式 | `DescribeDBInstanceDetail` |
| 创建数据库 | `CreateDatabase` | `InstanceId`、`DBName`、字符集及可选授权 | `DescribeDatabases` |
| 创建账号 | `CreateDBAccount` | `AccountName`、`AccountPassword`、`AccountType`、Host 和指定库权限 | `DescribeDBAccounts` |
| 设置账号权限 | `GrantDBAccountPrivilege` | 原有权限和完整目标授权；支持 `DryRun` | `DescribeDBAccounts` |
| 修改参数 | `ModifyDBInstanceParameters` | 当前值、允许范围、生效方式、应用节点；字段为 `Parameters[].ParameterName` / `ParameterValue` | `DescribeDBInstanceParameters`、变更历史 |
| 绑定白名单 | `AssociateAllowList` | `AllowListIds` 和 `InstanceIds` 都是数组 | `DescribeAllowLists` |
| 修改白名单 | `ModifyAllowList` | `AllowListId`、`AllowListName` 和 `ModifyMode`；检查绑定的全部实例 | `DescribeAllowListDetail` |
| 创建备份 | `CreateBackup` | 存储类型允许的 `BackupMethod`、备份范围和名称 | `DescribeBackups` |
| 重启实例 | `RestartDBInstance` | 节点范围、是否包含代理、执行窗口和业务影响 | 实例详情、节点、任务 |
| 删除实例 | `DeleteDBInstance` | 实例 ID、删除保护、备份保留策略和业务影响 | 实例列表、任务 |

创建实例时：

- 必填项包括 `DBEngineVersion`、`ChargeInfo.ChargeType`、`NodeInfo`、`StorageSpace`、`StorageType`、`SubnetId`、`VpcId`。不要沿用旧 API 的字段。
- 先用 `DescribeAvailabilityZones` 和 `DescribeDBInstanceSpecs` 核实目标地域支持的规格与存储，再按用户选择组装节点。`NodeInfo` 内使用 `NodeType`、`ZoneId`、`NodeSpec`；单/双/多节点的配置不能共用一份固定模板。
- 使用用户选择的 `InstanceType` 和存储类型；单节点不能选择 `LocalSSD`。存储范围和步长按规格及当前帮助核实，不固定成示例值。
- 创建结果先保存返回的实例 ID，然后轮询至 `Running`；`CreateFailed`、`Error` 或超时停止，不重复创建。再创建账号、数据库及其他子资源。

账号、参数和白名单的区别：

- 普通应用账号使用 `AccountType=Normal`，按业务选择 `ReadOnly` 或 `ReadWrite` 等权限；不因为方便就创建 `Super`。密码由安全输入提供，不在文档、回复或日志中落盘。
- `CreateDBAccount --DryRun true` 返回 `CreateAccountSQL`；`GrantDBAccountPrivilege --DryRun true` 返回 `GrantPrivilegeSQL`。预览可能含敏感信息，不原样输出。
- `GrantDBAccountPrivilege` 的帮助将 `AccountPrivileges` 描述为重置授权信息。先读取当前授权，保留仍需使用的权限，不能假定它只追加一个库。
- 参数默认应用范围可能为全部节点。传入 `TemplateId` 时模板配置优先，可能忽略 `Parameters`；先确认所需模式和引擎版本。
- `ModifyAllowList` 默认 `ModifyMode=Cover`，会覆盖已有 IP。仅增加 IP 时显式选 `Append`，保留现有访问范围；不要把排障请求当作开放所有 IP 的授权。

需要完整字段、取值或产品限制时，查看[云数据库 MySQL 文档](https://www.volcengine.com/docs/6313)和[产品规格](https://www.volcengine.com/docs/6313/74505)，再核对当前 CLI 帮助。
