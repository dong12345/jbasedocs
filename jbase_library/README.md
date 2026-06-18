# JBaseLibrary

一套实用的 .NET 基础类库，封装了常用的加密、扩展方法、帮助类、验证、对象映射、反射访问、拼音转换等工具，助您快速开发。

## 特性

- **跨平台**：基于 .NET Standard 2.1，支持 .NET Core 3.0+ 和 .NET 5+
- **功能丰富**：涵盖加密、扩展方法、帮助类、验证、对象映射、拼音转换等常用工具
- **高性能**：使用表达式树、IL Emit 等技术优化反射、转换等热路径
- **易扩展**：所有扩展方法遵循链式调用风格

## 模块概览

| 模块 | 命名空间 | 说明 |
|------|----------|------|
| [核心工具类](2-核心工具类) | `JBaseLibrary`、`JBaseLibrary.Accessors`、`JBaseLibrary.EasyComparers` | `Ensure` 参数断言、`Equatable<T>` 基类、`Base64UrlSafe`、对象反射访问、对象差异比较 |
| [加密模块](3-加密模块) | `JBaseLibrary.Encrypt` | MD5、SHA256、AES、DES 加解密 |
| [扩展方法](4-扩展方法) | `JBaseLibrary.Extensions` | 字符串、日期、对象、集合、JSON、LINQ、文件、Excel 等 |
| [帮助类](5-帮助类) | `JBaseLibrary.Helpers` | 文件、Excel、二维码、压缩、图片压缩、JSON、日期、缓存、搜索、验证 等 |
| [验证模块](6-验证模块) | `JBaseLibrary.Validate` | `ValidateContext` 链式验证器、`ValidateHelper` 通用校验 |
| [拼音模块](7-拼音模块) | `JBaseLibrary.NPinyin` | 汉字 ↔ 拼音 |

## NuGet 安装

```bash
dotnet add package JBaseLibrary
```

> **依赖说明**：JBaseLibrary 基于以下 NuGet 包构建：
> - Newtonsoft.Json（JSON 序列化）
> - NPOI（Excel 操作）
> - QRCoder、SkiaSharp（二维码 / 验证码生成）
> - SixLabors.ImageSharp（图片压缩）
> - ICSharpCode.SharpZipLib（ZIP 压缩）
> - Microsoft.AspNetCore.Http.Features（`IFormFile` 扩展，仅 `JBaseLibrary.Extensions.FileAndDirectoryInfoExtensions`）
> - System.ComponentModel.Annotations（数据注解）

## 快速开始

### 字符串扩展

```csharp
using JBaseLibrary.Extensions;

string name = null;
name.IsNullOrEmpty();  // true
name.IsNotNullOrEmpty();  // false

"HelloWorld".Left(5);  // "Hello"
"HelloWorld".Right(5); // "World"

"a, b, c".SplitAndTrim(',');  // ["a", "b", "c"]

"Hello, {name}".Render(new { name = "World" });  // "Hello, World"

"hello".FirstUpperCase();  // "Hello"
```

### 对象转换

```csharp
using JBaseLibrary.Extensions;

var json = user.ToJson();
var user2 = json.ToObject<User>();

var copy = user.Clone();

var num = "123".ObjTo<int>();  // 123

entity.ModifyByDto(dto);

var dict = user.ToDictionary();

var dto = user.Mapper<UserDto>();  // Mapper<T>() 新建目标对象
```

### 加密操作

```csharp
using JBaseLibrary.Encrypt;

var md5 = MD5Encrypt.MD5("password");                            // 32位大写十六进制
var sha256 = SHA256.SHA256Hash("password");                       // Base64UrlSafe

var encrypted = AESEncrypt.Encrypt("data", "key12345678901234");
var decrypted = AESEncrypt.Decrypt(encrypted, "key12345678901234");

var desEncrypted = DESEncrypt.Encrypt("data", "Key123Ace#321Key");
var desDecrypted = DESEncrypt.Decrypt(desEncrypted, "Key123Ace#321Key");

var urlSafe = Base64UrlSafe.Encode(bytes);
var original = Base64UrlSafe.Decode(urlSafe);
```

### 日期处理

```csharp
using JBaseLibrary.Extensions;

DateTime.Now.IsBetween(start, end);              // 范围判断
DateTime.Now.IsWeekend();                       // 是否周末
DateTime.Now.IsLeapYear();                      // 是否闰年
DateTime.Now.IsLastDayOfTheMonth();             // 是否月末

DateTime.Now.DiffDays(endDate);                 // 相差天数
DateTime.Now.DiffHours(endDate);                // 相差小时
DateTime.Now.DiffMinutes(endDate);              // 相差分钟

DateTime.Now.ToCommonString();                  // "yyyy-MM-dd HH:mm:ss"
DateTime.Now.ToCommonDateString();              // "yyyy-MM-dd"
DateTime.Now.Week();                            // "星期四"
DateTime.Now.DateToUpper();                     // "二〇二六年四月二十三日"

DateTime.Now.ToUnixTimeSeconds();               // 10位时间戳
1745328000.FromUnixTimeSeconds();               // DateTime
```

