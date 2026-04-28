# 供应链模块

管理供应商、采购订单的完整生命周期（创建 → 审批 → 发货 → 入库 → 结算）。

---

## 供应商（Supplier）

### 接口

| 操作 | 方法   | 路径                              |
| ---- | ------ | --------------------------------- |
| 列表 | POST   | `/supplier/suppliers/list`        |
| 创建 | POST   | `/supplier/suppliers/create`      |
| 详情 | GET    | `/supplier/suppliers/{id}`        |
| 更新 | POST   | `/supplier/suppliers/{id}/update` |
| 删除 | DELETE | `/supplier/suppliers/{id}`        |

### 供应商 SKU 编码映射

供应商可能使用自己的商品编码。系统支持维护供应商编码到系统 SKU 的映射。

| 操作     | 方法 | 路径                                            |
| -------- | ---- | ----------------------------------------------- |
| 列表     | POST | `/supplier/supplier-sku-code-maps/list`         |
| 批量保存 | POST | `/supplier/supplier-sku-code-maps/batch-save`   |
| 批量删除 | POST | `/supplier/supplier-sku-code-maps/batch-delete` |

---

## 供应链订单（SupplierOrder）

### 接口

| 操作     | 方法 | 路径                                    |
| -------- | ---- | --------------------------------------- |
| 缓存同步 | GET  | `/supplier/supplier-orders/cache-sync`  |
| 创建     | POST | `/supplier/supplier-orders/create`      |
| 详情     | GET  | `/supplier/supplier-orders/{id}`        |
| 更新     | POST | `/supplier/supplier-orders/{id}/update` |
| 删除     | POST | `/supplier/supplier-orders/{id}/delete` |
| 结算     | POST | `/supplier/supplier-orders/{id}/settle` |

### 列表筛选

- `status`：支持传数组，如 `[0, 1]` 查询多个状态
- `supplier_id`：按供应商筛选

### 需要解析的 ID

- `supplier_id` → 供应商名称
- `product_sku_sn_ids` → 商品快照详情（调用 list-by-ids）

---

## 完整业务流程

### 第一步：创建订单

```
1. 选择供应商（或先创建供应商）
2. 选择 SKU，填写数量和采购价
3. 调用 build-for-supplier-order 生成快照
   POST /product_sku_sn/product-sku-sns/build-for-supplier-order
   Body: {
     "items": [
       { "product_sku_id": 1, "quantity": 100, "supplier_price": 5.5 },
       { "product_sku_id": 2, "quantity": 200, "supplier_price": 3.0 }
     ]
   }
   → 返回 { "product_sku_sn_ids": [10, 11] }

4. 创建订单
   POST /supplier/supplier-orders/create
   Body: {
     "supplier_id": 1,
     "product_sku_sn_ids": [10, 11],
     "status": 0
   }
```

### 第二步：审批流

```
提交审批：POST /supplier/supplier-orders/{id}/update  { "status": 1 }
审批通过：POST /supplier/supplier-orders/{id}/update  { "status": 2 }
驳回：    POST /supplier/supplier-orders/{id}/update  { "status": 0 }
```

### 第三步：物流跟踪

```
已下单：  { "status": 3, "order_time": "2025-01-01" }
已发货：  { "status": 4, "ship_time": "..." }
已到港：  { "status": 5, "port_arrival_time": "..." }
清关中：  { "status": 6, "customs_clearance_time": "..." }
转运中：  { "status": 7, "transit_time": "..." }
```

### 第四步：入库

当订单到达仓库（status=7→8），需要创建入库记录：

```
1. 构建入库快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-io
   Body: {
     "type": 0,
     "items": [
       { "product_sku_id": 1, "quantity": 100, "expire_time": "2026-12-31" }
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

3. 更新订单状态
   POST /supplier/supplier-orders/{id}/update
   Body: { "status": 8, "warehouse_time": "..." }
```

入库完成后，系统会自动创建或增加 ProductStock 库存记录。

### 第五步：结算

只有 **status=8（已入库）** 的订单才能结算。

```
POST /supplier/supplier-orders/{id}/settle
Body: {
  "volume_m3": 15.5,
  "sku_volume_updates": [
    { "product_sku_id": 1, "size_value": 0.05 },
    { "product_sku_id": 2, "size_value": 0.03 }
  ]
}
```

结算会：

- 按体积（volume_m3）将运费分摊到每个 SKU 的库存成本字段（supplier_shipping_cost）
- 更新 SKU 的 size_value（单位体积）
- 订单状态变为 9（已结算）

---

## 供应链发票（SupplierInvoice）

登记向供应商的付款记录。

### 接口

| 操作 | 方法 | 路径                                      |
| ---- | ---- | ----------------------------------------- |
| 列表 | POST | `/supplier/supplier-invoices/list`        |
| 创建 | POST | `/supplier/supplier-invoices/create`      |
| 更新 | POST | `/supplier/supplier-invoices/{id}/update` |
| 删除 | POST | `/supplier/supplier-invoices/{id}/delete` |

### 创建发票

```json
{
  "supplier_order_id": 1,
  "price": 5000.0,
  "finance_type_id": 1,
  "remark": "首期付款",
  "pay_time": "2025-03-01"
}
```

### 需要解析的 ID

- `finance_type_id` → 财务类型名称（从 `/finance_type/finance-types/list` 获取）
