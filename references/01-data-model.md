# 数据模型

所有实体的字段定义与关系。按领域分组，公共字段 `id`、`create_time`、`update_time` 省略。

---

## 用户与权限

### User

| 字段            | 类型   | 说明     |
| --------------- | ------ | -------- |
| account         | string | 登录账号 |
| hashed_password | string | 密码哈希 |
| user_type       | int    | 用户类型 |
| role_id         | int    | 关联角色 |
| name            | string | 姓名     |
| avatar          | string | 头像     |
| mobile          | string | 手机号   |
| email           | string | 邮箱     |
| mark            | string | 备注     |

### Role

| 字段  | 类型   | 说明                   |
| ----- | ------ | ---------------------- |
| name  | string | 角色名称               |
| hide  | bool   | 是否隐藏               |
| auths | Auth[] | 关联权限列表（多对多） |

### Auth

| 字段 | 类型   | 说明     |
| ---- | ------ | -------- |
| name | string | 权限名称 |
| code | string | 权限编码 |

---

## 商品与分类

### Category（分类）

| 字段      | 类型     | 说明                  |
| --------- | -------- | --------------------- |
| name      | string   | 分类名称              |
| parent_id | int/null | 父分类 ID（树形结构） |
| level     | int      | 层级                  |
| order_num | int      | 排序                  |
| status    | int      | 状态                  |
| hide      | bool     | 是否隐藏              |

### Brand（品牌）

| 字段 | 类型   | 说明      |
| ---- | ------ | --------- |
| name | string | 品牌名称  |
| logo | string | Logo 图片 |
| hide | bool   | 是否隐藏  |

### AnnualTarget（品牌年度目标）

| 字段                  | 类型  | 说明         |
| --------------------- | ----- | ------------ |
| brand_id              | int   | 品牌 ID      |
| year                  | int   | 年份         |
| annual_target         | float | 年度目标金额 |
| q1_target ~ q4_target | float | 季度目标     |

### Product（商品/SPU）

| 字段                  | 类型   | 说明                  |
| --------------------- | ------ | --------------------- |
| code                  | string | 商品编码              |
| name                  | string | 商品名称（中文）      |
| name_tw               | string | 繁体名称              |
| name_en               | string | 英文名称              |
| brand_id              | int    | 品牌 ID               |
| desc                  | string | 描述                  |
| status                | int    | 状态                  |
| images                | string | 图片                  |
| custom_code           | string | 自定义编码            |
| country_code          | string | 产地国家编码          |
| custom_category_1/2/3 | string | 自定义分类            |
| hide                  | bool   | 是否隐藏              |
| category_ids          | int[]  | 关联分类 ID（多对多） |

### ProductSku（SKU）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| product_id | int | 所属商品 ID |
| brand_id | int | 品牌 ID |
| sku_code | string | SKU 编码 |
| spec_name | string | 规格名称 |
| spec_name_tw / spec_name_en | string | 规格名称（繁体/英文） |
| barcodes | string | 条形码 |
| unit | string | 单位 |
| status | int | 状态 |
| images | string | 图片 |
| desc | string | 描述 |
| size_desc | string | 尺寸描述 |
| size_value | float | 尺寸数值（体积 m³） |
| has_tax | bool | 是否含税 |
| hide | bool | 是否隐藏 |
| product | Product | 关联商品对象（接口返回时附带） |
| stocks | ProductStock[] | 库存列表（接口返回时附带） |

### ProductStock（库存）

唯一约束：(product_sku_id, warehouse_id, expire_time, supplier_order_id)

| 字段                    | 类型     | 说明                       |
| ----------------------- | -------- | -------------------------- |
| product_sku_id          | int      | SKU ID                     |
| warehouse_id            | int      | 仓库 ID                    |
| expire_time             | datetime | 过期时间                   |
| supplier_order_id       | int      | 来源供应链订单 ID          |
| supplier_price          | float    | 供应商成本价               |
| supplier_discount_price | float    | 供应商折扣价               |
| supplier_shipping_cost  | float    | 分摊运费                   |
| shelf_position          | string   | 货架位置                   |
| stock                   | int      | 当前库存                   |
| block_stock             | int      | 冻结库存（被销售订单占用） |
| block_user_id           | int      | 冻结操作用户               |
| pre_stock               | int      | 预入库数量                 |

