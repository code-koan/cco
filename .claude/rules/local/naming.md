# 命名约定

> 定义项目的命名规范，遵循 DDD 最佳实践。

---

## 一、领域层命名

### 实体与值对象

| 类型 | 规范 | 示例 |
|-----|------|------|
| 实体 | PascalCase（业务名词） | `User`, `Order`, `Product` |
| 值对象 | PascalCase（描述性名词） | `Money`, `Address`, `DateRange` |
| 聚合根 | PascalCase（与聚合同名） | `Order`（订单聚合根） |

### 仓储接口

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}Repository | `OrderRepository` | 接口定义在领域层 |

```typescript
// 领域层定义
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
  delete(order: Order): Promise<void>;
}
```

### 领域服务

| 规范 | 示例 | 说明 |
|------|------|------|
| {业务动作}DomainService | `PaymentDomainService` | 跨聚合的业务逻辑 |

### 领域事件

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}{过去式动词}Event | `OrderCreatedEvent` | 表达业务状态变更 |

常用过去式动词：`Created`, `Updated`, `Deleted`, `Completed`, `Cancelled`, `Reserved`, `Released`

---

## 二、应用层命名

### 应用服务

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}ApplicationService | `OrderApplicationService` | 编排用例 |

### 命令（Command）

| 规范 | 示例 | 说明 |
|------|------|------|
| {动词}{聚合名}Command | `CreateOrderCommand` | 写操作参数 |
| {动词}{聚合名}Command | `UpdateOrderCommand` | 写操作参数 |

常用动词：`Create`, `Update`, `Delete`, `Confirm`, `Cancel`, `Submit`

### 查询（Query）

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}By{条件}Query | `OrderByIdQuery` | 读操作参数 |
| {聚合名}ListQuery | `OrderListQuery` | 列表查询 |

### 事件处理器

| 规范 | 示例 | 说明 |
|------|------|------|
| {事件名}Handler | `OrderCreatedEventHandler` | 处理领域事件 |

---

## 三、用户界面层命名

### Controller

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}Controller | `OrderController` | REST API 入口 |

### DTO

| 类型 | 规范 | 示例 |
|-----|------|------|
| 请求 DTO | {动作}{聚合名}Request | `CreateOrderRequest` |
| 响应 DTO | {聚合名}Response | `OrderResponse` |
| 列表响应 | {聚合名}ListResponse | `OrderListResponse` |

### Assembler

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}Assembler | `OrderAssembler` | DTO 与领域对象转换 |

---

## 四、基础设施层命名

### 仓储实现

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}RepositoryImpl | `OrderRepositoryImpl` | 实现领域层接口 |

### 持久化对象（PO）

| 规范 | 示例 | 说明 |
|------|------|------|
| {聚合名}PO | `OrderPO` | 数据库映射对象 |

### 外部服务适配器

| 规范 | 示例 | 说明 |
|------|------|------|
| {服务名}Adapter | `PaymentGatewayAdapter` | 外部服务封装 |

---

## 五、文件命名规范

### 领域层文件

| 类型 | 规范 | 示例 |
|-----|------|------|
| 实体 | {EntityName}.ts | `Order.ts` |
| 值对象 | {ValueObjectName}.ts | `Money.ts` |
| 仓储接口 | {AggregateName}Repository.ts | `OrderRepository.ts` |
| 领域服务 | {ServiceName}.ts | `PaymentDomainService.ts` |
| 领域事件 | {EventName}.ts | `OrderCreatedEvent.ts` |

### 应用层文件

| 类型 | 规范 | 示例 |
|-----|------|------|
| 应用服务 | {AggregateName}ApplicationService.ts | `OrderApplicationService.ts` |
| 命令 | {CommandName}.ts | `CreateOrderCommand.ts` |
| 查询 | {QueryName}.ts | `OrderByIdQuery.ts` |
| 事件处理器 | {EventName}Handler.ts | `OrderCreatedEventHandler.ts` |

### 用户界面层文件

| 类型 | 规范 | 示例 |
|-----|------|------|
| Controller | {AggregateName}Controller.ts | `OrderController.ts` |
| DTO | {DTOName}.ts | `CreateOrderRequest.ts` |
| Assembler | {AggregateName}Assembler.ts | `OrderAssembler.ts` |

### 基础设施层文件

| 类型 | 规范 | 示例 |
|-----|------|------|
| 仓储实现 | {AggregateName}RepositoryImpl.ts | `OrderRepositoryImpl.ts` |
| PO | {AggregateName}PO.ts | `OrderPO.ts` |
| Mapper | {AggregateName}Mapper.ts | `OrderMapper.ts` |

---

## 六、变量命名规范

### 领域对象变量

| 类型 | 规范 | 示例 |
|-----|------|------|
| 实体变量 | 业务名词 | `order`, `customer` |
| 值对象变量 | 描述性名词 | `money`, `address` |
| ID 类型 | 名词 + Id | `orderId`, `customerId` |

### 集合变量

| 类型 | 规范 | 示例 |
|-----|------|------|
| 列表 | 复数形式 | `orders`, `items` |
| Map | 名词 + Map | `orderMap`, `productMap` |

---

## 七、方法命名规范

### 实体方法

| 类型 | 规范 | 示例 |
|-----|------|------|
| 业务行为 | 动词 + 名词 | `addItem()`, `calculateTotal()` |
| 状态变更 | set + 状态 | `setAsCompleted()`, `setAsCancelled()` |
| 查询 | is/has/can + 形容词 | `isCompleted()`, `hasItems()` |

### 仓储方法

| 类型 | 规范 | 示例 |
|-----|------|------|
| 单条查询 | findBy + 条件 | `findById()`, `findByOrderNo()` |
| 列表查询 | findAll + 条件 | `findAllByCustomerId()` |
| 保存 | save | `save(order)` |
| 删除 | delete | `delete(order)` |

### 应用服务方法

| 类型 | 规范 | 示例 |
|-----|------|------|
| 命令处理 | handle + Command | `handleCreateOrder()` |
| 查询处理 | handle + Query | `handleOrderByIdQuery()` |
