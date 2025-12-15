# 🅿️ 停车场系统 API 接口规范

> 本文档用于 **前后端联调、后端协作、验收与维护**
> **数据库结构不在本文修改范围**，以《数据库设计.md》为唯一准绳

## 0️⃣ 全局约束与设计原则（必须先读）

| 项目       | 说明                                            |
| ---------- | ----------------------------------------------- |
| 认证方式   | JWT，通过 **HttpOnly + Secure Cookie** 自动携带 |
| Token 规则 | ❌ 前端不读取、不存储、不拼接 token              |
| 状态驱动   | 停车流程为 **后端状态机驱动**                   |
| 资源分配   | ❌ 前端不得参与车位分配、计费、状态变更          |
| Redis 职责 | **仅用于实时状态**，不作为业务事实源            |
| DB 职责    | **唯一事实源**（计费、记录、支付结果）          |
| 并发原则   | 所有“状态写入”接口必须具备 **幂等或冲突检测**   |

## 1️⃣ 认证与登录接口

### 1.1 用户登录

| 项目     | 内容                                |
| -------- | ----------------------------------- |
| 路由     | `POST /api/v1/auth/login`           |
| 权限     | 匿名                                |
| 入参     | `{ phone, password }`               |
| 出参     | `{ user: { id, level, nickname } }` |
| Cookie   | ✅ 成功后写入 `auth_token`           |
| 禁止行为 | ❌ 返回 token 字符串                 |

## 2️⃣ 停车核心业务（状态机驱动）

> **核心原则：前端只“请求动作”，不“决定结果”**

### 2.1 入场（创建记录 + 分配车位）

| 项目   | 内容                                               |
| ------ | -------------------------------------------------- |
| 路由   | `POST /api/v1/parking/entry`                       |
| 权限   | 已登录用户（`user.level ≥ 0 && is_active = true`） |
| 入参   | `{ vehicle_id }`                                   |
| 出参   | `{ record_id, slot_no, floor_no }`                 |
| 状态码 | `201 Created`                                      |

**后端行为（必须原子化）：**

- 校验会员等级 ≥ 楼层限制
- 校验车辆能源类型匹配
- Redis 判断空闲车位
- 创建 `parking_record`（仅 entry_time）
- 更新 Redis：
  - `slot:{id} = 1`
  - `floor:{id}:free_count--`
  - `user:{id}:current_parking = recordId`

⚠️ **前端禁止传 slot_id**

### 2.2 费用预览（只读）

| 项目     | 内容                                 |
| -------- | ------------------------------------ |
| 路由     | `GET /api/v1/parking/{recordId}/fee` |
| 权限     | 记录所属用户                         |
| 出参     | `{ duration_min, fee_amount }`       |
| 数据写入 | ❌ 无                                 |

### 2.3 支付确认（唯一修改 paid 的接口）

| 项目 | 内容                                  |
| ---- | ------------------------------------- |
| 路由 | `POST /api/v1/parking/{recordId}/pay` |
| 权限 | 记录所属用户                          |
| 出参 | `{ paid: true, fee_amount }`          |

**事务内行为：**

- 重新计算费用（防篡改）
- 更新 `parking_record.fee_amount`
- 更新 `parking_record.paid = true`

✅ **唯一允许修改 paid 的接口（陈负责）**

### 2.4 离场（释放资源）

| 项目     | 内容                                   |
| -------- | -------------------------------------- |
| 路由     | `POST /api/v1/parking/{recordId}/exit` |
| 前置条件 | `paid == true`                         |
| 出参     | `{ message: "exit success" }`          |

**后端行为：**

- 写 `exit_time`
- 写 `duration_min`
- 释放 Redis 状态

## 3️⃣ 用户与车辆管理（陈负责）

| 功能         | 路由                       | 方法 | 权限 |
| ------------ | -------------------------- | ---- | ---- |
| 注册         | `/api/v1/user/register`    | POST | 匿名 |
| 查询车辆     | `/api/v1/user/vehicles`    | GET  | 用户 |
| 绑定车辆     | `/api/v1/user/vehicle`     | POST | 用户 |
| 商场消费记录 | `/api/v1/user/consumption` | GET  | 用户 |

⚠️ **停车费不计入 mall_consumption**

## 4️⃣ VIP 服务（陈负责）

| 功能     | 路由                       | 权限             |
| -------- | -------------------------- | ---------------- |
| 创建服务 | `POST /api/v1/vip/service` | `user.level ≥ 1` |
| 查询服务 | `GET /api/v1/vip/service`  | `user.level ≥ 1` |

✅ 计入 `user.total_consumption`

