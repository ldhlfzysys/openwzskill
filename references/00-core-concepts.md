# 核心概念

理解以下三个核心模式，是正确操作五洲 SaaS API 的前提。

---

## 1. 快照机制（ProductSkuSn）

### 为什么需要快照

订单中的商品信息（名称、价格、规格等）在下单时被"冻结"为快照。即使商品后续修改了价格或名称，订单中的记录不受影响。

### 工作流程

```
选择 SKU 并填写数量/价格
        ↓
调用 build-for-xxx 接口生成快照
        ↓
返回 product_sku_sn_ids（整数数组）
        ↓
将 product_sku_sn_ids 传入 create/update 接口
```

### 六种 build 接口

| 场景 | 接口路径 | 必填字段 |
| --- | --- | --- |
| 供应链订单 | `POST /product_sku_sn/product-sku-sns/build-for-supplier-order` | items[].product_sku_id, quantity; 可选 supplier_price, supplier_tax_total, product_sku_sn_id |
| 销售订单 | `POST /product_sku_sn/product-sku-sns/build-for-sale-order` | store_id, items[].product_sku_id, quantity, warehouse_id; 可选 product_stock_id, sale_price, sale_discount_price, product_sku_sn_id |
| 销售退货 | `POST /product_sku_sn/product-sku-sns/build-for-sale-return` | sale_order_id, items[].quantity; 可选 product_sku_sn_id(原订单快照), return_snapshot_id(更新已有退货行) |
| 仓库出入库 | `POST /product_sku_sn/product-sku-sns/build-for-warehouse-io` | type(0/1/2), items[].product_sku_id, quantity; 可选 expire_time, shelf_position, product_stock_id, supplier_product_sku_sn_id |
| 仓库盘点 | `POST /product_sku_sn/product-sku-sns/build-for-warehouse-check` | items[].product_sku_id, actual_quantity; 可选 shelf_position, product_stock_id |
| 仓库移库 | `POST /product_sku_sn/product-sku-sns/build-for-warehouse-transfer` | items[].product_sku_id, product_stock_id, quantity; 可选 shelf_position |

### 读取快照

```
POST /product_sku_sn/product-sku-sns/list-by-ids
Body: { "product_sku_sn_ids": [1, 2, 3] }
→ 返回快照详情数组，包含冻结时的商品名称、价格等信息
```

### 更新已有快照

在 build 请求的 items 中传入已有的 `product_sku_sn_id`，系统会更新该快照而不是新建。

---

## 2. ID→名称解析

系统中大量实体通过 ID 关联，展示时需要查询对应实体获取名称。

### 速查表

| 字段 | 含义 | 查询接口 | 匹配方式 |
| --- | --- | --- | --- |
| `warehouse_id` | 仓库 | `POST /warehouse/warehouses/list` | 返回列表中按 id 匹配取 `name` |
| `supplier_id` | 供应商 | `POST /supplier/suppliers/list` | 按 id 匹配取 `name` |
| `store_id` | 商户 | `POST /sale/stores/list` | 按 id 匹配取 `name` |
| `brand_id` | 品牌 | `POST /product/brands/list` | 按 id 匹配取 `name` |
| `category_ids` | 分类 | `POST /product/categories/list` | 按 id 数组匹配取 `name` |
| `finance_type_id` | 财务类型 | `POST /finance_type/finance-types/list` | 按 id 匹配取 `name` |
| `province_id` | 省份 | `POST /system/provinces/list` 或 `POST /sale/provinces/list` | 按 id 匹配取 `name` |
| `product_sku_sn_ids` | 商品快照 | `POST /product_sku_sn/product-sku-sns/list-by-ids` | 传入 id 数组获取快照详情 |
| `supplier_order_id` | 供应链订单 | `GET /supplier/supplier-orders/{id}` | 获取订单详情 |
| `sale_order_id` | 销售订单 | `GET /sale/sale-orders/{id}` | 获取订单详情 |

### 典型示例

获取供应链订单列表并展示供应商名称：

```
1. GET /supplier/supplier-orders/cache-sync → 拉取供应链订单增量后得到订单列表（含 supplier_id）
2. POST /supplier/suppliers/list       → 得到供应商列表
3. 用 supplier_id 在供应商列表中匹配出 name
```

---

## 3. 状态驱动流程

系统中的订单和任务都由 **status 字段** 驱动，不同状态决定可执行的操作。

### 供应链订单状态（SupplierOrder.status）

| status | 含义   | 可执行操作                         |
| ------ | ------ | ---------------------------------- |
| 0      | 草稿   | 编辑、提交审批（→1）、删除         |
| 1      | 待审批 | 审批通过（→2）、驳回（→0）         |
| 2      | 已审批 | 标记已下单（→3）                   |
| 3      | 已下单 | 标记已发货（→4）                   |
| 4      | 已发货 | 标记已到港（→5）                   |
| 5      | 已到港 | 标记清关中（→6）                   |
| 6      | 清关中 | 标记转运中（→7）                   |
| 7      | 转运中 | 标记已入库（→8），同时创建入库记录 |
| 8      | 已入库 | 结算（→9）                         |
| 9      | 已结算 | 终态，不可变更                     |

### 销售订单状态（SaleOrder.status）

| status | 含义   | 可执行操作                         |
| ------ | ------ | ---------------------------------- |
| 0      | 草稿   | 编辑、提交审批（→1）、删除         |
| 1      | 待审批 | 审批通过（→2）、驳回（→0）         |
| 2      | 已审批 | 标记已出库（→3），需先创建出库记录 |
| 3      | 已出库 | 标记配送中（→4）                   |
| 4      | 配送中 | 结算（→5）                         |
| 5      | 已结算 | 终态                               |
| 6      | 已取消 | 终态                               |

### 销售退货状态（SaleReturn.status）

| status | 含义   |
| ------ | ------ |
| 0      | 退货中 |
| 1      | 已退库 |
| 2      | 已取消 |

### 仓库盘点状态（WarehouseCheck.status）

| status | 含义                        |
| ------ | --------------------------- |
| 0      | 草稿                        |
| 1      | 已完成（需设置 check_time） |

### 状态更新方式

通过 update 接口修改 status 字段：

```
POST /supplier/supplier-orders/{id}/update
Body: { "status": 1 }
```

---

## 4. 仓库出入库类型（WarehouseIO.type）

| type | 含义 | 关联字段                        |
| ---- | ---- | ------------------------------- |
| 0    | 入库 | `order_id`（供应链订单 ID）     |
| 1    | 出库 | `order_id`（销售订单 ID）       |
| 2    | 退库 | `sale_return_id`（销售退货 ID） |

---

## 5. 库存变动记录类型（ProductStockRecord.type）

| type | 含义 |
| ---- | ---- |
| 0    | 入库 |
| 1    | 出库 |
| 2    | 退库 |
