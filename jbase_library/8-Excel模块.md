# Excel 模块

> 命名空间：`JBaseLibrary.Excel`
> 主类：`JBaseLibrary.Excel.ExcelHelper`（static）
> 配套属性：`JBaseLibrary.Attributes.ExcelHeaderAttribute`、`JBaseLibrary.Attributes.ExcelRequiredAttribute`
> 依赖：NPOI（`.xls` / `.xlsx`）

本模块围绕 `ExcelHelper` 静态类，提供 Excel 的**导入 / 导出 / 模板生成**三大能力。基于 NPOI 实现，跨平台、不依赖 Office 与 GDI。

- **导出**：固定列导出、动态列导出、图片导出（同步 / 异步）
- **读取**：常规读取返回 `IList<T>` 或 `DataSet`；百万行场景可用流式 API
- **模板生成**：根据 DTO 自动生成带表头的导入模板，必填列表头自动标红

---

## 1. 通用前置：定义 DTO 与表头注解

几乎所有 API 都依赖同一个 DTO 模型。`ExcelHelper` 通过反射读属性，自动生成表头、匹配读取列、识别必填列。

```csharp
using JBaseLibrary.Attributes;

public class ProductDto
{
    // 列名映射：导出/读取时"名称"对应 Name 属性
    [ExcelHeader("名称")]
    public string Name { get; set; }

    [ExcelHeader("价格")]
    public decimal Price { get; set; }

    // 标记为必填：模板中该列表头自动变红
    [ExcelHeader("SKU")]
    [ExcelRequired]
    public string Sku { get; set; }

    // 标记为图片 URL（图片导出时自动下载并嵌入）
    [ExcelHeader("封面", IsImageUrl = true)]
    public string CoverUrl { get; set; }

    public DateTime CreateTime { get; set; }   // 无注解时使用属性名作为表头
}
```

**列匹配规则**（读取与生成模板时）：

1. 若属性标记了 `[ExcelHeader(HeaderName)]`，按 `HeaderName` 匹配 Excel 表头
2. 否则按属性名匹配（忽略大小写）

**必填识别规则**（生成模板时）：

1. `[ExcelRequired]`（JBaseLibrary 自定义）
2. `[Required]`（`System.ComponentModel.DataAnnotations`）

满足任一即可，该列表头自动变红提示必填。

---

## 2. 导出

### 2.1 固定列导出

最常见的导出场景：`Dictionary<表头文本, 属性名>` 显式声明列。

```csharp
using JBaseLibrary.Excel;

var headers = new Dictionary<string, string>
{
    { "名称", nameof(ProductDto.Name) },
    { "价格", nameof(ProductDto.Price) },
    { "创建时间", nameof(ProductDto.CreateTime) }
};

var products = new List<ProductDto>
{
    new() { Name = "手机",   Price = 5999m, CreateTime = DateTime.Now },
    new() { Name = "耳机",   Price =  299m, CreateTime = DateTime.Now }
};

// 返回 byte[]（适合小数据量 / 直接返回文件下载）
byte[] bytes = ExcelHelper.ExportExcel(
    excelName:   "产品列表",
    headerValue: headers,
    list:        products);

// 写到任意 Stream（适合大文件 / 自定义输出）
using var fs = File.Create("products.xlsx");
ExcelHelper.ExportExcelToStream(
    excelName:    "产品列表",
    headerValue:  headers,
    list:         products,
    isXlsx:       true,
    isShowTitle:  true,
    titleRGB:     new byte[] { 26, 140, 155 },   // 大标题背景色，默认 {26,140,155}
    outputStream: fs);
```

**参数速查**

| 参数 | 类型 | 默认 | 说明 |
|------|------|------|------|
| `excelName` | string | 必填 | 文件名（不含扩展名），同时作为 Sheet 名与大标题文本 |
| `headerValue` | `Dictionary<string,string>` | 必填 | Key=表头文本，Value=属性名 |
| `list` | `IReadOnlyList<T>` | 必填 | 数据集合 |
| `isXlsx` | bool | `true` | `true`=xlsx，`false`=xls |
| `isShowTitle` | bool | `true` | 是否在首行显示大标题 |
| `titleRGB` | `byte[]` | `{26,140,155}` | 大标题背景色 RGB（3 字节） |
| `outputStream` | `Stream` | 必填（流式版本） | 目标输出流 |

### 2.2 动态列导出

适用于"列数不固定"的场景，如评分表（每位专家一列，但专家人数可变）。

> 静态版"动态列"是指从集合属性派生多列，而非运行时新增列。

