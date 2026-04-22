# JWT 认证

JBase.Common 提供完整的 JWT 认证解决方案，包括 Token 颁发、验证、刷新等功能。

## 核心组件

| 组件 | 说明 |
|------|------|
| `TokenService` | Token 颁发和验证服务 |
| `JwtMiddleware` | JWT 认证中间件 |
| `JwtOptions` | JWT 配置选项 |

## 配置

### appsettings.json

```json
{
  "Jwt": {
    "Issuer": "MyApp",
    "Audience": "MyApp.User",
    "SecretKey": "你的密钥至少32位字符，用于HMAC签名",
    "Expires": 30,
    "ValidateExpires": true,
    "HeadField": "Authorization",
    "Prefix": "Bearer",
    "EnableAutoRefreshToken": true,
    "RefreshTokenExpires": 5
  }
}
```

### JwtOptions 配置项说明

```csharp
public class JwtOptions
{
    /// <summary>
    /// 发布者（Iss），标识谁创建了这个 Token
    /// </summary>
    public string Issuer { get; set; }

    /// <summary>
    /// 接收者（Aud），标识这个 Token 是给谁用的
    /// </summary>
    public string Audience { get; set; }

    /// <summary>
    /// 加密密钥，至少 32 个字符
    /// </summary>
    public string SecretKey { get; set; }

    /// <summary>
    /// Token 有效期（分钟），默认 30 分钟
    /// </summary>
    public int Expires { get; set; } = 30;

    /// <summary>
    /// 是否验证过期时间
    /// </summary>
    public bool ValidateExpires { get; set; } = true;

    /// <summary>
    /// 请求头字段名，默认 "Authorization"
    /// </summary>
    public string HeadField { get; set; } = "Authorization";

    /// <summary>
    /// Token 前缀（如 "Bearer"），默认不设置
    /// </summary>
    public string Prefix { get; set; }

    /// <summary>
    /// 是否启用自动刷新 Token
    /// </summary>
    public bool EnableAutoRefreshToken { get; set; }

    /// <summary>
    /// Token 自动刷新窗口（分钟），当 Token 剩余有效期小于此值时自动刷新
    /// </summary>
    public int RefreshTokenExpires { get; set; }
}
```

## 注册服务

JWT 中间件 `UseJwt()` 是自包含的，无需额外注册服务。只需在中间件管道中启用即可。

## 中间件配置

```csharp
public void Configure(IApplicationBuilder app)
{
    // 建议放在靠前的位置
    app.UseJwt();

    // 其他中间件...
}
```

## 基本使用

### 1. 登录并颁发 Token

```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly ITokenService _tokenService;

    public AuthController(ITokenService tokenService)
    {
        _tokenService = tokenService;
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginDto dto)
    {
        // 验证用户账号密码（这里简化处理）
        var user = await _userService.ValidateAsync(dto.Username, dto.Password);
        if (user == null)
        {
            return BadRequest("用户名或密码错误");
        }

        // 创建 Claims
        var claims = new List<Claim>
        {
            new Claim("userId", user.Id.ToString()),
            new Claim("userName", user.Name),
            new Claim("role", "Admin")
        };

        // 颁发 Token
        var token = _tokenService.IssueToken(claims);

        return Ok(new { token });
    }
}
```

### 2. 获取 Token 验证结果

```csharp
public class SomeService
{
    public void Validate(string token)
    {
        var (isValid, result) = _tokenService.ValidateToken(token, out var claims);

        if (isValid)
        {
            var userId = claims.FirstOrDefault(c => c.Type == "userId")?.Value;
            var userName = claims.FirstOrDefault(c => c.Type == "userName")?.Value;
        }
        else
        {
            switch (result)
            {
                case JwtResEnum.Expired:
                    // Token 已过期
                    break;
                case JwtResEnum.Unauthorized:
                    // Token 无效
                    break;
            }
        }
    }
}
```

### 3. 刷新 Token

```csharp
[HttpPost("refresh")]
public IActionResult Refresh([FromBody] RefreshTokenDto dto)
{
    try
    {
        var newToken = _tokenService.RefreshToken(dto.Token);
        return Ok(new { token = newToken });
    }
    catch (CommonException ex)
    {
        return BadRequest(ex.Message);
    }
}
```

## 自动刷新 Token