### ProductStockRecord（库存变动记录）

| 字段                  | 类型   | 说明                   |
| --------------------- | ------ | ---------------------- |
| product_sku_id        | int    | SKU ID                 |
| warehouse_id          | int    | 仓库 ID                |
| product_stock_id      | int    | 库存记录 ID            |
| type                  | int    | 0=入库, 1=出库, 2=退库 |
| quantity              | int    | 变动数量               |
| stock_before          | int    | 变动前库存             |
| stock_after           | int    | 变动后库存             |
| product_sku_sn_id     | int    | 关联快照 ID            |
| supplier_order_id     | int    | 关联供应链订单         |
| sale_order_id         | int    | 关联销售订单           |
| sale_return_id        | int    | 关联退货单             |
| warehouse_io_id       | int    | 关联出入库记录         |
| warehouse_check_id    | int    | 关联盘点记录           |
| warehouse_transfer_id | int    | 关联移库记录           |
| remark                | string | 备注                   |

### ProductSaleStock（销售占用库存）

| 字段             | 类型 | 说明        |
| ---------------- | ---- | ----------- |
| sale_order_id    | int  | 销售订单 ID |
| product_stock_id | int  | 库存记录 ID |
| product_sku_id   | int  | SKU ID      |
| warehouse_id     | int  | 仓库 ID     |
| quantity         | int  | 占用数量    |

### ProductSkuSalePrice（SKU 售价）

| 字段           | 类型  | 说明                              |
| -------------- | ----- | --------------------------------- |
| product_sku_id | int   | SKU ID                            |
| store_id       | int   | 商户 ID（为某商户设置的专属价格） |
| level          | int   | 等级（多级价格体系）              |
| price          | float | 售价                              |

### ProductSkuSaleDiscount（SKU 折扣）

| 字段           | 类型   | 说明       |
| -------------- | ------ | ---------- |
| product_sku_id | int    | SKU ID     |
| store_id       | int    | 商户 ID    |
| level          | int    | 等级       |
| remark         | string | 折扣说明   |
| discount       | float  | 折扣金额   |
| custom         | string | 自定义信息 |

### ProductSkuSn（商品快照）

订单/出入库等操作的商品冻结快照。字段以 `sn_` 为前缀，记录下单时刻的商品信息。

| 字段                   | 类型  | 说明                         |
| ---------------------- | ----- | ---------------------------- |
| type                   | int   | 快照类型（区分来源场景）     |
| sn_product_id          | int   | 快照中的商品 ID              |
| sn_product_sku_id      | int   | 快照中的 SKU ID              |
| sn_quantity            | int   | 数量                         |
| sn_supplier_price      | float | 供应商单价                   |
| sn_sale_price          | float | 销售单价                     |
| sn_sale_discount_price | float | 销售折扣金额                 |
| ...                    | ...   | 其他冻结字段（名称、规格等） |

---

## 供应链

### Supplier（供应商）

| 字段         | 类型   | 说明       |
| ------------ | ------ | ---------- |
| name         | string | 供应商名称 |
| country      | string | 国家       |
| email        | string | 邮箱       |
| phone        | string | 电话       |
| payment_days | int    | 账期天数   |
| remark       | string | 备注       |
| hide         | bool   | 是否隐藏   |

### SupplierOrder（供应链订单）

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| supplier_id | int | 供应商 ID |
| product_sku_sn_ids | int[] | 商品快照 ID 列表 |
| status | int | 状态（0~9，详见 00-core-concepts.md） |
| order_time | datetime | 下单时间 |
| ship_time | datetime | 发货时间 |
| port_arrival_time | datetime | 到港时间 |
| customs_clearance_time | datetime | 清关时间 |
| transit_time | datetime | 转运时间 |
| warehouse_time | datetime | 入库时间 |
| container | string | 集装箱号 |
| size | string | 集装箱尺寸 |
| cases | int | 箱数 |
| customs\_\* | string/float | 报关相关字段 |
| pod | string | 卸货港 |
| eta\_\* | datetime | 预计到达时间 |
| delivery\_\* | string | 配送信息 |
| freight\_\* | float | 运费相关 |

