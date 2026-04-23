# JBaseLibrary

一套实用的 .NET 基础类库，封装了常用的加密、扩展方法、帮助类、验证等工具，助您快速开发。

## 特性

- **跨平台**：基于 .NET Standard 2.1，支持 .NET Core 3.0+ 和 .NET 5+
- **功能丰富**：涵盖加密、扩展方法、帮助类、验证、拼音转换等常用工具
- **高性能**：使用表达式树、IL Emit 等技术优化性能
- **易扩展**：所有扩展方法遵循链式调用风格

## NuGet 安装

```bash
dotnet add package JBaseLibrary
```

> **依赖说明**：JBaseLibrary 基于以下 NuGet 包构建：
> - Newtonsoft.Json（JSON 序列化）
> - NPOI（Excel 操作）
> - QRCoder、SkiaSharp（二维码生成）
> - SixLabors.ImageSharp（图片处理）
> - SharpZipLib（ZIP 压缩）
> - System.ComponentModel.Annotations（数据注解）

## 快速开始

### 字符串扩展

```csharp
using JBaseLibrary.Extensions;

// 判断字符串是否为空
string name = null;
name.IsNullOrEmpty();  // true
name.IsNotNullOrEmpty();  // false

// 截取字符串
"HelloWorld".Left(5);  // "Hello"
"HelloWorld".Right(5);  // "World"

// 字符串分割并去空格
"a, b, c".SplitAndTrim(',');  // ["a", "b", "c"]

// 字符串格式化
"Hello, {name}".Render(new { name = "World" });  // "Hello, World"

// 首字母大写
"hello".FirstUpperCase();  // "Hello"
```

### 对象转换

```csharp
using JBaseLibrary.Extensions;

// 对象转 JSON
var json = user.ToJson();

// JSON 反序列化
var user = json.ToObject<User>();

// 对象深拷贝
var copy = user.Clone();

// 对象属性映射
entity.ModifyByDto(dto);

// 对象转字典
var dict = user.ToDictionary();

// 类型转换（高性能）
var num = "123".ObjTo<int>();  // 123
```

### 加密操作

```csharp
using JBaseLibrary.Encrypt;

// MD5 加密
var md5 = MD5Encrypt.MD5("password");  // 返回32位大写

// SHA256 加密（返回 Base64UrlSafe 编码）
var sha256 = SHA256.SHA256Hash("password");  // Base64UrlSafe 编码

// AES 加解密
var encrypted = AESEncrypt.Encrypt("data", "key12345678901234");
var decrypted = AESEncrypt.Decrypt(encrypted, "key12345678901234");

// DES 加解密
var desEncrypted = DESEncrypt.Encrypt("data", "Key123Ace#321Key");
var desDecrypted = DESEncrypt.Decrypt(desEncrypted, "Key123Ace#321Key");

// Base64UrlSafe（JWT 使用）
var urlSafe = Base64UrlSafe.Encode(bytes);
var original = Base64UrlSafe.Decode(urlSafe);
```

### 日期处理

```csharp
using JBaseLibrary.Extensions;

// 日期判断
DateTime.Now.IsBetween(start, end);  // 是否在范围内
DateTime.Now.IsWeekend();  // 是否周末
DateTime.Now.IsLeapYear();  // 是否闰年
DateTime.Now.IsLastDayOfTheMonth();  // 是否月末

// 日期差异
DateTime.Now.DiffDays(endDate);  // 相差天数
DateTime.Now.DiffHours(endDate);  // 相差小时
DateTime.Now.DiffMinutes(endDate);  // 相差分钟

// 日期格式
DateTime.Now.ToCommonString();  // "yyyy-MM-dd HH:mm:ss"
DateTime.Now.ToCommonDateString();  // "yyyy-MM-dd"

// 日期转中文
DateTime.Now.Week();  // "星期四"
DateTime.Now.DateToUpper();  // "二〇二六年四月二十三日"

// 时间戳转换
DateTime.Now.ToUnixTimeSeconds();  // 10位时间戳
DateTime.Now.ToUnixTimeMilliseconds();  // 13位时间戳
1745328000.FromUnixTimeSeconds();  // DateTime
```

### 数值处理

```csharp
using JBaseLibrary.Extensions;

// 金额转换
10000m.YuanToWY();  // 1万元
1m.WYuanToY();  // 10000元

// 小数处理
123.4500m.TrimZero();  // 123.45
123.456789m.SetDigits(2);  // 123.45（截取）
123.456789m.RoundTo(2);  // 123.46（四舍五入）

// 百分比
0.9543.ToPercentage();  // "95.43%"
0.9543.ToPercentage(1);  // "95.4%"

// 数字转中文
123.NumberToUpper();  // "一百二十三"
12345.NumberToUpper();  // "一万二千三百四十五"
165.7m.MoneyToUpper();  // "壹佰陆拾伍点柒"

// 时间格式化
3600.FormatSeconds();  // "1小时0分0秒"
```

### 集合操作

```csharp
using JBaseLibrary.Extensions;

// 树形结构
var tree = list.ToTree(
    (a, b) => b.ParentId == 0,  // 根节点条件
    (parent, child) => parent.Id == child.ParentId,  // 父子关系
    (parent, children) => parent.Children = children.ToList()  // 添加子节点
);

// 条件筛选
var result = list.WhereIf(x => x.Status == 1, isActive);

// 分页
var page = list.GetPage(1, 10);

// 批量处理
var batches = list.Batch(100);

// 随机选择
var random = list.SelectRandom();

// 集合比较
var (added, removed) = newList.GetDiff(oldList);
```

### 参数断言

```csharp
using JBaseLibrary;

// 基本断言
Ensure.That(condition, "条件必须为真");
Ensure.Not(condition, "条件必须为假");
Ensure.NotNull(obj, "对象不能为空");
Ensure.Equal(a, b, "值必须相等");
Ensure.NotEqual(a, b, "值不能相等");
Ensure.NotNullOrEmpty(collection, "集合不能为空");
Ensure.NotNullOrWhiteSpace(str, "字符串不能为空");

// 文件/目录存在检查
Ensure.Exists(directoryInfo);
Ensure.Exists(fileInfo);
```

### 验证器

```csharp
using JBaseLibrary.Validate;

var context = new ValidateContext<int>(100);
context.MustGreaterThan(50)
       .MustLessThan(200);

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

### Excel 操作

```csharp
using JBaseLibrary.Helpers;

// 导出 Excel
var headers = new Dictionary<string, string>
{
    { "姓名", "Name" },
    { "年龄", "Age" }
};
var bytes = ExcelHelper.ExportExcel("学生信息", headers, students);

// 读取 Excel
var students = ExcelHelper.ReadExcel<Student>("path.xlsx", 0, 1);
```

### 二维码生成

```csharp
using JBaseLibrary.Helpers;
using SkiaSharp;

// 生成二维码字节数组
var bytes = QrCodeHelper.GenerateQrCodeBytes("https://example.com");

// 生成 Base64（用于 HTML）
var base64 = QrCodeHelper.GenerateQrCodeBase64("https://example.com");
// <img src="data:image/png;base64,..." />

// 保存到文件
QrCodeHelper.GenerateQrCodeToFile("https://example.com", "qrcode.png");
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