## 5️⃣ 管理员接口（李 & 陈）

| 功能     | 路由                                  | 方法 |
| -------- | ------------------------------------- | ---- |
| 楼层空位 | `/api/v1/admin/floor/{id}/free-count` | GET  |
| 停车记录 | `/api/v1/admin/parking/records`       | GET  |
| 修改计费 | `/api/v1/admin/fee/rule`              | PUT  |
| 用户封禁 | `/api/v1/admin/user/status`           | PUT  |

## 6️⃣ 查询类接口（只读）

| 功能     | 路由                           | 说明    |
| -------- | ------------------------------ | ------- |
| 停车历史 | `/api/v1/user/parking/history` | DB 查询 |
| 可用车位 | `/api/v1/parking/slots`        | 仅展示  |

⚠️ **展示 ≠ 分配**

## 7️⃣ HTTP 状态码规范

| 状态码 | 含义     |
| ------ | -------- |
| 200    | 成功     |
| 201    | 创建成功 |
| 400    | 参数错误 |
| 401    | 未认证   |
| 403    | 权限不足 |
| 404    | 不存在   |
| 409    | 状态冲突 |

### 👥 协作边界（最终确认）

| 模块     | 负责人 | 严禁行为           |
| -------- | ------ | ------------------ |
| 停车核心 | 李     | 修改 paid、登录    |
| 用户系统 | 陈     | 车位分配、Redis    |
| 前端     | 梁     | 传 slot_id、拼计费 |



## 第一部分：类名-属性-属性内容说明（基于文档中的“类与表映射说明”和字段注释）

你这段内容**不是“乱”，而是被压扁了**：本来是一个很清晰的「领域对象 → 字段 → 含义」三列结构，现在全糊在一行里，可读性直接归零。
我按**工程文档可交付标准**，给你规范成**统一表格格式**，字段语义不改、一字不增，只做**结构整理与命名对齐**。

------

## 📦 领域对象 · 类名 — 属性 — 属性说明（规范版）

------

### **User（用户）**

| 属性名            | 类型 / 约束 | 属性内容说明                          |
| ----------------- | ----------- | ------------------------------------- |
| id                | BIGINT PK   | 用户唯一标识                          |
| phone             | VARCHAR     | 登录手机号                            |
| password_hash     | VARCHAR     | 密码哈希（绝不存明文）                |
| nickname          | VARCHAR     | 显示昵称                              |
| level             | INT         | 会员等级：0 普通 / 1 VIP / 2 超级会员 |
| total_consumption | DECIMAL     | 累计消费金额（用于优惠 / 升级）       |
| enter_count       | INT         | 进入停车场次数                        |
| is_active         | BOOLEAN     | 账号是否有效（用于封禁）              |
| created_at        | DATETIME    | 注册时间                              |

------

### **Admin（管理员）**

| 属性名        | 类型 / 约束 | 属性内容说明                        |
| ------------- | ----------- | ----------------------------------- |
| id            | BIGINT PK   | 管理员唯一标识                      |
| username      | VARCHAR     | 管理员账号                          |
| password_hash | VARCHAR     | 管理员密码哈希                      |
| role          | VARCHAR     | 角色（如：超级管理员 / 普通管理员） |
| created_at    | DATETIME    | 创建时间                            |

------

### **Vehicle（车辆）**

| 属性名       | 类型 / 约束    | 属性内容说明                |
| ------------ | -------------- | --------------------------- |
| id           | BIGINT PK      | 车辆唯一标识                |
| plate_number | VARCHAR UNIQUE | 车牌号（全系统唯一）        |
| energy_type  | INT            | 能源类型：0 燃油 / 1 新能源 |
| user_id      | BIGINT FK      | 所属用户 ID                 |
| created_at   | DATETIME       | 录入时间                    |

------

### **ParkingFloor（停车楼层）**

| 属性名      | 类型 / 约束 | 属性内容说明             |
| ----------- | ----------- | ------------------------ |
| id          | BIGINT PK   | 楼层唯一标识             |
| floor_no    | INT         | 楼层编号（如 1 / 2 / 3） |
| level_limit | INT         | 进入该层最低会员等级     |

------

### **ParkingZone（停车区域）**

| 属性名      | 类型 / 约束 | 属性内容说明                    |
| ----------- | ----------- | ------------------------------- |
| id          | BIGINT PK   | 区域唯一标识                    |
| floor_id    | BIGINT FK   | 所属楼层 ID                     |
| energy_type | INT         | 区域能源限制：0 燃油 / 1 新能源 |

------

### **ParkingBlock（停车小区）**

