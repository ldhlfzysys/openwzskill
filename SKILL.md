---
name: wuzhou-saas-api
description: |
  五洲 SaaS 后端 API 与业务逻辑技能。
  供 Agent 直接调用五洲 SaaS HTTP 接口，执行商品、供应链、销售、仓库、财务等完整业务流程。
  所有操作通过 API 完成，不涉及浏览器或前端页面。
---

# 五洲 SaaS API 技能

## 系统简介

五洲 SaaS 是一套面向国际贸易/进口商品的 B2B 管理系统，核心业务流程为：

**采购 → 入库 → 销售 → 出库 → 结算**

系统覆盖：商品管理、供应链采购、销售订单、仓库进出库/盘点/移库、财务发票/税票、数据看板。

## 鉴权

- **Base URL**：`https://wzapi.wholety.com`（可在 skill config 的 `base_url` 覆盖）
- **登录**：`POST /user/login`，Body `{ username, password }`，响应中取 `access_token`
- **请求头**：除登录/注册外，所有请求需带 `Authorization: Bearer <access_token>`

## API 约定

### 统一响应格式

```json
{
  "success": true,
  "data": "...(对象或数组)",
  "message": "...",
  "total_records": 100
}
```

列表接口 `data` 为数组，`total_records` 为总条数。

### 通用列表入参

```json
{
  "page": 1,
  "page_size": 20,
  "...其他筛选字段"
}
```

### 路由前缀

| 前缀              | 模块                                    |
| ----------------- | --------------------------------------- |
| `/user`           | 用户、登录、角色                        |
| `/system`         | 省份、角色、权限                        |
| `/product`        | 分类、品牌、商品、SKU、库存、价格、折扣 |
| `/product_sku_sn` | 商品快照构建                            |
| `/supplier`       | 供应商、供应链订单、供应链发票          |
| `/sale`           | 商户、销售订单、退货、销售发票          |
| `/warehouse`      | 仓库、出入库、盘点、移库                |
| `/finance_type`   | 财务类型                                |
| `/finance`        | 财务看板、税票                          |
| `/dashboard`      | 仪表盘概览                              |
| `/oss`            | 文件上传/下载                           |

### CRUD 接口命名规律

绝大多数实体遵循统一的接口命名模式：

| 操作 | 方法 | 路径模式 |
| --- | --- | --- |
| 列表 | POST | `/{prefix}/{entities}/list` |
| 创建 | POST | `/{prefix}/{entities}/create` |
| 详情 | GET | `/{prefix}/{entities}/{id}` |
| 更新 | POST | `/{prefix}/{entities}/{id}/update` |
| 删除 | POST/DELETE | `/{prefix}/{entities}/{id}/delete` 或 `/{prefix}/{entities}/{id}` |

## 参考文档（渐进式阅读）

按以下顺序阅读，逐步建立对系统的理解：

| 顺序 | 文档 | 内容 | 何时阅读 |
| --- | --- | --- | --- |
| 1 | [00-core-concepts.md](references/00-core-concepts.md) | 快照机制、ID→名称解析、状态驱动流程 | **必读**，理解系统运作的三个核心模式 |
| 2 | [01-data-model.md](references/01-data-model.md) | 所有实体、字段、关系 | 需要了解某个实体有哪些字段时查阅 |
| 3 | [02-product.md](references/02-product.md) | 分类、商品 SPU/SKU、库存、价格折扣 | 操作商品相关接口时 |
| 4 | [03-supply-chain.md](references/03-supply-chain.md) | 供应商、采购订单全生命周期、结算 | 操作供应链采购时 |
| 5 | [04-sale.md](references/04-sale.md) | 商户、销售订单、退货、结算 | 操作销售相关接口时 |
| 6 | [05-warehouse.md](references/05-warehouse.md) | 入库/出库/退库、盘点、移库 | 操作仓库相关接口时 |
| 7 | [06-finance.md](references/06-finance.md) | 发票、税票、财务类型 | 操作财务相关接口时 |
| 8 | [07-system-dashboard.md](references/07-system-dashboard.md) | 用户、角色、省份、仪表盘 | 操作系统配置或查看看板时 |

## 快速示例

### 登录获取 Token

```
POST /user/login
Body: { "username": "admin", "password": "xxx" }
→ response.data.token.access_token
```

### 查询商品列表并解析品牌名称

```
1. POST /product/products/list  { "page": 1, "page_size": 20 }
   → 获取商品列表，每个商品有 brand_id
2. POST /product/brands/list    { "page": 1, "page_size": 500 }
   → 获取品牌列表，通过 brand_id 匹配出品牌名称
```

### 创建供应链订单（含快照）

```
1. POST /product_sku_sn/product-sku-sns/build-for-supplier-order
   Body: { "items": [{ "product_sku_id": 1, "quantity": 100, "supplier_price": 5.5 }] }
   → 获取 product_sku_sn_ids
2. POST /supplier/supplier-orders/create
   Body: { "supplier_id": 1, "product_sku_sn_ids": [...], "status": 0 }
```