```csharp
public class ScoreRecord
{
    public string Name { get; set; }
    public List<ExpertScore> ExpertScores { get; set; }
}
public class ExpertScore
{
    public string ExpertName { get; set; }
    public decimal Score { get; set; }
}

var fixedHeaders = new Dictionary<string, string>
{
    { "姓名", nameof(ScoreRecord.Name) }
};

var cfg = new DynamicColumnConfig
{
    PropertyName  = nameof(ScoreRecord.ExpertScores),   // 数据源属性名
    HeaderTitle   = "评分详情",                          // 动态列标题
    LabelProperty = nameof(ExpertScore.ExpertName),     // 子项的"标签"属性
    ValueProperty = nameof(ExpertScore.Score),          // 子项的"值"属性
    ValueSuffix   = "分"                                 // 可选，单元格后缀
};

byte[] bytes = ExcelHelper.ExportExcelWithDynamicColumns(
    excelName:           "成绩表",
    fixedHeaders:        fixedHeaders,
    dynamicColumnConfig: cfg,
    list:                records);
```

**`DynamicColumnConfig` 字段**

| 字段 | 必填 | 说明 |
|------|------|------|
| `PropertyName` | ✓ | DTO 上承载动态列数据的属性名（`List<TChild>` 类型） |
| `HeaderTitle` | ✓ | 这一组动态列共用的表头文本 |
| `LabelProperty` | ✓ | 子项的标签属性（如"专家A"） |
| `ValueProperty` | ✓ | 子项的数值属性（如 95） |
| `ValueSuffix` | × | 单元格值后缀（如 `"分"` → "95分"），为空则不追加 |

### 2.3 图片导出

当某一列存的是图片（`byte[]` 或图片 URL `string`）时使用。

#### 同步版（属性为 `byte[]`）

```csharp
byte[] bytes = ExcelHelper.ExportExcelWithImages(
    excelName: "产品列表",
    headers:   headers,
    list:      products,
    imageOptions: new ExcelImageOptions
    {
        Width  = 100,
        Height = 100
    });
```

#### 异步版（属性为 URL `string`）

异步版会在导出前自动从 `ImageBaseUrl` 拼接 URL，**并行预下载**图片数据，适合大数据量。

```csharp
byte[] bytes = await ExcelHelper.ExportExcelWithImagesAsync(
    excelName: "产品列表",
    headers:   headers,
    list:      products,
    imageOptions: new ExcelImageOptions
    {
        ImageBaseUrl = "https://cdn.example.com",   // string 属性值将拼接此域名
        Width        = 120,
        Height       = 120
    });
```

#### 自动识别图片列

无需手写 `ImagePropertyNames`，只要 DTO 上用 `[ExcelHeader(IsImageUrl = true)]` 标注即可：

```csharp
public class ProductImageDto
{
    [ExcelHeader("名称")]                       public string Name { get; set; }
    [ExcelHeader("封面", IsImageUrl = true)]    public string CoverUrl { get; set; }
}

// 配合 ExcelImageOptions.WithAutoImageColumns<T>()，自动收集所有 IsImageUrl=true 的属性
var opts = new ExcelImageOptions { Width = 100, Height = 100 }
    .WithAutoImageColumns<ProductImageDto>();
```

**`ExcelImageOptions` 字段**

| 字段 | 默认 | 说明 |
|------|------|------|
| `ImageBaseUrl` | `null` | 图片 URL 域名（仅异步版 + string 列生效） |
| `Width` / `Height` | 100 / 100 | 图片显示尺寸（像素） |
| `LockAspectRatio` | `true` | 是否锁定宽高比 |
| `ImageFormat` | `Auto` | `Auto`=按字节探测；可强制 `Png` / `Jpeg` / `Bmp` |
| `ImagePropertyNames` | `null` | 显式声明哪些 string 属性是图片列；为空时按 `[ExcelHeader(IsImageUrl=true)]` 自动收集 |

---

## 3. 读取

### 3.1 普通读取

读取后自动转 `IList<T>`，列按 `[ExcelHeader]` 或属性名匹配（忽略大小写）。

```csharp
using JBaseLibrary.Excel;
using Microsoft.AspNetCore.Http;

// 从上传文件读取（Web 项目）
public IActionResult Import(IFormFile file)
{
    var items = ExcelHelper.ReadExcel<ProductDto>(
        formFile:      file,
        headRowNum:    0,    // 第 1 行（0-based）是表头
        contentRowNum: 1);   // 从第 2 行开始读取数据

    return Ok(items);
}

// 从文件路径读取
var items2 = ExcelHelper.ReadExcel<ProductDto>("products.xlsx", 0, 1);

// 只想拿到原始 DataSet（自己再做转换）
DataSet ds = ExcelHelper.ReadExcel(file, 0, 1);
```

### 3.2 读取选项

`ReadExcelOptions` 全部字段默认为 `true`（兼容旧行为），按需关闭：

