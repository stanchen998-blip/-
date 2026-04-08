# XTrade 后端时序图（维护文档）

> 更新时间：2026-03-30  
> 目的：用时序图描述当前后端真实业务链路，作为 `BACKEND_LOGIC.md` 的配套文档。  
> 约束：本文件只描述当前代码已经实现的逻辑，不写“理想流程”。

---

## 1. 参与方说明

- 买家：下单用户
- 管理员：审核订单、分配采购、审批转单
- 配货员：执行采购任务、申请转单、可走直发
- 仓库：执行采购入库、标记质检不合格、发货
- 订单模块：`/api/checkout` + `/api/orders`
- 采购模块：`/api/procurement`
- 商品库存：
  - 虚拟库存：`saleStockDetails`
  - 真实库存：`stockDetails`

---

## 2. 总主链路

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as 买家
    actor Admin as 管理员
    actor Picker as 配货员
    actor Warehouse as 仓库
    participant Checkout as 结算/下单
    participant Orders as 订单模块
    participant SaleStock as 虚拟库存 saleStockDetails
    participant RealStock as 真实库存 stockDetails
    participant Pool as 采购池
    participant Proc as 采购任务

    Buyer->>Checkout: 提交订单(/api/checkout/cart|direct)
    Checkout->>SaleStock: 校验并扣减虚拟库存
    Checkout->>Orders: 创建订单与单件 orderItems\n状态=待审核
    Admin->>Orders: 审批订单(/api/orders/{id}/approve)
    Orders->>RealStock: 按 orderItem 判断是否可锁库

    alt 真实库存足够
        Orders->>Orders: orderItem -> 质检通过/待发货
    else 真实库存不足
        Orders->>Orders: orderItem -> 已接单/待调货
        Orders->>Pool: 缺货项进入采购池视图
        Admin->>Pool: 分配给配货员
        Picker->>Proc: 创建采购任务

        alt 采购后进仓库
            Warehouse->>Proc: 仓库入库(/warehouse/complete)
            Proc->>RealStock: 增加真实库存
            Proc->>Orders: 对应缺货 orderItem -> 质检通过/待发货
        else 配货员采购后直发客户
            Picker->>Proc: 直发完成(/direct-ship/complete + proofImage)
            Proc->>Orders: 对应 orderItem -> 已发货
        end
    end

    alt 已进入待发货
        Warehouse->>Orders: 发货(/api/orders/{id}/ship)
        Orders->>RealStock: 扣减真实库存
        Orders->>Orders: orderItem -> 已发货
    end
```

---

## 3. 下单阶段

### 3.1 买家下单

```mermaid
sequenceDiagram
    autonumber
    actor Buyer as 买家
    participant Checkout as /api/checkout
    participant SaleStock as saleStockDetails
    participant Orders as orders + order_items

    Buyer->>Checkout: 下单(/cart 或 /direct)
    Checkout->>Checkout: 按商品+尺码汇总需求
    Checkout->>SaleStock: 校验可售库存

    alt 虚拟库存不足
        SaleStock-->>Checkout: 库存不足
        Checkout-->>Buyer: 下单失败
    else 虚拟库存充足
        Checkout->>SaleStock: 扣减对应 saleStockDetails
        Checkout->>Orders: 创建订单 status=待审核
        Checkout->>Orders: 每双生成一条 OrderItem(quantity=1)
        Checkout-->>Buyer: 返回订单
    end
```

关键点：

- 下单时扣的是虚拟库存，不是真实库存。
- 一双鞋对应一条 `OrderItem`，后续采购、转单、质检、发货都追踪这条 `orderItemId`。

### 3.2 管理员驳回订单

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 管理员
    participant Orders as /api/orders/{id}/reject
    participant SaleStock as saleStockDetails

    Admin->>Orders: 驳回订单
    Orders->>Orders: 检查 sale_stock_deducted

    alt 该订单下单时已扣虚拟库存
        Orders->>SaleStock: 按订单项逐尺码恢复虚拟库存
    end

    Orders->>Orders: 订单状态 -> 已取消
    Orders->>Orders: 全部 orderItem -> 已取消
```

