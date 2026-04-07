# WeaveDoc 技术参考手册：OpenXML 样式映射 · Pandoc AST · AFD 模板协议

> **用途**：本文档供 AI 编程助手（Claude / Copilot 等）读取，作为生成 WeaveDoc 语义转换模块代码的上下文参考。
> **版本**：v1.1 | **更新日期**：2026-04-06
> **适用模块**：语义转换系统（WeaveDoc.Converter）、本地配置管理（WeaveDoc.Config）

---

## 1. 架构概览

```
用户编写 Markdown (.md)
        │
        ▼
   Markdig 解析 ──→ Markdown AST
        │
        ▼
  AFD 模板 (JSON) ──→ 样式规则注入
        │
        ▼
   Pandoc 转换管道
   ├─ Markdown → DOCX (OpenXML)
   └─ Markdown → PDF (via LaTeX/typst)
        │
        ▼
   OpenXML 样式映射层（C# DocumentFormat.OpenXml）
   ├─ styles.xml 生成/修改
   ├─ 段落样式 (ParagraphStyle)
   ├─ 字符样式 (RunStyle / RunProperties)
   └─ 编号/列表样式 (NumberingDefinition)
        │
        ▼
   最终输出: .docx / .pdf
```

### 1.1 核心数据流

```
AFD_Template.json → AfdStyleRule[] → Pandoc --reference-doc → OpenXML StyleCorrection → Output.docx
```

### 1.2 模块职责边界

| 模块 | 输入 | 输出 | 职责 |
|------|------|------|------|
| AFD 样式解析器 | AFD JSON 文件 | `AfdStyleRule` 对象集合 | 解析模板，提供样式查询接口 |
| Pandoc 转换管道 | Markdown + reference-doc | 原始 .docx | 调用 Pandoc CLI，完成格式转换 |
| OpenXML 样式映射 | 原始 .docx + AFD 样式规则 | 最终 .docx | 精确修正字体、间距、编号等样式 |
| BibTeX 管理器 | .bib 文件 | 引用数据 | 解析 BibTeX，提供引用插入接口 |

---

## 2. AFD 模板协议规范

### 2.1 JSON Schema

```jsonc
{
  "$schema": "https://weavedoc.dev/schemas/afd-template-v1.json",
  "meta": {
    "templateName": "武汉大学本科毕业论文",
    "version": "1.0.0",
    "author": "WeaveDoc",
    "description": "适用于武汉大学计算机学院本科毕业论文格式要求"
  },

  // 全局默认样式
  "defaults": {
    "fontFamily": "宋体",
    "fontSize": 12,           // 单位: pt (磅)
    "lineSpacing": 1.5,       // 行距倍数
    "pageSize": {
      "width": 210,           // 单位: mm
      "height": 297
    },
    "margins": {
      "top": 25,              // 单位: mm
      "bottom": 25,
      "left": 30,
      "right": 30
    }
  },

  // 分级样式定义
  "styles": {
    "heading1": {
      "displayName": "标题 1",
      "fontFamily": "黑体",
      "fontSize": 16,
      "bold": true,
      "alignment": "center",
      "spaceBefore": 24,      // 单位: pt
      "spaceAfter": 18,
      "lineSpacing": 1.5
    },
    "heading2": {
      "displayName": "标题 2",
      "fontFamily": "黑体",
      "fontSize": 14,
      "bold": true,
      "alignment": "left",
      "spaceBefore": 18,
      "spaceAfter": 12,
      "lineSpacing": 1.5
    },
    "heading3": {
      "displayName": "标题 3",
      "fontFamily": "黑体",
      "fontSize": 13,
      "bold": true,
      "alignment": "left",
      "spaceBefore": 12,
      "spaceAfter": 6,
      "lineSpacing": 1.5
    },
    "body": {
      "displayName": "正文",
      "fontFamily": "宋体",
      "fontSize": 12,
      "firstLineIndent": 24,  // 首行缩进，单位: pt (2字符 ≈ 24pt)
      "lineSpacing": 1.5
    },
    "caption": {
      "displayName": "题注",
      "fontFamily": "宋体",
      "fontSize": 10.5,
      "alignment": "center",
      "spaceBefore": 6,
      "spaceAfter": 6
    },
    "footnote": {
      "displayName": "脚注",
      "fontFamily": "宋体",
      "fontSize": 9,
      "lineSpacing": 1.0
    },
    "reference": {
      "displayName": "参考文献",
      "fontFamily": "宋体",
      "fontSize": 10.5,
      "lineSpacing": 1.0,
      "hangingIndent": 24     // 悬挂缩进
    },
    "abstract": {
      "displayName": "摘要",
      "fontFamily": "宋体",
      "fontSize": 12,
      "lineSpacing": 1.5,
      "firstLineIndent": 24
    }
  },

  // 页眉页脚
  "headerFooter": {
    "header": {
      "text": "武汉大学本科毕业论文",
      "fontFamily": "宋体",
      "fontSize": 9,
      "alignment": "center"
    },
    "footer": {
      "pageNumbering": true,
      "format": "arabic",       // arabic | roman
      "alignment": "center",
      "startPage": 1
    }
  },

  // 编号方案
  "numbering": {
    "headingNumbering": {
      "format": "decimal",     // decimal | chinese | alpha
      "separator": ".",
      "levels": [
        { "format": "一,二,三", "suffix": "、" },
        { "format": "1,2,3", "suffix": "." },
        { "format": "1,2,3", "suffix": ")" }
      ]
    },
    "listStyle": {
      "bullet": "●",
      "orderedFormat": "1,2,3"
    }
  }
}
```