### SupplierInvoice（供应链发票）

| 字段              | 类型     | 说明        |
| ----------------- | -------- | ----------- |
| supplier_order_id | int      | 关联订单 ID |
| price             | float    | 金额        |
| finance_type_id   | int      | 财务类型 ID |
| remark            | string   | 备注        |
| invoice_file      | string   | 发票文件    |
| log               | string   | 操作日志    |
| pay_time          | datetime | 付款时间    |

### SupplierSkuCodeMap（供应商 SKU 编码映射）

| 字段           | 类型   | 说明                 |
| -------------- | ------ | -------------------- |
| supplier_id    | int    | 供应商 ID            |
| code           | string | 供应商自己的商品编码 |
| product_sku_id | int    | 系统 SKU ID          |

---

## 销售

### Province（省份/税区）

| 字段                          | 类型   | 说明     |
| ----------------------------- | ------ | -------- |
| name                          | string | 省份名称 |
| code                          | string | 省份编码 |
| type                          | int    | 类型     |
| GST / PST / QST / HST / Total | float  | 各项税率 |
| hide                          | bool   | 是否隐藏 |

### Store（商户）

| 字段         | 类型   | 说明                      |
| ------------ | ------ | ------------------------- |
| code         | string | 商户编码                  |
| name         | string | 商户名称                  |
| contact\_\*  | string | 联系人信息                |
| billing\_\*  | string | 账单信息                  |
| province_id  | int    | 省份 ID（决定税率）       |
| level        | int    | 商户等级（影响售价/折扣） |
| credit_limit | float  | 信用额度                  |
| payment_days | int    | 账期天数                  |
| current_debt | float  | 当前欠款                  |
| status       | int    | 状态                      |
| remark       | string | 备注                      |
| hide         | bool   | 是否隐藏                  |

### StoreAddress（商户地址）

| 字段          | 类型   | 说明     |
| ------------- | ------ | -------- |
| store_id      | int    | 商户 ID  |
| name          | string | 地址名称 |
| detail        | string | 详细地址 |
| contact_name  | string | 联系人   |
| contact_phone | string | 联系电话 |
| remark        | string | 备注     |
| status        | int    | 状态     |
| hide          | bool   | 是否隐藏 |

### StoreSkuCodeMap（商户 SKU 编码映射）

| 字段           | 类型   | 说明               |
| -------------- | ------ | ------------------ |
| store_id       | int    | 商户 ID            |
| code           | string | 商户自己的商品编码 |
| product_sku_id | int    | 系统 SKU ID        |

### SaleOrder（销售订单）

| 字段               | 类型     | 说明                                  |
| ------------------ | -------- | ------------------------------------- |
| store_id           | int      | 商户 ID                               |
| product_sku_sn_ids | int[]    | 商品快照 ID 列表                      |
| status             | int      | 状态（0~6，详见 00-core-concepts.md） |
| contract_file      | string   | 合同文件                              |
| remark             | string   | 备注                                  |
| approver_id        | int      | 审批人 ID                             |
| approver_remark    | string   | 审批备注                              |
| approver_time      | datetime | 审批时间                              |
| log                | string   | 操作日志                              |
| is_batch_delivery  | bool     | 是否分批配送                          |
| product_total      | float    | 商品总额                              |
| discount           | float    | 折扣总额                              |
| tax_total          | float    | 税费总额                              |
| total              | float    | 订单总额                              |
| delivery\_\*       | string   | 配送信息                              |
| hide               | bool     | 是否隐藏                              |

### SaleReturn（销售退货）

| 字段               | 类型   | 说明                         |
| ------------------ | ------ | ---------------------------- |
| sale_order_id      | int    | 关联销售订单 ID              |
| product_sku_sn_ids | int[]  | 退货商品快照 ID 列表         |
| status             | int    | 0=退货中, 1=已退库, 2=已取消 |
| remark             | string | 备注                         |

### SaleInvoice（销售发票）

