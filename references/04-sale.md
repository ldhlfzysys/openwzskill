# 销售模块

管理商户、销售订单的完整生命周期（创建 → 审批 → 出库 → 结算），以及退货流程。

---

## 商户（Store）

### 接口

| 操作 | 方法 | 路径                       |
| ---- | ---- | -------------------------- |
| 列表 | POST | `/sale/stores/list`        |
| 创建 | POST | `/sale/stores/create`      |
| 详情 | GET  | `/sale/stores/{id}`        |
| 更新 | POST | `/sale/stores/{id}/update` |
| 删除 | POST | `/sale/stores/{id}/delete` |

### 商户地址（StoreAddress）

| 操作 | 方法   | 路径                                            |
| ---- | ------ | ----------------------------------------------- |
| 列表 | POST   | `/sale/store-addresses/list`（需传 `store_id`） |
| 创建 | POST   | `/sale/store-addresses/create`                  |
| 更新 | POST   | `/sale/store-addresses/{id}/update`             |
| 删除 | DELETE | `/sale/store-addresses/{id}`                    |

### 商户 SKU 编码映射

| 操作     | 方法 | 路径                                     |
| -------- | ---- | ---------------------------------------- |
| 列表     | POST | `/sale/store-sku-code-maps/list`         |
| 批量保存 | POST | `/sale/store-sku-code-maps/batch-save`   |
| 批量删除 | POST | `/sale/store-sku-code-maps/batch-delete` |

### 省份列表（用于商户表单）

```
POST /sale/provinces/list
Body: { "page": 1, "page_size": 500 }
```

### 需要解析的 ID

- `province_id` → 省份名称（影响税率计算）
- 商户的 `level` 影响售价/折扣匹配优先级

---

## 销售订单（SaleOrder）

### 接口

| 操作     | 方法 | 路径                                        |
| -------- | ---- | ------------------------------------------- |
| 缓存同步 | GET  | `/sale/sale-orders/cache-sync`              |
| 创建     | POST | `/sale/sale-orders/create`                  |
| 详情     | GET  | `/sale/sale-orders/{id}`                    |
| 更新     | POST | `/sale/sale-orders/{id}/update`             |
| 删除     | POST | `/sale/sale-orders/{id}/delete`             |
| 结算预览 | GET  | `/sale/sale-orders/{id}/settlement/preview` |
| 结算     | POST | `/sale/sale-orders/{id}/settle`             |

### 列表筛选

- `status`：支持数组
- `store_id`：按商户筛选

### 需要解析的 ID

- `store_id` → 商户名称
- `product_sku_sn_ids` → 商品快照详情

---

## 完整业务流程

### 第一步：创建订单

```
1. 选择商户（或先创建商户）
2. 选择有库存的 SKU，填写数量、售价、折扣
   - 查询可用库存：POST /product/product-stocks/list-with-sale-stock
   - 可用库存 = stock - block_stock
3. 构建销售快照
   POST /product_sku_sn/product-sku-sns/build-for-sale-order
   Body: {
     "store_id": 1,
     "items": [
       {
         "product_sku_id": 1,
         "quantity": 50,
         "warehouse_id": 1,
         "product_stock_id": 10,
         "sale_price": 12.0,
         "sale_discount_price": 1.0
       }
     ]
   }
   → 返回 { "product_sku_sn_ids": [20, 21] }

4. 创建订单
   POST /sale/sale-orders/create
   Body: {
     "store_id": 1,
     "product_sku_sn_ids": [20, 21],
     "status": 0
   }
```

**重要**：创建销售订单时，系统会自动占用（冻结）对应的库存（block_stock 增加）。

### 第二步：审批流

```
提交审批：POST /sale/sale-orders/{id}/update  { "status": 1 }
审批通过：POST /sale/sale-orders/{id}/update  { "status": 2 }
驳回：    POST /sale/sale-orders/{id}/update  { "status": 0 }
```

### 第三步：出库

审批通过后（status=2），创建出库记录：

```
1. 构建出库快照
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-io
   Body: { "type": 1, "items": [...] }

2. 创建出库记录
   POST /warehouse/warehouse-ios/create
   Body: {
     "warehouse_id": 1,
     "type": 1,
     "order_id": <sale_order_id>,
     "product_sku_sn_ids": [...]
   }

3. 更新订单状态
   POST /sale/sale-orders/{id}/update  { "status": 3 }
```

出库完成后，系统自动扣减库存。

### 第四步：配送与结算

```
标记配送中：POST /sale/sale-orders/{id}/update  { "status": 4 }

结算预览（查看年化收益、周转率等）：
GET /sale/sale-orders/{id}/settlement/preview

执行结算：
POST /sale/sale-orders/{id}/settle  {}
→ 订单状态变为 5（已结算），生成 SaleAnalysis 分析数据
```

**结算前置条件**：

- 订单 status >= 4（已出库或配送中）
- 关联的供应链订单已结算（需要供应成本数据来计算利润）

---

## 销售退货（SaleReturn）

### 接口

| 操作 | 方法 | 路径                             |
| ---- | ---- | -------------------------------- |
| 列表 | POST | `/sale/sale-returns/list`        |
| 创建 | POST | `/sale/sale-returns/create`      |
| 详情 | GET  | `/sale/sale-returns/{id}`        |
| 更新 | POST | `/sale/sale-returns/{id}/update` |

### 退货流程

```
1. 计算可退数量
   可退数量 = 订单中该 SKU 数量 − 该订单所有未取消退货单中该 SKU 的已退数量

2. 构建退货快照
   POST /product_sku_sn/product-sku-sns/build-for-sale-return
   Body: {
     "sale_order_id": 1,
     "items": [
       { "product_sku_sn_id": 20, "quantity": 10 }
     ]
   }

3. 创建退货单
   POST /sale/sale-returns/create
   Body: {
     "sale_order_id": 1,
     "product_sku_sn_ids": [...],
     "remark": "质量问题"
   }

4. 创建退库记录（type=2）
   POST /product_sku_sn/product-sku-sns/build-for-warehouse-io
   Body: { "type": 2, "items": [...] }

   POST /warehouse/warehouse-ios/create
   Body: {
     "warehouse_id": 1,
     "type": 2,
     "sale_return_id": <sale_return_id>,
     "product_sku_sn_ids": [...]
   }

5. 更新退货状态为已退库
   POST /sale/sale-returns/{id}/update  { "status": 1 }
```

---

## 销售发票（SaleInvoice）

登记从商户收到的付款记录。

### 接口

| 操作 | 方法 | 路径                              |
| ---- | ---- | --------------------------------- |
| 列表 | POST | `/sale/sale-invoices/list`        |
| 创建 | POST | `/sale/sale-invoices/create`      |
| 更新 | POST | `/sale/sale-invoices/{id}/update` |
| 删除 | POST | `/sale/sale-invoices/{id}/delete` |

### 创建发票

```json
{
  "sale_order_id": 1,
  "price": 8000.0,
  "finance_type_id": 1,
  "remark": "全额付款",
  "pay_time": "2025-03-15"
}
```
