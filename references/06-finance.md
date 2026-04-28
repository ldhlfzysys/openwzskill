# 财务模块

管理财务类型、供应链发票（应付）、销售发票（应收）和税票。

---

## 财务类型（FinanceType）

用于分类发票的支付/收款方式（如银行转账、现金、支票等）。

### 接口

| 操作 | 方法 | 路径                                      |
| ---- | ---- | ----------------------------------------- |
| 列表 | POST | `/finance_type/finance-types/list`        |
| 详情 | GET  | `/finance_type/finance-types/{id}`        |
| 创建 | POST | `/finance_type/finance-types/create`      |
| 更新 | POST | `/finance_type/finance-types/{id}/update` |
| 删除 | POST | `/finance_type/finance-types/{id}/delete` |

---

## 供应链发票（应付 — SupplierInvoice）

记录向供应商的每笔付款。一个供应链订单可以有多笔发票（分期付款）。

### 接口

| 操作 | 方法 | 路径                                      |
| ---- | ---- | ----------------------------------------- |
| 列表 | POST | `/supplier/supplier-invoices/list`        |
| 创建 | POST | `/supplier/supplier-invoices/create`      |
| 更新 | POST | `/supplier/supplier-invoices/{id}/update` |
| 删除 | POST | `/supplier/supplier-invoices/{id}/delete` |

### 创建

```json
{
  "supplier_order_id": 1,
  "price": 5000.0,
  "finance_type_id": 1,
  "remark": "首期付款 50%",
  "pay_time": "2025-03-01"
}
```

### 应付计算

应付总额 = 供应链订单中所有 SKU 的 (数量 × 单价) 之和已付总额 = 该订单所有发票的 price 之和待付余额 = 应付总额 − 已付总额

---

## 销售发票（应收 — SaleInvoice）

记录从商户收到的每笔付款。一个销售订单可以有多笔发票。

### 接口

| 操作 | 方法 | 路径                              |
| ---- | ---- | --------------------------------- |
| 列表 | POST | `/sale/sale-invoices/list`        |
| 创建 | POST | `/sale/sale-invoices/create`      |
| 更新 | POST | `/sale/sale-invoices/{id}/update` |
| 删除 | POST | `/sale/sale-invoices/{id}/delete` |

### 创建

```json
{
  "sale_order_id": 1,
  "price": 8000.0,
  "finance_type_id": 1,
  "remark": "全额收款",
  "pay_time": "2025-03-15"
}
```

### 应收计算

应收总额 = 销售订单的 total 字段已收总额 = 该订单所有发票的 price 之和待收余额 = 应收总额 − 已收总额

---

## 税票（TaxInvoice）

按订单维度记录税票信息。

### 接口

| 操作             | 方法     | 路径                                           |
| ---------------- | -------- | ---------------------------------------------- |
| 列表             | POST     | `/finance/tax-invoices/list`                   |
| 创建             | POST     | `/finance/tax-invoices/create`                 |
| 更新             | POST     | `/finance/tax-invoices/{id}/update`            |
| 删除             | DELETE   | `/finance/tax-invoices/{id}`                   |
| 按供应链订单汇总 | POST/GET | `/finance/tax-invoices/sum-by-supplier-orders` |
| 按销售订单汇总   | POST/GET | `/finance/tax-invoices/sum-by-sale-orders`     |

### 类型

| type | 含义               | 关联字段            |
| ---- | ------------------ | ------------------- |
| 0    | 供应链税票（应付） | `supplier_order_id` |
| 1    | 销售税票（应收）   | `sale_order_id`     |

### 创建

```json
{
  "type": 0,
  "supplier_order_id": 1,
  "tax_total": 1500.0,
  "remark": "进口关税"
}
```

---

## 财务看板

### 接口

| 操作 | 方法 | 路径 | 说明 |
| --- | --- | --- | --- |
| 未结算概览 | GET | `/finance/dashboard/unsettled` | 未付/未收的供应链和销售订单汇总 |
| 已结算概览 | GET | `/finance/dashboard/settled` | 已付/已收的供应链和销售订单汇总 |
| 税务概览 | GET | `/finance/dashboard/tax` | 税票汇总数据 |

---

## 需要解析的 ID

| 字段                | 查询接口                                |
| ------------------- | --------------------------------------- |
| `finance_type_id`   | `POST /finance_type/finance-types/list` |
| `supplier_order_id` | `GET /supplier/supplier-orders/{id}`    |
| `sale_order_id`     | `GET /sale/sale-orders/{id}`            |