### 2.2 AFD 字段到 OpenXML 属性的映射表

| AFD 字段 | OpenXML 元素 | XML 路径 | 说明 |
|----------|-------------|----------|------|
| `fontFamily` | `<w:rFonts>` | `w:rPr/w:rFonts` | `w:ascii`, `w:eastAsia`, `w:hAnsi` 分别控制西文/中文/ANSI 字体 |
| `fontSize` | `<w:sz>` | `w:rPr/w:sz` | 值为半磅 (half-points)，12pt = 24 |
| `bold` | `<w:b>` | `w:rPr/w:b` | 自闭合元素，存在即为粗体 |
| `italic` | `<w:i>` | `w:rPr/w:i` | 同上 |
| `alignment` | `<w:jc>` | `w:pPr/w:jc` | 值: `left`, `center`, `right`, `both` |
| `lineSpacing` | `<w:spacing>` | `w:pPr/w:spacing` | `w:line` 值 = 行距 × 240 (for 1.5x = 360) |
| `spaceBefore` | `<w:spacing>` | `w:pPr/w:spacing` | `w:before` 值单位为二十分之一磅 (twips) |
| `spaceAfter` | `<w:spacing>` | `w:pPr/w:spacing` | `w:after` 同上 |
| `firstLineIndent` | `<w:ind>` | `w:pPr/w:ind` | `w:firstLine` 值单位为 twips (1pt = 20 twips) |
| `hangingIndent` | `<w:ind>` | `w:pPr/w:ind` | `w:hanging` 同上 |
| `margins.*` | `<w:pgMar>` | `w:sectPr/w:pgMar` | 值单位为 twips (1mm ≈ 567 twips) |
| `pageSize.*` | `<w:pgSz>` | `w:sectPr/w:pgSz` | 值单位为 twips |

---

## 3. OpenXML 与 Pandoc 详细参考

> OpenXML SDK 的详细 API 用法（样式创建/修改/应用、属性速查、单位换算等）请参阅：
> **[openxml-sdk-reference.md](openxml-sdk-reference.md)**
>
> Pandoc 的详细用法（CLI 参数、reference-doc 机制、Lua Filter 语法、AST JSON 等）请参阅：
> **[pandoc-reference.md](pandoc-reference.md)**
>
> Markdig 的详细用法（Pipeline 构建、AST 节点遍历、自定义扩展等）请参阅：
> **[markdig-reference.md](markdig-reference.md)**

本节仅保留 AFD → OpenXML/Pandoc 的**集成映射关系**和**端到端代码**。

---

## 4. 完整转换流程示例

### 4.1 端到端管道代码

