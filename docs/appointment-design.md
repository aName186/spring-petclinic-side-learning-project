### 预约模块设计文档

#### 文档目的与范围

本文档为第 1 周第 8 步产出，用于指导第 2 周预约主链路开发。系统定位为诊所内部三角色使用（ADMIN / VET / RECEPTIONIST），采用“业务状态 + 转换限制”方式实现，不引入状态机框架。

#### 预约表数据模型

1. `id` `int` （主键，自增）
2. `pet_id` `int`（外键，pets表，标识宠物）
3. `vet_id` `int`（外键，vets表，标识医生）
4. `start_time` `DATETIME`（入院时间）
5. `end_time` `DATETIME`（出院时间，`end_time > start_time`，采用左闭右开模式）
6. `status` `VARCHAR(20) NOT NULL`（预约状态，取值： CREATED / CONFIRMED / COMPLETED / CANCELLED / TIMEOUT）
7. `created_at` `DATETIME`（预约创建时间）
8. `updated_at` `DATETIME`（预约修改时间）
9. `created_by` `int`（外键，暂时为空，引用目标待第2周登录/权限实现时确定，标识创建者/系统操作者）
10. `updated_by` `int`（标识最后修改者/操作者，语义同 created_by）
11. `idx_appointment_vet_time (vet_id, start_time, end_time)`（普通索引，用于区间查询/区间锁）

#### 外键关联关系

- `pet_id` -> `pets.id`，标识本次预约对应的宠物
- `vet_id` -> `vets.id`，标识本次预约对应的医生
- `created_by` / `updated_by` -> 引用目标待第 2 周登录/权限实现时确定（当前项目无独立用户表），表示系统操作者，不是宠物主人 / owners

#### 状态定义

| 状态 | 含义 |
| --- | --- |
| CREATED | 预约已创建，等待医生确认 |
| CONFIRMED | 医生已确认预约 |
| COMPLETED | 预约已完成 |
| CANCELLED | 预约已取消 |
| TIMEOUT | 已确认但超过预约时间仍未完成，由后台定时任务自动置为超时 |

#### 状态转换模型

| 当前状态   | 动作     | 目标状态  | 允许角色                 | 前置条件                                |
| ---------- | -------- | --------- | ------------------------ | --------------------------------------- |
| 无（初始） | 创建预约 | CREATED   | ADMIN、RECEPTIONIST      | start/end 与已有预约不重叠，end > start |
| CREATED    | 确认     | CONFIRMED | VET                      | 预约时间未过，且未被取消/完成           |
| CREATED    | 取消     | CANCELLED | ADMIN、RECEPTIONIST      | 预约时间未过                            |
| CONFIRMED  | 完成     | COMPLETED | VET                      | 该预约已确认且预约时间已结束            |
| CONFIRMED  | 取消     | CANCELLED | ADMIN、RECEPTIONIST、VET | 预约时间未过                            |
| CREATED    | 改时间   | CREATED   | ADMIN、RECEPTIONIST      | 新时段不被占用                          |
| CONFIRMED  | 改时间   | CREATED   | ADMIN、RECEPTIONIST      | 新时段不被占用                          |
| COMPLETED  | —        | —         | —                        | 终态，不可再流转                        |
| CANCELLED  | —        | —         | —                        | 终态，不可再流转                        |
| TIMEOUT    | —        | —         | —                        | 终态，不可再流转                        |

#### 后台定时任务

- 触发：`@Scheduled` 定时任务（如每分钟）
- 查询：`status = CONFIRMED AND end_time < current_time`
- 动作：将上述条件匹配的预约状态更新为 `TIMEOUT`
- 权限：系统/调度，无用户角色

#### 并发控制

##### 1.预约时段并发控制	

- 采用应用层查询区间重叠（`SELECT ... WHERE vet_id = ? AND start_time < ? AND end_time > ? AND id <> ?`）+ 事务内 `FOR UPDATE`控制预约时段唯一性

##### 2.状态转换唯一性

- 确认/完成/取消：UPDATE ... WHERE status=预期状态进行条件更新

#### 关键业务规则

- `end_time > start_time`，时间区间采用左闭右开 `[start_time, end_time)`
- 同一医生同一时间区间内不允许存在其他预约（由应用层区间查询 + `FOR UPDATE` 保证）
- 创建预约时，若区间与现有预约重叠或 `end_time <= start_time`，则拒绝创建
- 修改时间后，无论原状态为 `CREATED` 还是 `CONFIRMED`，目标状态均回落为 `CREATED`，等待医生再次确认
- `COMPLETED` / `CANCELLED` / `TIMEOUT` 均为终态，不可再流转
- 角色权限：`ADMIN` 覆盖全部操作；`RECEPTIONIST` 可创建/改时间/取消；`VET` 可确认/完成/取消

#### 已知限制 / 待定

- 当前无独立用户表，`created_by` / `updated_by` 的引用目标待第 2 周登录/权限实现时确定
- 超时采用后台定时任务极简版，仅做状态流转，不处理通知、补偿或消息
- 不做自动过期取消（与超时不同：超时指已确认但未完成，自动过期取消指创建后未确认/未就诊）
- 若后续支持外部用户（宠物主人）自助预约，需新增 `OWNER` 角色与入口，本设计暂未覆盖