```csharp
var options = new ReadExcelOptions
{
    SkipHiddenSheets      = true,    // 跳过"隐藏 / 仅打印"等非数据 Sheet
    SkipMergedBlankCells  = true,    // 跳过合并区域内的空白单元格
    KeepRawNumeric        = false,   // true 时保留数值原始精度（避免科学计数法截断）
    FormulaFallbackValue  = ""       // 公式求值失败时返回的兜底值
};

var items = ExcelHelper.ReadExcel<ProductDto>(file, 0, 1, options);
```

### 3.3 流式读取（百万行场景）

普通 `ReadExcel<T>` 会把所有行一次性构建成 `DataSet`，大文件场景下内存占用大。流式版按行惰性产出，**边读边处理**，对 GC 更友好。

```csharp
await foreach (var row in ExcelHelper.ReadExcelRowStreamAsync(file, 0, 1))
{
    // row.SheetName : Sheet 名
    // row.RowIndex  : 1-based 行号（与 Excel 用户视角一致）
    // row.Values    : 列名 → 单元格值的字典（已按 ReadExcelOptions 处理）

    var name = row.Values.GetValueOrDefault("名称");

    // 一行处理完即可丢弃引用，无需全量驻留内存
    await SaveToDbAsync(name, row);
}
```

**`ExcelRow` 字段**

| 字段 | 类型 | 说明 |
|------|------|------|
| `SheetName` | `string` | Sheet 名 |
| `RowIndex` | `int` | 1-based 行号 |
| `Values` | `IReadOnlyDictionary<string,string>` | 列名 → 单元格值（已按 ReadExcelOptions 处理） |

> **stream 生命周期**：框架内部 `try/finally` 自动释放文件流，`break` / 抛异常均不会泄漏。

### 3.4 错误信息聚合

读取过程中如果某行某列类型转换失败、必填字段为空，框架会收集到错误集合：

```csharp
// ExcelImportResult<T> 包含成功实体与错误集合
// （如需此能力，可直接使用 DataTableExtensions.Excel 中的 ToListWithHeader 链路，
//   或扩展自定义读取实现）

// 已有错误集合的格式化扩展：
var lines = errors.FormatErrors();
// 输出示例：
// 第3行：SKU 不能为空；价格 必须是数字
// 第5行：SKU 不能为空
```

**`ExcelImportErrorExtensions` 方法**

| 方法 | 说明 |
|------|------|
| `FormatErrors(rowPrefix, rowSuffix, separator)` | 按行分组，返回 `List<string>` |
| `FormatErrorsAsText(...)` | 按行分组，返回单条字符串（换行分隔） |
| `GetErrorCount()` | 错误总数 |
| `GetErrorRowCount()` | 涉及错误行数（去重） |

---

## 4. 模板生成

根据 DTO 自动生成"仅有表头"的标准导入模板，必填列表头自动标红。

```csharp
// 生成 byte[]
byte[] template = ExcelHelper.GenerateImportTemplate<ProductDto>(
    sheetName:    "产品导入",          // 可空；为空时使用"类型名 + 导入模板"
    isXlsx:       true,
    isShowTitle:  false,                // 导入模板通常不需要大标题
    titleRGB:     new byte[] { 26, 140, 155 });

// 写入到响应流（Web 下载）
using var fs = File.Create("产品导入模板.xlsx");
ExcelHelper.GenerateImportTemplateToStream<ProductDto>(
    sheetName:    "产品导入",
    isXlsx:       true,
    isShowTitle:  false,
    titleRGB:     null,
    outputStream: fs);
```

---

## 5. ASP.NET Core 一站式下载（`JBase.Common`）

如果项目已引入 `JBase.Common`，可直接用 `HttpResponse` 扩展把 Excel **流式输出到浏览器**，无需手动处理 `Content-Type` / `Content-Disposition`（已自动按 RFC 5987 编码中文文件名）：

```csharp
using JBase.Common.Excel;

[HttpGet]
public async Task Export()
{
    var products = LoadProducts();

    await Response.WriteExcelWithImagesAsync(
        sheetName:         "产品列表",
        list:              products,
        downloadFileName:  "产品列表.xlsx",         // 中文文件名安全
        imageOptions:      new ExcelImageOptions { Width = 100, Height = 100 });
}
```

`JBase.Common.Excel` 的 `WriteExcelWithImagesAsync` 会自动：

- 设置正确的 `Content-Type`（xlsx / xls）
- 用 RFC 5987 编码中文文件名
- 临时开启 Kestrel 同步 IO（NPOI 写入为同步 API），写完后恢复原值

> 详见 `JBase.Common` 项目文档。

---

## 6. 方法汇总