```csharp
namespace WeaveDoc.Converter;

/// <summary>
/// WeaveDoc 核心转换引擎：Markdown → AFD → DOCX/PDF
/// </summary>
public class DocumentConversionEngine
{
    private readonly PandocPipeline _pandoc;
    private readonly string _referenceDocPath; // 基础参考文档

    public DocumentConversionEngine(PandocPipeline pandoc, string referenceDocPath)
    {
        _pandoc = pandoc;
        _referenceDocPath = referenceDocPath;
    }

    /// <summary>
    /// 完整转换流程
    /// </summary>
    public async Task<ConversionResult> ConvertAsync(
        string markdownPath,
        string outputFormat,     // "docx" | "pdf"
        AfdTemplate template,
        string? luaFilterPath = null,
        CancellationToken ct = default)
    {
        var tempDir = Path.Combine(Path.GetTempPath(), $"weavedoc-{Guid.NewGuid():N}");
        Directory.CreateDirectory(tempDir);

        try
        {
            // Step 1: 准备 reference-doc（基于模板生成）
            var refDocPath = Path.Combine(tempDir, "reference.docx");
            await PrepareReferenceDocAsync(refDocPath, template);

            // Step 2: Pandoc 转换（Markdown → DOCX）
            var rawDocxPath = Path.Combine(tempDir, "raw_output.docx");
            if (luaFilterPath != null)
            {
                await _pandoc.ConvertWithLuaFilterAsync(
                    markdownPath, rawDocxPath, luaFilterPath, "docx", ct);
            }
            else
            {
                await _pandoc.MarkdownToDocxAsync(
                    markdownPath, rawDocxPath, refDocPath, ct);
            }

            // Step 3: OpenXML 样式精确修正
            OpenXmlStyleCorrector.ApplyAfdStyles(rawDocxPath, template);
            OpenXmlStyleCorrector.ApplyPageSettings(rawDocxPath, template.Defaults);

            if (template.HeaderFooter != null)
            {
                OpenXmlStyleCorrector.ApplyHeaderFooter(
                    rawDocxPath, template.HeaderFooter);
            }

            // Step 4: 输出到最终位置
            var outputPath = markdownPath.Replace(".md", $".{outputFormat}");
            if (outputFormat == "docx")
            {
                File.Copy(rawDocxPath, outputPath, overwrite: true);
            }
            else if (outputFormat == "pdf")
            {
                // 通过 Word COM 或 LibreOffice 转换为 PDF
                // 或使用 Pandoc 直接输出 PDF（需要 XeLaTeX）
                await _pandoc.MarkdownToPdfAsync(
                    markdownPath, outputPath, ct: ct);
            }

            return new ConversionResult
            {
                Success = true,
                OutputPath = outputPath,
                Format = outputFormat
            };
        }
        catch (Exception ex)
        {
            return new ConversionResult
            {
                Success = false,
                ErrorMessage = ex.Message
            };
        }
        finally
        {
            // 清理临时文件
            try { Directory.Delete(tempDir, recursive: true); } catch { }
        }
    }

    private async Task PrepareReferenceDocAsync(string path, AfdTemplate template)
    {
        // 创建一个最小化的 reference-doc 并注入 AFD 样式
        using var doc = WordprocessingDocument.Create(path, WordprocessingDocumentType.Document);
        var mainPart = doc.AddMainDocumentPart();
        mainPart.Document = new Document(new Body());
        var stylePart = mainPart.AddNewPart<StyleDefinitionsPart>();
        stylePart.Styles = new Styles();

        OpenXmlStyleCorrector.ApplyAfdStyles(path, template);
    }
}

public record ConversionResult
{
    public bool Success { get; init; }
    public string OutputPath { get; init; } = "";
    public string Format { get; init; } = "";
    public string ErrorMessage { get; init; } = "";
}
```

---

## 5. AFD 模板数据模型 (C#)

```csharp
namespace WeaveDoc.Converter.Models;

/// <summary>AFD 模板完整定义</summary>
public record AfdTemplate
{
    public AfdMeta Meta { get; init; } = new();
    public AfdDefaults Defaults { get; init; } = new();
    public Dictionary<string, AfdStyleDefinition> Styles { get; init; } = new();
    public AfdHeaderFooter? HeaderFooter { get; init; }
    public AfdNumbering? Numbering { get; init; }
}

public record AfdMeta
{
    public string TemplateName { get; init; } = "";
    public string Version { get; init; } = "1.0.0";
    public string Author { get; init; } = "";
    public string Description { get; init; } = "";
}

public record AfdDefaults
{
    public string FontFamily { get; init; } = "宋体";
    public double? FontSize { get; init; }         // pt
    public double? LineSpacing { get; init; }       // 倍数
    public AfdPageSize? PageSize { get; init; }
    public AfdMargins? Margins { get; init; }
}

public record AfdPageSize
{
    public double Width { get; init; }     // mm
    public double Height { get; init; }    // mm
}

public record AfdMargins
{
    public double Top { get; init; }       // mm
    public double Bottom { get; init; }
    public double Left { get; init; }
    public double Right { get; init; }
}

public record AfdStyleDefinition
{
    public string? DisplayName { get; init; }
    public string? FontFamily { get; init; }
    public double? FontSize { get; init; }           // pt
    public bool? Bold { get; init; }
    public bool? Italic { get; init; }
    public string? Alignment { get; init; }          // left | center | right | both
    public double? SpaceBefore { get; init; }         // pt
    public double? SpaceAfter { get; init; }          // pt
    public double? LineSpacing { get; init; }         // 倍数
    public double? FirstLineIndent { get; init; }     // pt
    public double? HangingIndent { get; init; }       // pt
}

public record AfdHeaderFooter
{
    public AfdHeaderContent? Header { get; init; }
    public AfdFooterContent? Footer { get; init; }
}

public record AfdHeaderContent
{
    public string Text { get; init; } = "";
    public string? FontFamily { get; init; }
    public double? FontSize { get; init; }
    public string? Alignment { get; init; }
}

public record AfdFooterContent
{
    public bool PageNumbering { get; init; }
    public string Format { get; init; } = "arabic";   // arabic | roman
    public string? Alignment { get; init; }
    public int StartPage { get; init; } = 1;
}

public record AfdNumbering
{
    public AfdHeadingNumbering? HeadingNumbering { get; init; }
    public AfdListStyle? ListStyle { get; init; }
}

public record AfdHeadingNumbering
{
    public string Format { get; init; } = "decimal";
    public string Separator { get; init; } = ".";
    public List<AfdNumberLevel> Levels { get; init; } = new();
}

public record AfdNumberLevel
{
    public string Format { get; init; } = "";
    public string Suffix { get; init; } = "";
}

public record AfdListStyle
{
    public string Bullet { get; init; } = "●";
    public string OrderedFormat { get; init; } = "1,2,3";
}
```

