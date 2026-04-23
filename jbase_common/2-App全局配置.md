# App 全局配置

`App` 类是 JBase.Common 的核心管理器，提供全局配置访问、服务获取和当前用户信息等功能。

## 核心功能

### 1. 初始化

`App` 采用双阶段初始化模式：

```
阶段一：App.InitializeConfiguration()  → Build 之前调用
阶段二：App.InitializeServices()      → Build 之后调用
```

#### 初始化配置（Build 之前）

```csharp
// 方式一：直接传入 IConfiguration
App.InitializeConfiguration(configuration);

// 方式二：从指定路径加载配置
App.InitializeConfigurationFromPath("/path/to/config");

// 方式三：带选项初始化
App.InitializeConfiguration(configuration, new AppInitOptions
{
    JwtOptionsKey = "MyJwt",
    SwaggerConfigKey = "MySwagger",
    CacheConfigKey = "MyCache",
    AllowCorsKey = "MyCors"
});
```

#### 初始化服务（Build 之后）

```csharp
var app = Host.CreateDefaultBuilder(args)
    .ConfigureServices((context, services) =>
    {
        services.AddControllers();
    })
    .Build();

App.InitializeServices(app.ApplicationServices);
```

### 2. 配置访问

#### 获取字符串配置

```csharp
// 获取配置值（不存在返回空字符串）
var value = App.GetValue("Jwt:Issuer");

// 获取嵌套配置，带默认值
var secret = App.GetValue("Jwt:SecretKey", "默认密钥");
```

#### 获取强类型配置对象

```csharp
// 获取配置对象
var jwtOptions = App.GetConfig<JwtOptions>("Jwt");

// 访问配置属性
var issuer = jwtOptions.Issuer;
var expires = jwtOptions.Expires;
```

### 3. 服务获取

#### 获取服务实例

```csharp
// 获取缓存服务
var cache = App.GetServiceOrDefault<ICacheService>();

// 获取其他服务
var logger = App.GetServiceOrDefault<ILogger<MyClass>>();
```

> **注意**：`GetServiceOrDefault` 会优先从当前 HTTP 请求作用域解析服务（保证 Scoped 服务正常工作），如果不在请求上下文中，则从根 ServiceProvider 获取。

#### 创建服务作用域

```csharp
using (var scope = App.CreateScope())
{
    var service = scope.ServiceProvider.GetRequiredService<ISomeScopedService>();
    // 使用服务
}
```

### 4. 当前用户信息

```csharp
// 获取当前登录用户
var user = App.CurrentUser;

if (user != null && user.IsAuthenticated)
{
    var userId = user.Id;      // 用户ID
    var userName = user.Name; // 用户名
    var roles = user.Roles;   // 角色列表
}
```

### 5. HTTP 上下文

```csharp
// 获取 HttpContext
var httpContext = App.HttpContext;

// 获取 ClaimsPrincipal
var claimsPrincipal = App.User;
```

### 6. 静态配置属性

初始化后，可以直接访问以下静态属性：

| 属性 | 类型 | 说明 |
|------|------|------|
| `App.Configuration` | `IConfiguration` | 根配置对象 |
| `App.JwtOptions` | `JwtOptions` | JWT 配置 |
| `App.SwaggerConfig` | `SwaggerConfig` | Swagger 配置 |
| `App.CacheConfig` | `CacheConfig` | 缓存配置 |
| `App.AllowCors` | `string[]` | 允许跨域的域名列表 |
| `App.IsInitialized` | `bool` | 是否已初始化配置 |
| `App.IsServiceInitialized` | `bool` | 是否已初始化服务 |

### 7. AppInitOptions 配置项

```csharp
public class AppInitOptions
{
    /// <summary>
    /// Jwt配置项在配置文件中的节点路径，默认 "Jwt"
    /// </summary>
    public string JwtOptionsKey { get; set; } = "Jwt";

    /// <summary>
    /// Swagger配置项在配置文件中的节点路径，默认 "Swagger"
    /// </summary>
    public string SwaggerConfigKey { get; set; } = "Swagger";

    /// <summary>
    /// Cache配置项在配置文件中的节点路径，默认 "Cache"
    /// </summary>
    public string CacheConfigKey { get; set; } = "Cache";

    /// <summary>
    /// AllowCors配置项在配置文件中的节点路径，默认 "AllowCors"
    /// </summary>
    public string AllowCorsKey { get; set; } } = "AllowCors";
}
```

## 完整示例

```csharp
// Program.cs
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args)
    {
        return Host.CreateDefaultBuilder(args)
            .ConfigureWebHostDefaults(webBuilder =>
            {
                webBuilder.ConfigureAppConfiguration((context, config) =>
                {
                    // 阶段一：初始化配置
                    App.InitializeConfiguration(config.Build());
                });

                webBuilder.ConfigureServices((context, services) =>
                {
                    services.AddJBaseCore();
                    services.AddControllers();
                });

                webBuilder.Configure((context, app) =>
                {
                    // 阶段二：初始化服务
                    App.InitializeServices(app.ApplicationServices);

                    // 使用配置
                    Console.WriteLine($"JWT Issuer: {App.JwtOptions.Issuer}");
                    Console.WriteLine($"Cache Type: {App.CacheConfig.Type}");

                    app.UseRouting();
                    app.UseEndpoints(endpoints =>
                    {
                        endpoints.MapControllers();
                    });
                });
            });
    }
}

// 业务代码中使用
public class UserService
{
    public void Process()
    {
        // 获取当前用户
        var userId = App.CurrentUser?.Id;

        // 获取配置
        var jwtExpires = App.JwtOptions.Expires;

        // 获取服务
        var cache = App.GetServiceOrDefault<ICacheService>();
        var logger = App.GetServiceOrDefault<ILogger<UserService>>();

        // 读取配置
        var customValue = App.GetValue("CustomSection:Key");
        var customValue2 = App.GetValue("CustomSection:Key", "默认值");
        var customConfig = App.GetConfig<MyOptions>("MySection");
        var customConfig2 = App.GetConfig<MyOptions>("MySection", new MyOptions());
    }
}
```

## 注意事项

1. **初始化顺序**：必须先调用 `InitializeConfiguration()`，再调用 `InitializeServices()`
2. **线程安全**：初始化操作是线程安全的，支持多线程场景
3. **单例模式**：`App` 类内部使用双重检查锁定实现单例
4. **作用域服务**：`GetServiceOrDefault` 会优先从请求作用域解析 Scoped 服务