| 属性名   | 类型 / 约束 | 属性内容说明              |
| -------- | ----------- | ------------------------- |
| id       | BIGINT PK   | 小区唯一标识              |
| zone_id  | BIGINT FK   | 所属区域 ID               |
| block_no | VARCHAR     | 小区编号（A / B / C / D） |
| capacity | INT         | 小区最大车位数            |

------

### **ParkingSlot（停车位）**

| 属性名      | 类型 / 约束 | 属性内容说明         |
| ----------- | ----------- | -------------------- |
| id          | BIGINT PK   | 车位唯一标识         |
| block_id    | BIGINT FK   | 所属小区 ID          |
| slot_no     | VARCHAR     | 车位编号（如 A-001） |
| fee_rule_id | BIGINT FK   | 计费规则 ID          |

------

### **ParkingRecord（停车记录）**

| 属性名       | 类型 / 约束 | 属性内容说明     |
| ------------ | ----------- | ---------------- |
| id           | BIGINT PK   | 停车记录唯一标识 |
| user_id      | BIGINT FK   | 用户 ID          |
| vehicle_id   | BIGINT FK   | 车辆 ID          |
| slot_id      | BIGINT FK   | 车位 ID          |
| entry_time   | DATETIME    | 入场时间         |
| exit_time    | DATETIME    | 出场时间         |
| duration_min | INT         | 停车时长（分钟） |
| fee_amount   | DECIMAL     | 实际费用         |
| paid         | BOOLEAN     | 是否已缴费       |

------

### **FeeRule（计费规则）**

| 属性名         | 类型 / 约束 | 属性内容说明     |
| -------------- | ----------- | ---------------- |
| id             | BIGINT PK   | 计费规则唯一标识 |
| floor_id       | BIGINT FK   | 适用楼层 ID      |
| price_per_hour | DECIMAL     | 每小时价格       |
| free_minutes   | INT         | 免费停车分钟数   |

------

### **Consumption（商场消费记录）**

| 属性名     | 类型 / 约束 | 属性内容说明     |
| ---------- | ----------- | ---------------- |
| id         | BIGINT PK   | 消费记录唯一标识 |
| user_id    | BIGINT FK   | 用户 ID          |
| amount     | DECIMAL     | 消费金额         |
| created_at | DATETIME    | 消费时间         |

------

### **VipService（VIP 服务订单）**

| 属性名       | 类型 / 约束 | 属性内容说明                  |
| ------------ | ----------- | ----------------------------- |
| id           | BIGINT PK   | 服务订单唯一标识              |
| user_id      | BIGINT FK   | 用户 ID                       |
| service_type | INT         | 服务类型：0 洗车 / 1 酒店     |
| status       | INT         | 订单状态（处理中 / 已完成等） |
| created_at   | DATETIME    | 创建时间                      |

------

### 🧠 额外说明（但不写进表结构）

- Redis 状态（如 `slot:{id}`、`floor:{id}:free_count`）**不映射为领域对象**
- 所有金额字段默认 **DECIMAL，禁止浮点**
- 所有状态字段应有**枚举或约束定义**（即使数据库未显式声明）

## 第二部分：主要功能点 - 路由与接口设计（严格基于文档内容）