---

## 6. 单位换算速查表

| 源单位 | 目标单位 | 换算公式 | 示例 |
|--------|---------|----------|------|
| pt (磅) | twips (缇) | `pt × 20` | 12pt = 240 twips |
| pt (磅) | half-points | `pt × 2` | 12pt = 24 half-points |
| mm (毫米) | twips (缇) | `mm × 567` | 30mm = 17010 twips |
| cm (厘米) | twips (缇) | `cm × 5670` | 2.54cm = 14401.8 twips |
| inch (英寸) | twips (缇) | `in × 1440` | 1in = 1440 twips |
| em (em) | pt (磅) | `em × fontSize` | 2em @ 12pt = 24pt |
| 字符宽度 | pt (磅) | `char × fontSize` | 2字符 @ 12pt = 24pt |

---

## 7. 常见学术排版规则到 AFD 映射

| 学术排版要求 | AFD 配置 | 对应 OpenXML |
|-------------|---------|-------------|
| 正文宋体小四号 | `fontFamily: "宋体", fontSize: 12` | `<w:rFonts w:eastAsia="宋体"/> <w:sz w:val="24"/>` |
| 标题黑体三号 | `fontFamily: "黑体", fontSize: 16` | `<w:rFonts w:eastAsia="黑体"/> <w:sz w:val="32"/>` |
| 1.5 倍行距 | `lineSpacing: 1.5` | `<w:spacing w:line="360" w:lineRule="auto"/>` |
| 首行缩进两字符 | `firstLineIndent: 24` (at 12pt) | `<w:ind w:firstLine="480"/>` |
| A4 纸张 | `pageSize: {width: 210, height: 297}` | `<w:pgSz w:w="11907" w:h="16839"/>` |
| 上下 2.5cm 页边距 | `margins: {top: 25, bottom: 25}` | `<w:pgMar w:top="1418" w:bottom="1418"/>` |
| 左右 3cm 页边距 | `margins: {left: 30, right: 30}` | `<w:pgMar w:left="1701" w:right="1701"/>` |

---

## 8. 错误处理与调试策略

### 8.1 Pandoc 常见错误

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `pandoc: ...: withBinaryFile: does not exist` | 文件路径错误 | 检查路径是否含中文/空格，使用绝对路径 |
| `Error producing PDF` | 未安装 LaTeX 引擎 | 安装 TeX Live 或 MiKTeX，确保 `xelatex` 可用 |
| `Font ... not found` | 系统缺少指定字体 | 确认字体已安装，或使用 `-V mainfont=` 指定 |
| Unicode decode error | Markdown 文件编码问题 | 确保文件为 UTF-8 编码 |

### 8.2 OpenXML 调试方法

```csharp
// 将 .docx 解压并查看 styles.xml
// 方法 1：直接解压
System.IO.Compression.ZipFile.ExtractToDirectory("output.docx", "debug_output/");

// 方法 2：代码中输出 styles.xml 内容
using var doc = WordprocessingDocument.Open("output.docx", false);
var stylesPart = doc.MainDocumentPart!.StyleDefinitionsPart!;
Console.WriteLine(stylesPart.Styles.OuterXml);
```

### 8.3 AST 调试方法

```bash
# 导出 Pandoc AST JSON 用于分析
pandoc input.md -t json -o ast_debug.json

# 使用 jq 查看特定节点
cat ast_debug.json | jq '.blocks[0]'
```