---

## 4. 审批后有货链路

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 管理员
    participant Orders as /api/orders/{id}/approve
    participant RealStock as stockDetails
    participant Locked as 已锁库统计(待发货订单项)

    Admin->>Orders: 审批订单
    loop 每个 orderItem
        Orders->>RealStock: 查询该商品尺码真实库存
        Orders->>Locked: 查询同款同码已锁数量
        Orders->>Orders: available_to_lock = 真实库存 - 已锁数量

        alt available_to_lock >= 1
            Orders->>Orders: orderItem -> 质检通过/待发货
        else available_to_lock <= 0
            Orders->>Orders: orderItem -> 已接单/待调货
        end
    end
```

关键点：

- “有货”不代表自动发货，只代表能锁到真实库存。
- 锁库成功后的状态是 `质检通过/待发货`。
- 真正扣真实库存发生在后续发货时，不发生在审批时。

---

## 5. 审批后缺货链路

### 5.1 缺货进入采购池

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 管理员
    participant Orders as /api/orders/{id}/approve
    participant Pool as /api/procurement/pool

    Admin->>Orders: 审批订单
    Orders->>Orders: 无法锁库的 orderItem -> 已接单/待调货
    Pool->>Pool: 基于缺货 orderItem 聚合商品+尺码缺口
    Pool-->>Admin: 返回 missingQuantity / missingOrderItemIds / unassignedOrderItemIds
```

说明：

- 采购池不是单独“入池写表”，而是根据缺货 `orderItem` 实时聚合出来的视图。

### 5.2 管理员分配采购需求

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 管理员
    actor Picker as 配货员
    participant Pool as /api/procurement/pool/assign
    participant Assign as procurement_assignments
    participant AssignItem as procurement_assignment_items

    Admin->>Pool: 按商品+尺码分配 quantity 或 orderItemIds
    alt 显式传 orderItemIds
        Pool->>AssignItem: 把指定 orderItemIds 绑定给目标配货员
    else 不传 orderItemIds
        Pool->>AssignItem: 按订单时间顺序自动挑选缺货明细
    end
    Pool->>Assign: 累加/创建 assignment
    Picker-->>Assign: 后续通过 /api/procurement/assignments 查看自己的任务
```

### 5.3 配货员查看自己的分配

```mermaid
sequenceDiagram
    autonumber
    actor Picker as 配货员
    participant AssignAPI as /api/procurement/assignments
    participant Assign as procurement_assignments
    participant AssignItem as procurement_assignment_items

    Picker->>AssignAPI: 查询自己的分配
    AssignAPI->>Assign: 查询 picker_id 对应 assignment
    AssignAPI->>AssignItem: 统计 assigned / pending_transfer / tasked / completed / qc_reject_pending
    AssignAPI-->>Picker: 返回 assignment + orderItemIds 分类结果
```

---

## 6. 转单审批链路

```mermaid
sequenceDiagram
    autonumber
    actor PickerA as 原配货员
    actor Admin as 管理员
    actor PickerB as 目标配货员
    participant Transfer as /api/procurement/assignments/{id}/transfer
    participant Req as procurement_transfer_requests
    participant AssignItem as procurement_assignment_items

    PickerA->>Transfer: 提交转单申请(quantity 或 orderItemIds)
    Transfer->>AssignItem: 把对应 assignment_item 标记为 pending_transfer
    Transfer->>Req: 创建转单申请

    alt 管理员审批通过
        Admin->>Req: approve
        Req->>AssignItem: 把具体 orderItemIds 挪到 PickerB 的 assignment
        PickerB-->>Req: 后续可在自己的 assignments 中看到这些明细
    else 管理员驳回
        Admin->>Req: reject
        Req->>AssignItem: pending_transfer -> assigned
    end