当启用自动刷新后，Token 会在过期前自动更新：

```json
{
  "Jwt": {
    "EnableAutoRefreshToken": true,
    "RefreshTokenExpires": 5
  }
}
```

### 工作原理

```
用户请求 → JwtMiddleware 验证 Token
    ↓
Token 剩余有效期 < RefreshTokenExpires (5分钟)?
    ↓ 是
返回新 Token（在响应头 Refresh-Token 中）
```

### 前端处理

```javascript
// 封装请求方法，自动处理 Token 刷新
async function request(url, options = {}) {
    const response = await fetch(url, {
        ...options,
        headers: {
            'Authorization': `Bearer ${getToken()}`,
            'Content-Type': 'application/json',
            ...options.headers
        }
    });

    // 检查是否需要刷新 Token
    const newToken = response.headers.get('Refresh-Token');
    if (newToken) {
        saveToken(newToken);
    }

    return response;
}
```

## 手动解析 Token Claims

```csharp
public class UserService
{
    public void Process()
    {
        // 从 App.User 获取 ClaimsPrincipal
        var user = App.User;

        if (user.Identity?.IsAuthenticated == true)
        {
            var userId = user.FindFirst("userId")?.Value;
            var userName = user.FindFirst("userName")?.Value;
            var role = user.FindFirst("role")?.Value;
        }
    }
}
```

## 从 Claims 提取字典

```csharp
var claims = new List<Claim>
{
    new Claim("userId", "123"),
    new Claim("userName", "张三")
};

var dict = _tokenService.GetDicFromClaims(claims);
// 返回: { "userId": "123", "userName": "张三" }
```

## 使用 App.CurrentUser 获取当前用户

JBase.Common 提供了更便捷的 `App.CurrentUser` 访问方式：

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProfileController : ControllerBase
{
    [HttpGet("me")]
    public IActionResult GetCurrentUser()
    {
        var user = App.CurrentUser;

        if (user == null || !user.IsAuthenticated)
        {
            return Unauthorized();
        }

        return Ok(new
        {
            Id = user.Id,
            Name = user.Name,
            Roles = user.Roles
        });
    }
}
```

## IUser 接口

```csharp
public interface IUser
{
    /// <summary>
    /// 用户ID
    /// </summary>
    string Id { get; }

    /// <summary>
    /// 用户名
    /// </summary>
    string Name { get; }

    /// <summary>
    /// 角色列表
    /// </summary>
    IEnumerable<string> Roles { get; }

    /// <summary>
    /// 是否已认证
    /// </summary>
    bool IsAuthenticated { get; }
}
```

## 自定义用户信息解析

如果需要自定义 Claims 到用户信息的映射，可以实现自己的 `IUser`：

```csharp
public class CustomUser : IUser
{
    private readonly ClaimsPrincipal _principal;

    public CustomUser(ClaimsPrincipal principal)
    {
        _principal = principal;
    }

    public string Id => _principal.FindFirst("uid")?.Value;

    public string Name => _principal.FindFirst("nickname")?.Value
                      ?? _principal.FindFirst(ClaimTypes.Name)?.Value;

    public IEnumerable<string> Roles => _principal.FindAll("role")
        .Select(c => c.Value)
        .ToList();

    public bool IsAuthenticated => _principal.Identity?.IsAuthenticated ?? false;
}

// 注册自定义实现
services.AddScoped<IUser, CustomUser>();
```

## 常见问题

### 1. Token 验证失败的原因

| 原因 | 说明 | 解决方案 |
|------|------|----------|
| Token 过期 | `Expires` 时间已过 | 刷新 Token |
| 签名无效 | 密钥不匹配或被篡改 | 检查 `SecretKey` |
| Issuer/Audience 不匹配 | 配置不一致 | 检查 `Issuer` 和 `Audience` |
| 格式错误 | Token 格式不正确 | 检查前端传递的 Token |

### 2. SecretKey 安全建议

- 至少 32 个字符
- 使用随机生成的强密钥，不要使用简单字符串
- 生产环境建议通过环境变量或密钥管理服务注入
- 定期更换密钥

### 3. Token 安全建议

- 生产环境务必启用 HTTPS
- 不要在 Token 中存储敏感信息（如密码）
- 根据业务需求设置合理的 `Expires` 时间
- 敏感操作可以考虑短有效期 Token
