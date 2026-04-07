# DocumentFormat.OpenXml SDK 技术参考手册

> **用途**：供 AI 编程助手读取，作为生成 WeaveDoc OpenXML 样式映射模块代码的上下文参考。
> **来源**：Microsoft Learn 官方文档 + NuGet API 参考
> **版本**：DocumentFormat.OpenXml 3.5.1 | **更新日期**：2026-04-06

---

## 1. 概述

DocumentFormat.OpenXml 是微软官方的 .NET 库，用于**读写操作 Office Open XML 文档**（.docx、.xlsx、.pptx）。在 WeaveDoc 中主要用于精确修改 `.docx` 文件的样式（字体、字号、间距、对齐等）。

---

## 2. NuGet 安装

```xml
<PackageReference Include="DocumentFormat.OpenXml" Version="3.5.1" />
```

---

## 3. 核心 using 声明

```csharp
using DocumentFormat.OpenXml;
using DocumentFormat.OpenXml.Packaging;     // WordprocessingDocument, StyleDefinitionsPart
using DocumentFormat.OpenXml.Wordprocessing; // Style, Paragraph, Run, Bold, FontSize...
```

---

## 4. WordprocessingML 文档结构

### 4.1 元素到 C# 类的映射

| XML 元素 | C# 类 | 说明 |
|----------|-------|------|
| `<w:document>` | `Document` | 根元素 |
| `<w:body>` | `Body` | 文档主体容器 |
| `<w:p>` | `Paragraph` | 段落 |
| `<w:r>` | `Run` | 文本运行 |
| `<w:t>` | `Text` | 文本内容 |
| `<w:pPr>` | `ParagraphProperties` | 段落属性 |
| `<w:rPr>` | `RunProperties` / `StyleRunProperties` | 字符属性 |
| `<w:pStyle>` | `ParagraphStyleId` | 段落样式引用 |
| `<w:rFonts>` | `RunFonts` | 字体设置 |
| `<w:sz>` | `FontSize` | 字号（半磅单位） |
| `<w:b>` | `Bold` | 粗体 |
| `<w:i>` | `Italic` | 斜体 |
| `<w:jc>` | `Justification` | 对齐方式 |
| `<w:spacing>` | `Spacing` | 行距/段间距 |
| `<w:ind>` | `Indentation` | 缩进 |
| `<w:style>` | `Style` | 样式定义 |
| `<w:styles>` | `Styles` | 样式集合根元素 |

### 4.2 .docx 内部结构

```
document.docx (ZIP)
├── [Content_Types].xml
├── word/
│   ├── document.xml       ← 主文档
│   ├── styles.xml         ← 样式定义（核心操作目标）
│   ├── numbering.xml      ← 编号/列表
│   ├── settings.xml       ← 文档设置
│   ├── header1.xml        ← 页眉
│   ├── footer1.xml        ← 页脚
│   └── _rels/
│       └── document.xml.rels
└── docProps/
    ├── app.xml
    └── core.xml
```

---

## 5. 打开和操作文档

### 5.1 打开文档

```csharp
using var doc = WordprocessingDocument.Open(filePath, true); // true = 可编辑

// 创建新文档
using var doc = WordprocessingDocument.Create(filePath, WordprocessingDocumentType.Document);
```

### 5.2 获取样式部件

```csharp
MainDocumentPart mainPart = doc.MainDocumentPart ?? doc.AddMainDocumentPart();

// 获取 styles.xml 对应的部件
StyleDefinitionsPart? stylesPart = mainPart.StyleDefinitionsPart;

if (stylesPart is null)
{
    stylesPart = mainPart.AddNewPart<StyleDefinitionsPart>();
    stylesPart.Styles = new Styles();
}

Styles styles = stylesPart.Styles;
```

---

## 6. 样式操作（核心）

### 6.1 样式的 6 种类型

WordprocessingML 支持 6 种样式类型，通过 `Type` 属性设置：

```csharp
StyleValues.Paragraph  // 段落样式（最常用）
StyleValues.Character  // 字符样式
StyleValues.Table      // 表格样式
StyleValues.Numbering  // 编号样式
```

### 6.2 创建段落样式（微软官方示例）

