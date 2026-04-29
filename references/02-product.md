# 商品模块

管理分类、商品（SPU）、SKU、库存、售价、折扣。

---

## 分类（Category）

树形结构，通过 `parent_id` 实现层级。

### 接口

| 操作 | 方法 | 路径                              |
| ---- | ---- | --------------------------------- |
| 列表 | POST | `/product/categories/list`        |
| 创建 | POST | `/product/categories/create`      |
| 更新 | POST | `/product/categories/{id}/update` |
| 删除 | POST | `/product/categories/{id}/delete` |

### 业务规则

- 删除分类前需确认无关联商品，否则接口会拒绝
- 层级通过 `parent_id` 和 `level` 控制

---

## 品牌（Brand）

### 接口

| 操作         | 方法 | 路径                                    |
| ------------ | ---- | --------------------------------------- |
| 列表         | POST | `/product/brands/list`                  |
| 创建         | POST | `/product/brands/create`                |
| 详情         | GET  | `/product/brands/{id}`                  |
| 更新         | POST | `/product/brands/{id}/update`           |
| 删除         | POST | `/product/brands/{id}/delete`           |
| 品牌看板总览 | POST | `/product/brands/dashboard/overview`    |
| 品牌看板详情 | GET  | `/product/brands/{id}/dashboard/detail` |

### 年度目标

| 操作 | 方法 | 路径                                  |
| ---- | ---- | ------------------------------------- |
| 列表 | POST | `/product/annual-targets/list`        |
| 创建 | POST | `/product/annual-targets/create`      |
| 更新 | POST | `/product/annual-targets/{id}/update` |
| 删除 | POST | `/product/annual-targets/{id}/delete` |

---

## 商品（Product / SPU）

一个商品（SPU）对应多个 SKU。

### 接口

| 操作 | 方法 | 路径 | 说明 |
| --- | --- | --- | --- |
| 列表 | POST | `/product/products/list` | 支持 name、category_ids、country_code 等筛选 |
| 缓存同步 | GET | `/product/products/cache-sync` | Query: `local_product_version`、`local_product_sku_version`；返回全量商品与 SKU（含 stocks、售价、折扣），前端以此增量同步 |

> ⚠️ **注意**：`cache-sync` 返回的 `stocks` 数组中**没有 `available_stock` 字段**，每个库存对象的字段为 `id, warehouse_id, expire_time, stock, block_stock, pre_stock, shelf_position`。可用库存须用 `stock - block_stock` 计算。`product-stocks/list` 单独接口才有 `available_stock` 字段，但需传 `product_sku_id` 才能查到数据。
| 批量获取 | POST | `/product/products/batch-get` | Body: `{ "ids": [1,2,3] }`，最多 100 个 |
| 创建 | POST | `/product/products/create` |  |
| 详情 | GET | `/product/products/{id}` |  |
| 更新 | POST | `/product/products/{id}/update` |  |
| 删除 | POST | `/product/products/{id}/delete` |  |

### 列表筛选字段

| 字段                  | 说明             |
| --------------------- | ---------------- |
| name                  | 商品名称模糊搜索 |
| category_ids          | 分类 ID 数组     |
| country_code          | 产地国家编码     |
| custom_code           | 自定义编码       |
| custom_category_1/2/3 | 自定义分类       |

### 需要解析的 ID

- `brand_id` → 品牌名称（从 `/product/brands/list` 获取）
- `category_ids` → 分类名称（从 `/product/categories/list` 获取）

---

## SKU

### 接口

| 操作     | 方法 | 路径                                |
| -------- | ---- | ----------------------------------- |
| 批量获取 | POST | `/product/product-skus/batch-get`   |
| 创建     | POST | `/product/product-skus/create`      |
| 更新     | POST | `/product/product-skus/{id}/update` |
| 删除     | POST | `/product/product-skus/{id}/delete` |

### 业务规则

- SKU 全量数据由 `/product/products/cache-sync` 下发（含 `product`、`stocks`、售价与折扣）；`batch-get` 按 id 补全单条或多条
- `size_value` 记录该 SKU 的单位体积（m³），供应链结算时用于按体积分摊运费

---

## 库存（ProductStock）

### 接口

| 操作 | 方法 | 路径 | 说明 |
| --- | --- | --- | --- |
| 列表 | POST | `/product/product-stocks/list` | **必须传 `product_sku_id` 才能查到数据**，可选传 `warehouse_id`、`supplier_order_id` 进一步过滤 |
| 列表（含销售占用） | POST | `/product/product-stocks/list-with-sale-stock` | 创建销售订单选品时使用 |
| 更新 | POST | `/product/product-stocks/{id}/update` |  |
| 盘点更新/新建 | POST | `/product/product-stocks/inventory-update` | 盘点场景专用 |

### 库存变动记录

| 操作 | 方法 | 路径                                  |
| ---- | ---- | ------------------------------------- |
| 列表 | POST | `/product/product-stock-records/list` |

### 销售占用库存

| 操作 | 方法 | 路径                                |
| ---- | ---- | ----------------------------------- |
| 列表 | POST | `/product/product-sale-stocks/list` |

### 业务规则

- 库存由 (product_sku_id, warehouse_id, expire_time, supplier_order_id) 唯一确定
- `stock` 为实际库存，`block_stock` 为被销售订单冻结的数量
- 可用库存 = stock - block_stock
- 入库时自动创建/增加库存，出库时自动减少，退库时自动增回
- 盘点通过 `inventory-update` 接口，可按 SKU + 仓库 + 过期日查找或新建库存记录

---

## 售价与折扣

### 售价（ProductSkuSalePrice）

| 操作 | 方法 | 路径                                           |
| ---- | ---- | ---------------------------------------------- |
| 创建 | POST | `/product/product-sku-sale-prices/create`      |
| 更新 | POST | `/product/product-sku-sale-prices/{id}/update` |
| 删除 | POST | `/product/product-sku-sale-prices/{id}/delete` |

### 折扣（ProductSkuSaleDiscount）

| 操作 | 方法 | 路径                                              |
| ---- | ---- | ------------------------------------------------- |
| 创建 | POST | `/product/product-sku-sale-discounts/create`      |
| 更新 | POST | `/product/product-sku-sale-discounts/{id}/update` |
| 删除 | POST | `/product/product-sku-sale-discounts/{id}/delete` |

### 售价/折扣优先级

售价和折扣可以按 `store_id`（商户）和 `level`（等级）维度设置：

1. 优先匹配 **store_id 相同** 的记录
2. 再匹配 **level 相同** 的记录
3. 创建销售订单时，前端按此优先级自动填充售价和折扣
