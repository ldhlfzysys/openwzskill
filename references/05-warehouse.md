# 仓库模块

管理仓库、出入库操作、库存盘点和仓库间移库。

---

## 仓库（Warehouse）

### 接口

| 操作 | 方法 | 路径                                |
| ---- | ---- | ----------------------------------- |
| 列表 | POST | `/warehouse/warehouses/list`        |
| 创建 | POST | `/warehouse/warehouses/create`      |
| 详情 | GET  | `/warehouse/warehouses/{id}`        |
| 更新 | POST | `/warehouse/warehouses/{id}/update` |
| 删除 | POST | `/warehouse/warehouses/{id}/delete` |

### 关键字段

- `storage_cost_per_cbm_per_day`：每立方米每天的仓储费用，用于成本计算

---

## 出入库记录（WarehouseIO）

### 接口

| 操作 | 方法 | 路径                                   |
| ---- | ---- | -------------------------------------- |
| 列表 | POST | `/warehouse/warehouse-ios/list`        |
| 创建 | POST | `/warehouse/warehouse-ios/create`      |
| 更新 | POST | `/warehouse/warehouse-ios/{id}/update` |
| 删除 | POST | `/warehouse/warehouse-ios/{id}/delete` |

### 三种类型

| type | 含义 | 关联字段                     | 触发场景               |
| ---- | ---- | ---------------------------- | ---------------------- |
| 0    | 入库 | `order_id` = 供应链订单 ID   | 供应链订单标记已入库时 |
| 1    | 出库 | `order_id` = 销售订单 ID     | 销售订单审批通过后出库 |
| 2    | 退库 | `sale_return_id` = 退货单 ID | 销售退货商品退回仓库   |

### 需要解析的 ID

- `warehouse_id` → 仓库名称

---

## 入库操作（type=0）

关联供应链订单。通常在供应链订单 status 从 7（转运中）变为 8（已入库）时执行。

```
1. 构建入库快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-io
   Body: {
     "type": 0,
     "items": [
       {
         "product_sku_id": 1,
         "quantity": 100,
         "expire_time": "2026-12-31",
         "shelf_position": "A-01-03",
         "supplier_product_sku_sn_id": 10
       }
     ]
   }

2. 创建入库记录
   POST /warehouse/warehouse-ios/create
   Body: {
     "warehouse_id": 1,
     "type": 0,
     "order_id": <supplier_order_id>,
     "product_sku_sn_ids": [...]
   }
```

入库后系统自动创建或增加 ProductStock 库存。

---

## 出库操作（type=1）

关联销售订单。通常在销售订单审批通过后执行。

```
1. 构建出库快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-io
   Body: {
     "type": 1,
     "items": [
       {
         "product_sku_id": 1,
         "quantity": 50,
         "product_stock_id": 10
       }
     ]
   }

2. 创建出库记录
   POST /warehouse/warehouse-ios/create
   Body: {
     "warehouse_id": 1,
     "type": 1,
     "order_id": <sale_order_id>,
     "product_sku_sn_ids": [...]
   }
```

出库后系统自动扣减 ProductStock 库存。

---

## 退库操作（type=2）

关联销售退货。退货商品重新入库。

```
1. 构建退库快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-io
   Body: {
     "type": 2,
     "items": [
       { "product_sku_id": 1, "quantity": 10 }
     ]
   }

2. 创建退库记录
   POST /warehouse/warehouse-ios/create
   Body: {
     "warehouse_id": 1,
     "type": 2,
     "sale_return_id": <sale_return_id>,
     "product_sku_sn_ids": [...]
   }
```

退库后系统自动增加 ProductStock 库存。

---

## 盘点（WarehouseCheck）

对仓库中的实际库存进行核查，更新系统库存记录。

### 接口

| 操作 | 方法 | 路径                                      |
| ---- | ---- | ----------------------------------------- |
| 列表 | POST | `/warehouse/warehouse-checks/list`        |
| 创建 | POST | `/warehouse/warehouse-checks/create`      |
| 更新 | POST | `/warehouse/warehouse-checks/{id}/update` |

### 盘点流程

```
1. 构建盘点快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-check
   Body: {
     "items": [
       {
         "product_sku_id": 1,
         "actual_quantity": 95,
         "shelf_position": "A-01-03",
         "product_stock_id": 10
       }
     ]
   }

2. 创建盘点记录
   POST /warehouse/warehouse-checks/create
   Body: {
     "warehouse_id": 1,
     "product_sku_sn_ids": [...],
     "status": 0
   }

3. 完成盘点（库存自动调整）
   POST /warehouse/warehouse-checks/{id}/update
   Body: {
     "status": 1,
     "check_time": "2025-03-01T10:00:00"
   }
```

完成盘点后，系统自动调整库存至 `actual_quantity`。

---

## 移库（WarehouseTransfer）

将商品从一个仓库转移到另一个仓库。

### 接口

| 操作 | 方法 | 路径                                         |
| ---- | ---- | -------------------------------------------- |
| 列表 | POST | `/warehouse/warehouse-transfers/list`        |
| 创建 | POST | `/warehouse/warehouse-transfers/create`      |
| 详情 | POST | `/warehouse/warehouse-transfers/{id}/detail` |

### 移库流程

```
1. 构建移库快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-transfer
   Body: {
     "items": [
       {
         "product_sku_id": 1,
         "product_stock_id": 10,
         "quantity": 30,
         "shelf_position": "B-02-01"
       }
     ]
   }

2. 创建移库记录
   POST /warehouse/warehouse-transfers/create
   Body: {
     "from_warehouse_id": 1,
     "to_warehouse_id": 2,
     "product_sku_sn_ids": [...],
     "remark": "调拨至分仓"
   }
```

移库完成后，系统自动从源仓库扣减库存，在目标仓库增加库存。

---

## 库存查看

### 当前库存

```
POST /product/product-stocks/list
Body: { "warehouse_id": 1 }
```

### 含销售占用的库存

```
POST /product/product-stocks/list-with-sale-stock
```

### 库存变动记录

```
POST /product/product-stock-records/list
Body: { "product_sku_id": 1 }
```

记录中包含变动类型（type: 0 入库 / 1 出库 / 2 退库）、变动前后库存、关联的订单/退货/出入库/盘点/移库 ID。