```csharp
static void CreateAndAddParagraphStyle(
    StyleDefinitionsPart styleDefinitionsPart,
    string styleId,           // 内部标识符，如 "Heading1"
    string styleName,         // 显示名称，如 "heading 1"
    string aliases = "")      // 可选别名
{
    // 获取或创建 styles 根元素
    Styles styles = styleDefinitionsPart.Styles ??= new Styles();

    // 创建样式对象
    Style style = new Style()
    {
        Type = StyleValues.Paragraph,   // 段落样式
        StyleId = styleId,              // 内部 ID
        CustomStyle = true,             // 自定义样式
        Default = false                 // 非默认样式
    };

    // 样式元数据
    style.Append(new StyleName { Val = styleName });
    style.Append(new BasedOn { Val = "Normal" });     // 基于 Normal 样式
    style.Append(new NextParagraphStyle { Val = "Normal" }); // 下一段样式
    style.Append(new UIPriority { Val = 1 });         // UI 排序优先级

    if (!string.IsNullOrWhiteSpace(aliases))
    {
        style.Append(new Aliases { Val = aliases });
    }

    // 段落属性 (pPr)
    var pPr = new StyleParagraphProperties();
    pPr.Append(new Spacing { Before = "360", After = "80" });  // 段前段后间距
    pPr.Append(new Justification { Val = JustificationValues.Center }); // 居中
    style.Append(pPr);

    // 字符属性 (rPr)
    var rPr = new StyleRunProperties();
    rPr.Append(new RunFonts
    {
        Ascii = "黑体",
        EastAsia = "黑体",
        HighAnsi = "黑体"
    });
    rPr.Append(new Bold());                 // 粗体
    rPr.Append(new FontSize { Val = "32" }); // 16pt = 32 half-points
    rPr.Append(new FontSizeComplexScript { Val = "32" });
    style.Append(rPr);

    // 添加到样式集合
    styles.Append(style);
}
```

### 6.3 检查样式是否存在

```csharp
static bool IsStyleIdInDocument(WordprocessingDocument doc, string styleId)
{
    Styles? styles = doc.MainDocumentPart?.StyleDefinitionsPart?.Styles;
    if (styles is null) return false;

    return styles.Elements<Style>()
        .Any(s => s.StyleId == styleId && s.Type == StyleValues.Paragraph);
}
```

### 6.4 通过样式名查找 styleId

```csharp
static string? GetStyleIdFromName(WordprocessingDocument doc, string styleName)
{
    var styles = doc.MainDocumentPart?.StyleDefinitionsPart?.Styles;
    if (styles is null) return null;

    return styles.Descendants<StyleName>()
        .Where(sn => sn.Val?.Value == styleName
                  && ((Style?)sn.Parent)?.Type == StyleValues.Paragraph)
        .Select(sn => ((Style?)sn.Parent)?.StyleId?.Value)
        .FirstOrDefault();
}
```

---

## 7. 应用样式到段落

```csharp
// 获取或创建段落属性
if (p.Elements<ParagraphProperties>().Count() == 0)
{
    p.PrependChild(new ParagraphProperties());
}

p.ParagraphProperties ??= new ParagraphProperties();

// 设置样式 ID
p.ParagraphStyleId = new ParagraphStyleId { Val = "Heading1" };
```

---

## 8. 属性设置速查

### 8.1 字体

```csharp
// 字体族设置（三个属性分别控制不同文字范围）
new RunFonts
{
    Ascii = "宋体",        // 西文字体
    EastAsia = "宋体",     // 中文字体
    HighAnsi = "宋体"      // ANSI 字体
}
```

### 8.2 字号

```csharp
// FontSize 的 Val 值为半磅 (half-points)
// 12pt = "24", 14pt = "28", 16pt = "32", 小四 = "24", 三号 = "32"
new FontSize { Val = "24" }
new FontSizeComplexScript { Val = "24" }  // 复杂脚本（阿拉伯等）
```

### 8.3 粗体/斜体

```csharp
new Bold()       // 粗体（自闭合元素，存在即为启用）
new Italic()     // 斜体

// 显式关闭
new Bold { Val = false }
```

### 8.4 对齐

```csharp
new Justification { Val = JustificationValues.Left }    // 左对齐
new Justification { Val = JustificationValues.Center }  // 居中
new Justification { Val = JustificationValues.Right }   // 右对齐
new Justification { Val = JustificationValues.Both }    // 两端对齐
```