| 主要功能点                          | 页面路由               | API路由                                    | 权限                   | 方法名称 (Handler)    | 入参 (请求参数)                        | 出参 (方法输出参数)                                          |
| ----------------------------------- | ---------------------- | ------------------------------------------ | ---------------------- | --------------------- | -------------------------------------- | ------------------------------------------------------------ |
| 用户注册                            | /user/register         | POST /api/v1/user/register                 | 匿名                   | RegisterUser          | phone, password                        | status (201), message, user.id                               |
| 用户登录                            | /user/login            | POST /api/v1/user/login                    | 匿名                   | LoginUser             | phone, password                        | status (200), token, user.level                              |
| 查询用户车辆列表                    | /user/vehicles         | GET /api/v1/user/vehicles                  | 普通用户及以上         | GetUserVehicles       | —                                      | status (200), list of {id, plate_number, energy_type}        |
| 绑定新车辆                          | /user/vehicle/add      | POST /api/v1/user/vehicle                  | 普通用户及以上         | BindVehicle           | plate_number, energy_type              | status (201), vehicle.id                                     |
| 查询可用车位（按会员等级+能源类型） | /parking/slots         | GET /api/v1/parking/slots?floor=1&energy=1 | 普通用户及以上         | GetAvailableSlots     | floor_no (可选), energy_type (可选)    | status (200), list of {slot_id, slot_no, block_no, fee_rule} |
| 用户入场停车（占用车位）            | /parking/entry         | POST /api/v1/parking/entry                 | 普通用户及以上         | StartParking          | vehicle_id                             | status (201), parking_record.id, message                     |
| 用户出场结算                        | /parking/exit          | POST /api/v1/parking/exit                  | 普通用户及以上         | EndParking            | record_id                              | status (200), fee_amount, paid (true/false), message         |
| 查询停车记录                        | /user/history          | GET /api/v1/user/parking/history           | 普通用户及以上         | GetUserParkingHistory | —                                      | status (200), list of parking records                        |
| 查询商场消费记录                    | /user/consumption      | GET /api/v1/user/consumption               | 普通用户及以上         | GetUserConsumption    | —                                      | status (200), list of {amount, created_at}                   |
| VIP用户预约洗车/酒店服务            | /vip/service/order     | POST /api/v1/vip/service                   | VIP或超级会员          | CreateVipServiceOrder | service_type (0/1)                     | status (201), order.id, message                              |
| 查询VIP服务订单状态                 | /vip/service/orders    | GET /api/v1/vip/service                    | VIP或超级会员          | GetVipServiceOrders   | —                                      | status (200), list of {id, service_type, status, created_at} |
| 管理员查看所有停车记录              | /admin/parking/records | GET /api/v1/admin/parking/records          | 管理员                 | GetAllParkingRecords  | —                                      | status (200), list of full parking records                   |
| 管理员管理计费规则                  | /admin/fee/rules       | PUT /api/v1/admin/fee/rule                 | 管理员（角色需有权限） | UpdateFeeRule         | floor_id, price_per_hour, free_minutes | status (200), message                                        |
| 管理员查看楼层空位数（来自Redis）   | /admin/floor/status    | GET /api/v1/admin/floor/{floorId}/free     | 管理员                 | GetFloorFreeCount     | floorId (path)                         | status (200), free_count (INT)                               |
| 管理员封禁/启用用户                 | /admin/user/toggle     | PUT /api/v1/admin/user/status              | 管理员                 | ToggleUserStatus      | user_id, is_active (boolean)           | status (200), message                                        |

> 说明：
> - 权限说明：
>   - “普通用户及以上”：指 `user.level >= 0` 且 `is_active = true`
>   - “VIP或超级会员”：指 `user.level >= 1`
>   - “管理员”：需通过 `admin` 表验证，且 `role` 允许操作
> - 所有 API 均假设使用 JWT 或 Session 认证，用户身份由 token 解析
> - Redis 状态（如车位占用、空位数）仅在接口内部读写，不暴露给前端直接操作
> - 未包含“优惠计算”具体接口，因其通常在结算时自动应用，可由 `EndParking` 内部调用 `discount_rule` 逻辑

## 李（停车场核心业务开发）

### 负责模块：
- **停车场资源建模**
- **车辆进出流程**
- **计费与优惠计算**
- **Redis 实时状态管理**

### 涉及数据库表：
- `vehicle`
- `parking_floor` / `parking_zone` / `parking_block` / `parking_slot`
- `parking_record`
- `fee_rule`
- `discount_rule`
- `mall_consumption`（仅用于读取消费总额判断优惠）

### 关键功能点（含逻辑说明）：

| 功能点           | 说明                                                         | 涉及表                                                       | Redis 操作                                                   | 输出给前端/陈                              |
| ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------ |
| **车辆入场**     | 校验用户等级 ≥ 楼层 level_limit；校验车辆能源类型匹配区域；分配空闲车位 | user, vehicle, parking_floor, parking_zone, parking_block, parking_slot | `slot:{id}=1`，`floor:{floorId}:free_count--`，`user:{uid}:current_parking=recordId` | 返回 `parking_record.id`、分配的 `slot_no` |
| **车辆出场**     | 根据 entry_time 计算 duration_min；查 fee_rule 算 fee_amount；查 total_consumption 应用 discount | parking_record, fee_rule, user, discount_rule                | 清除 `user:{uid}:current_parking`；`slot:{id}=0`；`floor:{floorId}:free_count++` | 返回 `fee_amount`、`paid=false`            |
| **车位分配逻辑** | 按楼层→区域（能源）→小区（A/B/C/D）→空闲车位 顺序分配        | parking_floor → parking_zone → parking_block → parking_slot  | 读 `slot:{id}` 判断是否空闲                                  | 返回具体 `slot_id`                         |
| **计费规则应用** | 根据 floor_id 获取 price_per_hour 和 free_minutes；前 free_minutes 免费，超时按小时阶梯计费 | fee_rule, parking_record                                     | —                                                            | `fee_amount` 数值                          |
| **优惠逻辑**     | 若 `user.total_consumption ≥ discount_rule.min_amount`，则 `fee_amount *= discount_rate` | user, discount_rule                                          | —                                                            | 优惠后 `fee_amount`                        |
| **防重复入场**   | 检查 `user:{userId}:current_parking` 是否存在，存在则拒绝入场 | —                                                            | 读 `user:{uid}:current_parking`                              | 返回错误提示                               |

