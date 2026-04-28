# 系统管理与仪表盘

用户管理、角色权限、省份配置、全局仪表盘和审批管理。

---

## 用户管理

### 接口

| 操作 | 方法 | 路径 | 说明 |
| --- | --- | --- | --- |
| 登录 | POST | `/user/login` | Body: `{ username, password }` |
| 注册 | POST | `/user/register` |  |
| 当前用户信息 | GET | `/user/userInfo` |  |
| 用户自更新 | POST | `/user/users/update-self` | 用户修改自己的资料 |
| 管理员列表用户 | POST | `/user/admin/users/list` |  |
| 管理员创建用户 | POST | `/user/admin/users/create` |  |
| 管理员更新用户 | POST | `/user/admin/users/update` |  |
| 管理员更新用户类型 | POST | `/user/admin/users/update-type` |  |
| 密码哈希 | POST | `/user/get_hash` |  |

---

## 角色与权限

### 角色接口

| 操作 | 方法   | 路径                       |
| ---- | ------ | -------------------------- |
| 列表 | POST   | `/user/admin/roles/list`   |
| 详情 | GET    | `/user/admin/roles/{id}`   |
| 创建 | POST   | `/user/admin/roles/create` |
| 更新 | POST   | `/user/admin/roles/update` |
| 删除 | DELETE | `/user/admin/roles/{id}`   |

### 权限接口

| 操作     | 方法 | 路径                        |
| -------- | ---- | --------------------------- |
| 全部权限 | GET  | `/user/admin/auths/all`     |
| 权限列表 | GET  | `/system/auths/list`        |
| 创建     | POST | `/system/auths/create`      |
| 更新     | POST | `/system/auths/{id}/update` |

### 权限模型

- 角色（Role）与权限（Auth）是多对多关系
- 用户通过 `role_id` 关联角色
- 登录后可从用户信息中获取角色及其权限编码列表

---

## 省份/税区（Province）

用于商户的省份配置，不同省份有不同的税率。

### 接口

| 操作 | 方法 | 路径                            |
| ---- | ---- | ------------------------------- |
| 列表 | POST | `/system/provinces/list`        |
| 创建 | POST | `/system/provinces/create`      |
| 更新 | POST | `/system/provinces/{id}/update` |
| 删除 | POST | `/system/provinces/{id}/delete` |

也可通过销售模块获取：`POST /sale/provinces/list`

### 税率字段

| 字段  | 说明             |
| ----- | ---------------- |
| GST   | 联邦商品及服务税 |
| PST   | 省销售税         |
| QST   | 魁北克销售税     |
| HST   | 统一销售税       |
| Total | 总税率           |

---

## 待审批

系统中的待审批事项来自两个来源：

1. **供应链订单** status=1（待审批）
2. **销售订单** status=1（待审批）

查询方式：

```
GET /supplier/supplier-orders/cache-sync?local_supplier_order_version=0
GET /sale/sale-orders/cache-sync?local_sale_order_version=0
```

合并两个列表即为所有待审批事项。

---

## 仪表盘（Dashboard）

### 接口

| 操作       | 方法 | 路径                       | 说明                 |
| ---------- | ---- | -------------------------- | -------------------- |
| 业务指引   | GET  | `/dashboard/guide-summary` | 各流程阶段的数量统计 |
| 总览       | GET  | `/dashboard/overview`      | 全局统计数据         |
| 供应链看板 | GET  | `/dashboard/supplier`      | 供应链相关统计       |
| 销售看板   | GET  | `/dashboard/sale`          | 销售相关统计         |
| 仓库看板   | GET  | `/dashboard/warehouse`     | 库存预警、仓储统计   |
| 财务看板   | GET  | `/dashboard/finance`       | 财务概览             |

仓库看板支持查询参数：`?days=30&stockThreshold=10&overstockThreshold=1000`

---

## 文件上传（OSS）

### 接口模式

每种业务实体的文件操作遵循统一模式：

```
上传：POST /oss/{entity_type}/{id}/upload   (multipart/form-data)
列表：POST /oss/{entity_type}/{id}/list
删除：POST /oss/{entity_type}/{id}/delete
```

### 支持的实体类型

| entity_type       | 说明           |
| ----------------- | -------------- |
| `product`         | 商品图片       |
| `productsku`      | SKU 图片       |
| `brand`           | 品牌 Logo      |
| `supplierorder`   | 供应链订单附件 |
| `supplierinvoice` | 供应链发票附件 |
| `saleorder`       | 销售订单附件   |
| `saleinvoice`     | 销售发票附件   |
| `taxinvoice`      | 税票附件       |
| `warehouseio`     | 出入库图片     |

### 获取临时访问 URL

```
POST /oss/temp-url
Body: { "key": "文件路径" }
→ 返回临时可访问的 URL
```