### 8.5 行距与段间距

```csharp
// Spacing 元素同时控制行距和段间距
new Spacing
{
    Before = "360",              // 段前间距（twips，1pt = 20 twips）
    After = "240",               // 段后间距（twips）
    Line = "360",                // 行距（1.5倍 = 360，基于 240 为单倍）
    LineRule = LineSpacingRuleValues.Auto  // 自动行距规则
}
```

### 8.6 缩进

```csharp
new Indentation
{
    FirstLine = "480",    // 首行缩进（twips，24pt = 480 twips）
    Hanging = "480",      // 悬挂缩进
    Left = "360",         // 左缩进
    Right = "360"         // 右缩进
}
```

### 8.7 页面设置

```csharp
// 页面尺寸（A4: 210mm × 297mm）
new PageSize { Width = 11907, Height = 16839 }  // twips

// 页边距
new PageMargin
{
    Top = 1418,     // 25mm ≈ 1418 twips
    Bottom = 1418,
    Left = 1701,    // 30mm ≈ 1701 twips
    Right = 1701
}
```

### 8.8 颜色

```csharp
new Color { Val = "FF0000" }                          // 十六进制 RGB
new Color { ThemeColor = ThemeColorValues.Accent1 }   // 主题色
```

---

## 9. 单位换算

| 源单位 | 目标单位 | 公式 | 示例 |
|--------|---------|------|------|
| pt (磅) | half-points | `pt × 2` | 12pt = 24 |
| pt (磅) | twips (缇) | `pt × 20` | 12pt = 240 twips |
| mm (毫米) | twips | `mm × 567` | 30mm = 17010 twips |
| 字符宽 | pt | `字符数 × fontSize` | 2字符@12pt = 24pt |

---

## 10. 常见学术样式到 OpenXML 映射

| 学术格式 | C# 代码 |
|---------|---------|
| 黑体三号 | `new RunFonts { EastAsia = "黑体" }` + `new FontSize { Val = "32" }` |
| 宋体小四 | `new RunFonts { EastAsia = "宋体" }` + `new FontSize { Val = "24" }` |
| 1.5 倍行距 | `new Spacing { Line = "360", LineRule = Auto }` |
| 首行缩进两字符 | `new Indentation { FirstLine = "480" }` (at 12pt) |
| 标题居中 | `new Justification { Val = Center }` |

---

## 11. 完整工作流示例

```csharp
using var doc = WordprocessingDocument.Open("output.docx", true);
var mainPart = doc.MainDocumentPart!;
var stylesPart = mainPart.StyleDefinitionsPart ?? mainPart.AddNewPart<StyleDefinitionsPart>();

// 1. 确保 styles 根元素存在
stylesPart.Styles ??= new Styles();

// 2. 创建/修改 Heading1 样式
var heading1 = stylesPart.Styles.Elements<Style>()
    .FirstOrDefault(s => s.StyleId == "Heading1");

if (heading1 != null)
{
    // 更新已有样式
    var rPr = heading1.Elements<StyleRunProperties>().FirstOrDefault()
           ?? heading1.AppendChild(new StyleRunProperties());
    rPr.Append(new RunFonts { Ascii = "黑体", EastAsia = "黑体", HighAnsi = "黑体" });
    rPr.Append(new FontSize { Val = "32" });
    rPr.Append(new Bold());
}
else
{
    // 创建新样式
    CreateAndAddParagraphStyle(stylesPart, "Heading1", "heading 1");
}

// 3. 保存
stylesPart.Styles.Save();
```

---

## 12. 参考链接

- **NuGet**：https://www.nuget.org/packages/DocumentFormat.OpenXml
- **微软官方教程**：
  - [Create and add a paragraph style](https://learn.microsoft.com/en-us/office/open-xml/word/how-to-create-and-add-a-paragraph-style-to-a-word-processing-document)
  - [Apply a style to a paragraph](https://learn.microsoft.com/en-us/office/open-xml/word/how-to-apply-a-style-to-a-paragraph-in-a-word-processing-document)
- **API 参考**：https://learn.microsoft.com/en-us/dotnet/api/documentformat.openxml.wordprocessing
- **GitHub**：https://github.com/OfficeDev/Open-XML-SDK
- **ISO/IEC 29500 规范**：https://www.iso.org/standard/71691.html