| 字段            | 类型     | 说明            |
| --------------- | -------- | --------------- |
| sale_order_id   | int      | 关联销售订单 ID |
| price           | float    | 金额            |
| finance_type_id | int      | 财务类型 ID     |
| remark          | string   | 备注            |
| invoice_file    | string   | 发票文件        |
| log             | string   | 操作日志        |
| pay_time        | datetime | 收款时间        |

---

## 仓库

### Warehouse（仓库）

| 字段                         | 类型   | 说明                    |
| ---------------------------- | ------ | ----------------------- |
| name                         | string | 仓库名称                |
| address                      | string | 仓库地址                |
| phone                        | string | 电话                    |
| storage_cost_per_cbm_per_day | float  | 仓储日费（每立方米/天） |
| hide                         | bool   | 是否隐藏                |

### WarehouseIO（出入库记录）

| 字段               | 类型   | 说明                                          |
| ------------------ | ------ | --------------------------------------------- |
| warehouse_id       | int    | 仓库 ID                                       |
| type               | int    | 0=入库, 1=出库, 2=退库                        |
| order_id           | int    | 关联订单 ID（入库→供应链订单, 出库→销售订单） |
| sale_return_id     | int    | 关联退货 ID（退库时）                         |
| product_sku_sn_ids | int[]  | 商品快照 ID 列表                              |
| product_diff       | string | 差异说明                                      |
| remark             | string | 备注                                          |
| images             | string | 图片                                          |
| hide               | bool   | 是否隐藏                                      |

### WarehouseCheck（盘点）

| 字段               | 类型     | 说明             |
| ------------------ | -------- | ---------------- |
| warehouse_id       | int      | 仓库 ID          |
| user_id            | int      | 操作用户 ID      |
| product_sku_sn_ids | int[]    | 盘点快照 ID 列表 |
| status             | int      | 0=草稿, 1=已完成 |
| check_time         | datetime | 盘点时间         |

### WarehouseTransfer（移库）

| 字段                     | 类型   | 说明               |
| ------------------------ | ------ | ------------------ |
| from_warehouse_id        | int    | 源仓库 ID          |
| to_warehouse_id          | int    | 目标仓库 ID        |
| user_id                  | int    | 操作用户 ID        |
| product_sku_sn_ids       | int[]  | 商品快照 ID 列表   |
| source_product_stock_ids | int[]  | 源库存记录 ID 列表 |
| remark                   | string | 备注               |

---

## 财务

### FinanceType（财务类型）

| 字段 | 类型   | 说明     |
| ---- | ------ | -------- |
| name | string | 类型名称 |
| hide | bool   | 是否隐藏 |

### TaxInvoice（税票）

| 字段              | 类型   | 说明                             |
| ----------------- | ------ | -------------------------------- |
| type              | int    | 0=供应链（应付）, 1=销售（应收） |
| supplier_order_id | int    | 关联供应链订单（type=0 时）      |
| sale_order_id     | int    | 关联销售订单（type=1 时）        |
| tax_total         | float  | 税额                             |
| remark            | string | 备注                             |
| invoice_file      | string | 发票文件                         |
| log               | string | 操作日志                         |

---

## 实体关系概览

```
User → Role → Auth (多对多)

Category (树形自引用) ←→ Product (多对多)
Brand → Product → ProductSku → ProductStock
                              → ProductSkuSalePrice
                              → ProductSkuSaleDiscount
                              → ProductSkuSn (快照)

ProductStock → Warehouse + SupplierOrder
ProductStock → ProductStockRecord
ProductStock → ProductSaleStock → SaleOrder

Supplier → SupplierOrder → SupplierInvoice
                         → ProductSkuSn
                         → WarehouseIO (入库)
                         → TaxInvoice (type=0)

Store → Province
Store → SaleOrder → SaleInvoice
                  → SaleReturn → WarehouseIO (退库)
                  → ProductSaleStock
                  → TaxInvoice (type=1)
                  → WarehouseIO (出库)

Warehouse → WarehouseIO
          → WarehouseCheck
          → WarehouseTransfer

FinanceType → SupplierInvoice / SaleInvoice
```