> ⚠️ 注意：**不处理缴费动作**（由陈的“出场前缴费提示”触发，实际支付状态更新由陈完成）

---

## 陈（用户系统 + 会员体系 + 服务功能）

### 负责模块：
- **用户身份与权限**
- **会员等级管理**
- **消费记录**
- **VIP专属服务**
- **缴费状态更新**

### 涉及数据库表：
- `user`
- `admin`
- `mall_consumption`
- `vip_service_order`

### 关键功能点：

| 功能点             | 说明                                                         | 涉及表                  | 与李的交互                              |
| ------------------ | ------------------------------------------------------------ | ----------------------- | --------------------------------------- |
| **用户注册/登录**  | 手机号+密码注册/登录，返回 token + user.level                | user                    | —                                       |
| **管理员登录**     | username+password，验证 role                                 | admin                   | —                                       |
| **会员等级判定**   | 根据 `total_consumption` 和 `enter_count` 判断是否升级（如 ≥500 元升 VIP） | user                    | 调用李的“优惠逻辑”前需确保 level 准确   |
| **消费记录写入**   | 用户在商场消费后，写入 `mall_consumption`，并累加 `user.total_consumption` | mall_consumption, user  | 消费数据供李计算优惠                    |
| **VIP服务下单**    | 仅 level ≥1 可创建洗车（0）/酒店（1）订单                    | vip_service_order, user | 独立功能，无需李参与                    |
| **出场前缴费提示** | 前端请求时，调用李的“出场结算”接口获取 `fee_amount`，展示缴费页；用户支付后，**更新 `parking_record.paid = true`** | parking_record          | **关键对接点**：你负责将 paid 设为 true |

> ✅ 你**不实现**车位分配、计时、计费公式，但**必须调用李的接口获取费用**，并在支付成功后**更新 paid 字段**。

---

## 梁（前端 + Controller + 联调）

### 负责模块：
- **前后端对接**
- **页面渲染**
- **API 调用统一封装**

### 页面与 API 对接清单：

| 页面           | URL 路由         | 调用的后端 API（负责人）                                     | 说明                           |
| -------------- | ---------------- | ------------------------------------------------------------ | ------------------------------ |
| 用户登录页     | `/login`         | POST `/api/user/login`（陈）                                 | 区分用户/管理员登录            |
| 管理员后台首页 | `/admin`         | GET `/api/admin/floors/status`（李）<br>GET `/api/admin/records`（李） | 显示各楼层空位、当前停车车辆   |
| 用户首页       | `/user`          | GET `/api/user/info`（陈）<br>GET `/api/parking/available`（李） | 显示会员等级、总消费、可用车位 |
| 车辆入场页     | `/parking/entry` | POST `/api/parking/entry`（李）                              | 选择车辆、自动分配车位         |
| 缴费页面       | `/pay`           | GET `/api/parking/exit-preview`（李）<br>POST `/api/parking/pay`（陈） | 先预览费用，再确认支付         |
| VIP服务页      | `/vip/service`   | GET `/api/vip/orders`（陈）<br>POST `/api/vip/order`（陈）   | 显示洗车/酒店订单状态          |
| 历史记录       | `/user/history`  | GET `/api/user/parking/history`（李）<br>GET `/api/user/consumption`（陈） | 停车记录 + 商场消费            |

### Controller 层建议：
- 所有 API 入口由梁统一定义（如 `ParkingController`, `UserController`, `AdminController`）

- 梁**不写业务逻辑**，只调用李和陈提供的 **Service 层方法**

  ​


- 确保李的 `fee_amount` 能正确返回给前端
- 确保陈的 `paid = true` 更新后，李的 Redis 状态已释放
- 管理员页面需实时刷新 `floor:{id}:free_count`

---

## 🔗 协作边界清晰总结

| 功能                     | 负责人 | 协作者                          | 交付物                           |
| ------------------------ | ------ | ------------------------------- | -------------------------------- |
| 车怎么停/走/算钱         | 李     | 梁（调用 API）                  | parking_record + Redis 状态      |
| 人是谁/啥等级/能享受啥   | 陈     | 李（读 user.level）、梁（展示） | user 表 + vip_service_order      |
| 页面能不能点、数据对不对 | 梁     | 李、陈                          | 可运行的 Web 界面 + API 联调通过 |

​