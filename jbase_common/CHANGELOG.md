# 更新日志

所有重要的版本更新都会记录在此文件中。

---

## [3.6.2] - 2024-xx-xx

### 新增

- 新增 `PreventRapidFireAttribute` 防高频点击特性
- 新增 `OnCooldownEnd` 回调功能，支持冷却结束通知
- 新增 `CustomKeyPrefix` 和 `IgnoreParameters` 配置选项

### 修复

- 修复 `JobAutoRegistrar.RegisterJobs()` 单独使用时任务无法创建实例的问题
- 修复 `JobAutoRegistrar.RegisterJobs()` 调用的任务类型未注册到 DI 容器的问题

### 优化

- 优化异常处理中间件性能
- 优化 Redis 缓存连接稳定性

---

## [3.6.1] - 2024-xx-xx

### 新增

- 新增 `SetNXAsync` 异步方法
- 新增 `GetInfoAsync` 获取 Redis 服务器信息
- 新增 `GetDbSizeAsync` 获取数据库键数量

### 修复

- 修复 Memory 缓存 Key 列表同步问题
- 修复 `DelByPatternAsync` 正则匹配问题

---

## [3.6.0] - 2024-xx-xx

### 新增

- 新增定时任务分布式锁支持
- 新增任务执行重试机制
- 新增任务超时控制
- 新增 `JobExecutionContext` 生命周期钩子

### 优化

- 优化定时任务调度性能
- 优化 `BatchRegisterService` 扫描逻辑

---

## [3.5.0] - 2024-xx-xx

### 新增

- 新增 `OperationLogAttribute` 操作日志特性
- 新增敏感字段过滤功能
- 新增 `IOperationLogDatabaseWriter` 接口

---

## [3.4.0] - 2024-xx-xx

### 新增

- 新增 `PreventDuplicateRequestsAttribute` 防重复提交特性
- 新增 `UseGlobal` 全局锁支持
- 新增 `FactorNames` 组合参数防重

---

## [3.3.0] - 2024-xx-xx

### 新增

- 新增 JWT 自动刷新 Token 功能
- 新增 `RefreshTokenExpires` 配置项

### 优化

- 优化 Token 验证性能

---

## [3.2.0] - 2024-xx-xx

### 新增

- 新增 `RedisSentinelsConfig` Redis 哨兵模式配置
- 新增 Memory 缓存实现

---

## [3.1.0] - 2024-xx-xx

### 新增

- 新增 `App.CurrentUser` 全局用户访问
- 新增 `IUser` 接口
- 新增自定义用户信息解析支持

---

## [3.0.0] - 2024-xx-xx

### 重大更新

- 全面支持 .NET 6+ / .NET 8+
- 优化中间件执行顺序
- 移除过时 API

---

## [2.x.x] - 历史版本

### 2023-xx-xx

- 初始版本发布
- 支持 ASP.NET Core 3.1