| 方法 | 返回 | 说明 |
|------|------|------|
| `ExportExcel<T>(name, headers, list, isXlsx?, isShowTitle?, titleRGB?)` | `byte[]` | 固定列导出（byte[]） |
| `ExportExcelToStream<T>(...)` | `void` | 固定列导出（写到 Stream） |
| `ExportExcelWithDynamicColumns<T>(...)` | `byte[]` | 动态列导出（byte[]） |
| `ExportExcelWithDynamicColumnsToStream<T>(...)` | `void` | 动态列导出（写到 Stream） |
| `ExportExcelWithImages<T>(...)` | `byte[]` | 图片导出，**同步**（byte[] 图片列） |
| `ExportExcelWithImagesToStream<T>(...)` | `void` | 图片导出，**同步**（写到 Stream） |
| `ExportExcelWithImagesAsync<T>(...)` | `Task<byte[]>` | 图片导出，**异步**（下载 URL） |
| `ExportExcelWithImagesAsyncToStream<T>(...)` | `Task` | 图片导出，**异步**（写到 Stream） |
| `ReadExcel<T>(IFormFile, headRow, contentRow, options?)` | `IList<T>` | 读取上传文件，强类型 |
| `ReadExcel<T>(string, headRow, contentRow, options?)` | `IList<T>` | 读取文件路径，强类型 |
| `ReadExcel(IFormFile, headRow, contentRow, options?)` | `DataSet` | 读取上传文件，原始 DataSet |
| `ReadExcel(string, headRow, contentRow, options?)` | `DataSet` | 读取文件路径，原始 DataSet |
| `ReadExcelRowStreamAsync(IFormFile, headRow, contentRow, options?, ct?)` | `IAsyncEnumerable<ExcelRow>` | 流式读取上传文件 |
| `ReadExcelRowStreamAsync(string, headRow, contentRow, options?, ct?)` | `IAsyncEnumerable<ExcelRow>` | 流式读取文件路径 |
| `GenerateImportTemplate<T>(sheetName?, isXlsx?, isShowTitle?, titleRGB?)` | `byte[]` | 生成导入模板（byte[]） |
| `GenerateImportTemplateToStream<T>(...)` | `void` | 生成导入模板（写到 Stream） |
| `BuildHeadersFromType<T>()` | `ExcelHeadersResult` | 反射构建表头 + 图片列名（内部辅助） |

---

## 7. 选型速查

| 场景 | 推荐 API |
|------|----------|
| 导出小数据量、直接返回文件下载 | `ExportExcel<T>` → `byte[]` |
| 导出大数据量、写入 `Response.Body` | `ExportExcelToStream<T>` |
| 评分表 / 专家列表等"列不固定" | `ExportExcelWithDynamicColumns<T>` |
| 列表里有图片，`byte[]` 类型 | `ExportExcelWithImages<T>` |
| 列表里有图片，存的是 URL 字符串 | `ExportExcelWithImagesAsync<T>` |
| 读取普通 Excel → `IList<T>` | `ReadExcel<T>(file, 0, 1)` |
| 读取并想保留原始 `DataSet` | `ReadExcel(file, 0, 1)` |
| 百万行导入，内存敏感 | `ReadExcelRowStreamAsync(file, 0, 1)` |
| 给用户下载导入模板 | `GenerateImportTemplate<T>` |
| Web 一键下载（处理中文文件名 / Kestrel 同步 IO） | `JBase.Common.Excel.WriteExcelWithImagesAsync` |

---

## 8. 配套类型

| 类型 | 命名空间 | 说明 |
|------|----------|------|
| `ExcelImageOptions` | `JBaseLibrary.Excel` | 图片列配置 |
| `ImageFormat` | `JBaseLibrary.Excel` | 图片格式枚举（Auto/Png/Jpeg/Bmp） |
| `DynamicColumnConfig` | `JBaseLibrary.Excel` | 动态列配置 |
| `ReadExcelOptions` | `JBaseLibrary.Excel` | 读取选项 |
| `ExcelRow` | `JBaseLibrary.Excel` | 流式读取单行（SheetName / RowIndex / Values） |
| `ExcelImportError` | `JBaseLibrary.Excel` | 单条导入错误 |
| `ExcelImportErrorType` | `JBaseLibrary.Excel` | 错误类型枚举（必填为空 / 类型转换失败 / 列不存在） |
| `ExcelImportResult<T>` | `JBaseLibrary.Excel` | 导入结果（Items + Errors） |
| `ExcelImportErrorExtensions` | `JBaseLibrary.Excel` | 错误集合格式化扩展 |
| `ExcelHeaderAttribute` | `JBaseLibrary.Attributes` | 表头注解（`HeaderName`、`IsImageUrl`） |
| `ExcelRequiredAttribute` | `JBaseLibrary.Attributes` | 必填列标记（无参，标记即生效） |