```

关键点：

- 转单提交后，数量会先被冻结，不允许继续建采购任务。
- 只有审批通过，`orderItemId` 才真正转到别的配货员名下。

---

## 7. 采购任务链路

### 7.1 创建采购任务

```mermaid
sequenceDiagram
    autonumber
    actor Picker as 配货员
    participant TaskAPI as /api/procurement/tasks
    participant Assign as procurement_assignments
    participant AssignItem as procurement_assignment_items
    participant Task as procurement_tasks
    participant TaskItem as procurement_task_items

    Picker->>TaskAPI: 创建采购任务(assignmentId + quantity/orderItemIds + trackingNumber)
    TaskAPI->>Assign: 校验 assignment availableQuantity
    TaskAPI->>AssignItem: 绑定具体 orderItemIds
    TaskAPI->>Task: 创建或物理合并 procurement_task
    TaskAPI->>TaskItem: 为每双鞋创建一条 task_item
```

关键点：

- 一个采购任务里仍然是逐双绑定 `orderItemId`。
- 后续仓库入库或直发，都是对这些 `task_items` 按双完成。

---

## 8. 缺货采购后的两条分支

## 8.1 分支 A：采购后进入仓库

```mermaid
sequenceDiagram
    autonumber
    actor Warehouse as 仓库
    participant Proc as /api/procurement/tasks/{id}/warehouse/complete
    participant RealStock as stockDetails
    participant InvLog as inventory_logs
    participant Orders as order_items
    participant Trace as order_item_procurement_logs

    Warehouse->>Proc: 按 quantity 完成入库
    Proc->>RealStock: 增加真实库存
    Proc->>InvLog: 写 inbound 审计
    loop 每个完成的 task_item
        Proc->>Orders: 对应 orderItem -> 质检通过/待发货
        Proc->>Trace: 记录采购成本/供应商/回单号/承运商
    end
```

结果：

- 货先进仓库。
- 对应缺货的订单项会被直接锁库，进入 `质检通过/待发货`。
- 后面再由仓库走发货接口。

## 8.2 分支 B：配货员采购后直接发客户（直发）

```mermaid
sequenceDiagram
    autonumber
    actor Picker as 配货员
    participant Proc as /api/procurement/tasks/{id}/direct-ship/complete
    participant Files as 直发凭证图片目录
    participant Orders as order_items
    participant Trace as order_item_procurement_logs
    participant TaskEvent as procurement_task_events

    Picker->>Proc: 提交直发完成(quantity + note + proofImage)
    Proc->>Files: 保存直发凭证图片
    loop 每个完成的 task_item
        Proc->>Orders: 对应 orderItem -> 已发货
        Proc->>Orders: 写 outbound_tracking_number = 任务 trackingNumber
        Proc->>Orders: 写 outbound_carrier = 任务 carrier
        Proc->>Trace: 写 action=direct_ship + proofImageUrl
    end
    Proc->>TaskEvent: 写 direct_ship_complete 事件 + proofImageUrl
```

结果：

- 这条链路不经过仓库。
- 不增加真实库存。
- 不扣减真实库存。
- 订单项直接进入 `已发货`。
- 物流信息直接取采购任务的 `trackingNumber` / `carrier`。

---

## 9. 仓库质检不合格池链路

```mermaid
sequenceDiagram
    autonumber
    actor Warehouse as 仓库
    actor Picker as 配货员
    participant RejectAPI as /api/procurement/tasks/{id}/warehouse/mark-qc-reject
    participant RejectPool as procurement_qc_rejects
    participant AssignItem as procurement_assignment_items
    participant ProcessAPI as /api/procurement/qc-rejects/{id}/process
    participant Orders as order_items

    Warehouse->>RejectAPI: 标记部分待处理数量为不合格
    RejectAPI->>RejectPool: 生成 qc_reject 记录(逐双绑定 orderItemId)
    RejectAPI->>AssignItem: assignment_item -> qc_reject_pending

    Picker->>RejectPool: 查看自己的不合格池
    Picker->>ProcessAPI: 填 returnTrackingNumber + replacementTrackingNumber
    ProcessAPI->>RejectPool: 状态 -> processed
    ProcessAPI->>AssignItem: qc_reject_pending -> assigned
    ProcessAPI->>Orders: 对应 orderItem -> 已接单/待调货
