# JBase.FreeSql.Core

FreeSql Core 基础服务库，封装了常用的 CRUD 操作、审计功能、事务管理、DTO 映射等基础功能。

## 特性

- **多层架构**：Entity → Repository → Service，职责清晰
- **灵活主键支持**：支持 int、long、Guid、string 四种主键类型
- **自动审计**：插入/更新/删除时自动填充审计字段（创建人、创建时间、更新时间等）
- **软删除支持**：通过接口实现自动判断软删除或硬删除
- **高性能 DTO 映射**：使用 IL 动态生成映射代码，性能接近直接调用
- **枚举映射**：可配置的枚举到数据库类型转换策略
- **异步事务**：支持嵌套事务管理
- **丰富分页**：提供多种分页查询扩展方法
- **依赖注入**：完整的 IServiceCollection 扩展支持

## 项目结构

```
JBase.FreeSql.Core/
├── Entity/
│   └── BaseEntity.cs           # 实体基类和审计接口
├── Repository/
│   ├── IBaseRepository.cs      # 仓储接口
│   └── BaseRepository.cs        # 仓储实现
├── Service/
│   ├── IBaseService.cs         # 服务接口
│   └── BaseService.cs          # 服务实现
├── Options/
│   ├── AuditOptions.cs         # 审计配置选项
│   └── EnumMappingOptions.cs   # 枚举映射配置
├── Extensions/
│   ├── FreeExtensions.cs       # FreeSql 核心扩展
│   ├── ServiceExtensions.cs    # 服务扩展
│   └── RepositoryExtensions.cs # 仓储扩展
└── Mapper/
    └── MapperGenerator.cs      # IL 映射器生成器
```

## 依赖版本

| 包名 | 版本 |
|------|------|
| FreeSql | 3.5.211 |
| FreeSql.Provider.MySql | 3.5.211 |
| FreeSql.Provider.SqlServer | 3.5.211 |
| Microsoft.Extensions.DependencyInjection | 6.0.0 |

## 目标框架

- netstandard2.1

## NuGet 安装

```bash
dotnet add package JBase.FreeSql.Core
```

## 快速开始

参见 [快速开始](1-快速开始) 文档。