### 数值处理

```csharp
using JBaseLibrary.Extensions;

10000m.YuanToWY();                              // 1万元
1m.WYuanToY();                                  // 10000元

123.4500m.TrimZero();                           // 123.45
123.456789m.SetDigits(2);                       // 123.45
123.456789m.RoundTo(2);                         // 123.46

0.9535.ToPercentage();                          // "95.35%"
0.9535.ToPercentage(1);                        // "95.4%"

123.NumberToUpper();                            // "一百二十三"
12345.NumberToUpper();                          // "一万二千三百四十五"
165.7m.MoneyToUpper();                          // "壹佰陆拾伍点柒"

3600.FormatSeconds();                           // "1小时0分0秒"
```

### 集合操作

```csharp
using JBaseLibrary.Extensions;

// 树形结构
var tree = list.ToTree(
    (a, b) => b.ParentId == 0,
    (a, b) => a.Id == b.ParentId,
    (parent, children) => parent.Children = children.ToList()
);

// 条件筛选
var result = list.WhereIf(x => x.Status == 1, isActive);

// 分页
var page = list.GetPage(1, 10);

// 批量处理
var batches = list.Batch(100);

// 随机选择
var random = list.SelectRandom();

// 集合差异
var (added, removed) = newList.GetDiff(oldList);

// 集合比较（顺序敏感/不敏感）
bool eq = list1.ListComparator(list2);
bool eq2 = list1.ListComparator(list2, byOrder: false);
```

### 参数断言

```csharp
using JBaseLibrary;

Ensure.That(condition, "条件必须为真");
Ensure.Not(condition, "条件必须为假");
Ensure.NotNull(obj, nameof(obj));
Ensure.Equal(a, b, "值必须相等");
Ensure.NotEqual(a, b, "值不能相等");
Ensure.NotNullOrEmpty(collection, "集合不能为空");
Ensure.NotNullOrWhiteSpace(str, "字符串不能为空");

Ensure.Exists(directoryInfo);
Ensure.Exists(fileInfo);
```

### 验证器

```csharp
using JBaseLibrary.Validate;

var context = new ValidateContext<int>(100);
context.MustGreaterThan(50)
       .MustLessThan(200)
       .MustNotNull();

if (context.IsValid)
{
    // 验证通过
}
else
{
    foreach (var error in context.GetErrorMessages())
    {
        Console.WriteLine(error);
    }
}
```

完整使用示例（含 DTO + 嵌套属性 + Service 调用）见 [验证模块 - 实战案例](6-验证模块#9-实战案例)。

### Excel 操作

```csharp
using JBaseLibrary.Helpers;

var headers = new Dictionary<string, string>
{
    { "姓名", "Name" },
    { "年龄", "Age" }
};
var bytes = ExcelHelper.ExportExcel("学生信息", headers, students);
var students = ExcelHelper.ReadExcel<Student>("path.xlsx", 0, 1);
```

### 二维码生成

```csharp
using JBaseLibrary.Helpers;

var bytes = QrCodeHelper.GenerateQrCodeBytes("https://example.com");
var base64 = QrCodeHelper.GenerateQrCodeBase64("https://example.com");
// <img src="data:image/png;base64,@base64" />
QrCodeHelper.GenerateQrCodeToFile("https://example.com", "qrcode.png");
```

### 对象差异比较

```csharp
using JBaseLibrary.EasyComparers;

bool isEqual = EasyComparer.Instance.Compare(
    old, new, inherit: true, includePrivate: false,
    out var variances);

foreach (var (prop, v) in variances)
{
    if (v.Varies)
    {
        Console.WriteLine($"{prop.Name}: {v.LeftValue} -> {v.RightValue}");
    }
}
```

### 顺序 GUID

```csharp
using JBaseLibrary.Helpers;

// SQL Server 推荐 AtEnd
GuidHelper.SequentialGuidType = SequentialGuidType.AtEnd;
Guid id = GuidHelper.Next();
```

## 文档目录

- [快速开始](1-快速开始)
- [核心工具类](2-核心工具类)
- [加密模块](3-加密模块)
- [扩展方法](4-扩展方法)
- [帮助类](5-帮助类)
- [验证模块](6-验证模块)
- [拼音模块](7-拼音模块)

## 版本信息

- 当前版本：2.1.4
- 目标框架：.NET Standard 2.1

## 许可证

本项目采用 MIT 许可证。