```

结果：

- 仓库不是直接“退货完成”，而是先打入不合格池。
- 配货员处理完退回单号和新发单号后，这双鞋重新回到正常采购链路。

---

## 10. 仓库发货链路

```mermaid
sequenceDiagram
    autonumber
    actor Warehouse as 仓库/管理员
    participant Orders as /api/orders/{id}/ship
    participant RealStock as stockDetails
    participant InvLog as inventory_logs

    Warehouse->>Orders: 发货(itemIndexes 或 shipments[])
    loop 每个发货订单项
        Orders->>RealStock: 扣减真实库存
        Orders->>InvLog: 写 outbound 审计
        Orders->>Orders: orderItem -> 已发货
        Orders->>Orders: 写 outbound_tracking_number / outbound_carrier / outbound_note
    end
```

关键点：

- 仓库发货链路只适用于已经是 `质检通过/待发货` 的订单项。
- 如果同一订单 10 双鞋要分 10 个物流单发，后端支持逐双发货。

---

## 11. 撤销发货链路

```mermaid
sequenceDiagram
    autonumber
    actor Warehouse as 仓库/管理员
    participant Orders as /api/orders/{id}/revoke-ship
    participant RealStock as stockDetails
    participant InvLog as inventory_logs

    Warehouse->>Orders: 撤销发货(itemIndexes 或 trackingNumbers)
    loop 每个被撤销的已发货订单项
        Orders->>RealStock: 加回真实库存
        Orders->>InvLog: 写 revoke_outbound 审计
        Orders->>Orders: orderItem -> 质检通过/待发货
        Orders->>Orders: 清空出库物流信息
    end
```

---

## 12. 单双鞋的反查链路

```mermaid
sequenceDiagram
    autonumber
    actor Admin as 管理员/运营
    participant TraceAPI as /api/orders/{orderId}/items/{itemId}/procurement
    participant Item as order_items
    participant Log as order_item_procurement_logs
    participant Task as procurement_tasks
    participant Event as procurement_task_events
    participant Transfer as procurement_transfer_requests

    Admin->>TraceAPI: 查询某个 orderItemId 的采购轨迹
    TraceAPI->>Item: 查订单项当前状态
    TraceAPI->>Log: 查采购履历
    TraceAPI->>Task: 查关联采购任务
    TraceAPI->>Event: 查任务事件
    TraceAPI->>Transfer: 查转单申请历史
    TraceAPI-->>Admin: 返回该双鞋从下单到采购/转单/直发/入库的轨迹
```

可追踪内容包括：

- 这双鞋有没有缺货
- 分配给了哪个配货员
- 是否发起过转单
- 建过哪个采购任务
- 是仓库入库还是采购直发
- 采购成本、市场价、供应商、结算信息
- 直发凭证图片

---

## 13. 当前逻辑结论

### 13.1 有货

- 下单时只扣虚拟库存。
- 审批后如果真实库存可锁，则进入 `质检通过/待发货`。
- 不会自动发货。
- 后续仍需走仓库发货接口。

### 13.2 没货

- 审批后进入 `已接单/待调货`。
- 采购池按商品+尺码聚合缺口，但底层始终绑定具体 `orderItemId`。
- 后续可以：
  - 采购后进仓库
  - 采购后直发客户

### 13.3 直发

- 直发不是“有货自动发货”。
- 直发是“缺货 -> 进入采购池 -> 建采购任务 -> 配货员采购后直接发客户”的分支。
- 当前正式接口：`POST /api/procurement/tasks/{id}/direct-ship/complete`

