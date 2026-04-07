# Pandoc 技术参考手册

> **用途**：供 AI 编程助手读取，作为生成 WeaveDoc Pandoc 转换管道代码的上下文参考。
> **来源**：Context7 /websites/pandoc（8831 代码片段）
> **版本**：Pandoc 3.9.x | **更新日期**：2026-04-06

---

## 1. 概述

Pandoc 是"文档界的瑞士军刀"——一个通用的文档格式转换器。

**对 WeaveDoc 最重要的转换路线**：

```
Markdown ──→ DOCX     （核心：学术论文导出）
Markdown ──→ PDF      （通过 XeLaTeX，中文必须）
Markdown ──→ AST JSON  （调试分析）
```

---

## 2. 安装与本地化（WeaveDoc 方案）

Pandoc 已安装在项目本地 `tools/pandoc-3.9.0.2/pandoc.exe`，不污染系统环境。

```csharp
// C# 中调用时使用项目相对路径
var pandocPath = Path.Combine(AppContext.BaseDirectory, "tools", "pandoc-3.9.0.2", "pandoc.exe");
```

---

## 3. 核心 CLI 用法

### 3.1 Markdown → DOCX

```bash
# 基础转换
pandoc input.md -o output.docx --standalone

# 使用参考文档（继承样式）
pandoc input.md -o output.docx --reference-doc=template.docx

# 使用 Lua Filter
pandoc input.md --lua-filter=filter.lua -o output.docx

# 组合使用
pandoc input.md \
  -f markdown+tex_math_dollars+pipe_tables+raw_html \
  -t docx \
  --reference-doc=template.docx \
  --lua-filter=afd-filter.lua \
  -o output.docx
```

### 3.2 Markdown → PDF（中文支持）

```bash
pandoc input.md \
  -f markdown+tex_math_dollars+pipe_tables \
  --pdf-engine=xelatex \
  -V CJKmainfont="宋体" \
  -V geometry:margin=2.5cm \
  -o output.pdf
```

### 3.3 导出 AST JSON（调试用）

```bash
pandoc input.md -t json -o ast.json
```

### 3.4 生成/获取参考文档

```bash
# 导出 Pandoc 默认的 reference.docx，可在此基础上修改样式
pandoc -o custom-reference.docx --print-default-data-file reference.docx

# 同理导出 ODT 和 PPTX 参考文档
pandoc -o custom-reference.odt --print-default-data-file reference.odt
pandoc -o custom-reference.pptx --print-default-data-file reference.pptx
```

---

## 4. reference-doc 机制（关键）

`--reference-doc` 是 Pandoc DOCX 输出的核心样式控制手段。

### 4.1 工作原理

1. Pandoc 生成 DOCX 时**忽略**参考文档的内容
2. 但**复制**参考文档的以下内容：
   - `styles.xml` 中的所有样式定义
   - 文档属性（页边距、页面大小）
   - 页眉页脚
3. 然后将转换后的内容套用这些样式

### 4.2 最佳实践

1. 先用 `--print-default-data-file reference.docx` 导出默认模板
2. 在 Word 中打开，修改 `Heading 1`、`Normal` 等样式的字体、字号、间距
3. **只修改样式，不要修改文档内容**
4. 保存后在 Pandoc 命令中用 `--reference-doc` 引用

### 4.3 Pandoc 使用的标准样式名

| Markdown 元素 | DOCX 样式 ID | 说明 |
|---|---|---|
| `# 标题` | `Heading1` | 一级标题 |
| `## 标题` | `Heading2` | 二级标题 |
| `### 标题` | `Heading3` | 三级标题 |
| 段落 | `Normal` | 正文（也用 `BodyText`） |
| `> 引用` | `BlockText` | 引用块 |
| 代码块 | `SourceCode` | 围栏代码 |
| `- 列表` | 列表无特定样式 | 通过 numbering.xml 控制 |
| 脚注 | `FootnoteText` | 脚注正文 |
| 图片 | `Figure` / `CaptionedFigure` | 图片容器 |
| 图片说明 | `ImageCaption` | 图片题注 |
| 表格说明 | `TableCaption` | 表格题注 |
| 参考文献 | `Bibliography` | 参考文献条目 |

---

## 5. custom-style 属性（精细控制）

除了修改 `reference.docx`，还可以在 Markdown 中用 `custom-style` 属性直接指定样式：

```markdown
::: {custom-style="First Paragraph"}
这是使用"First Paragraph"样式的段落。
:::

::: {custom-style="Body Text"}
这是普通正文，包含[强调]{custom-style="Emphatic"}文字。
:::
```

**前提**：指定的样式必须在 `reference.docx` 中已定义。

### 5.1 行内 custom-style

```markdown
这是一段文字，其中[这部分用自定义样式]{custom-style="Emphatic"}显示。
```

### 5.2 读取 DOCX 中的自定义样式

```bash
# 从已有 DOCX 反向提取 custom-style 标记
pandoc custom-style-reference.docx -f docx+styles -t markdown
```

---

## 6. Lua Filter（核心扩展机制）

### 6.1 基本结构

```lua
-- 返回一个表，键是 AST 节点类型名，值是处理函数
return {
  Header = function(el)
    -- 处理标题节点
    return el  -- 必须返回元素（修改后的或新的）
  end,

  Para = function(el)
    -- 处理段落节点
    return el
  end
}
```

### 6.2 可用的 AST 节点类型

**块级**：
- `Header` — 标题（`el.level`, `el.content`, `el.attr`）
- `Para` — 段落（`el.content`）
- `CodeBlock` — 代码块（`el.text`, `el.attr.classes[1]` 为语言）
- `BulletList` — 无序列表（`el.content` 为列表项列表）
- `OrderedList` — 有序列表（`el.content`, `el.listAttributes`）
- `BlockQuote` — 引用块（`el.content`）
- `Table` — 表格
- `Div` — 通用块容器（`el.attr`, `el.content`）
- `RawBlock` — 原始格式块（`el.format`, `el.text`）
- `HorizontalRule` — 水平线
- `Null` — 空

**行内**：
- `Str` — 字符串（`el.text`）
- `Emph` — 斜体（`el.content`）
- `Strong` — 粗体（`el.content`）
- `Code` — 行内代码（`el.text`）
- `Space` — 空格
- `SoftBreak` — 软换行
- `LineBreak` — 硬换行
- `Math` — 数学公式（`el.mathtype`, `el.text`）
- `Link` — 链接（`el.content`, `el.target`）
- `Image` — 图片（`el.content`, `el.src`, `el.title`）
- `Span` — 通用行内容器（`el.attr`, `el.content`）
- `RawInline` — 原始行内格式
- `Note` — 脚注
- `Cite` — 引用

### 6.3 属性操作

```lua
function Header(el)
    -- 读取属性
    local id = el.attr.identifier        -- id
    local classes = el.attr.classes       -- {class1, class2}
    local attrs = el.attr.attributes      -- {{key1, val1}, {key2, val2}}

    -- 设置自定义属性
    el.attr.attributes["afd-style"] = "heading" .. el.level

    return el
end
```

### 6.4 实用 Filter 示例

**标题样式标记**：

```lua
-- 标记标题供后续 OpenXML 映射
function Header(el)
    el.attr.attributes["afd-style"] = "heading" .. el.level
    if el.level == 1 then
        el.attr.attributes["alignment"] = "center"
    end
    return el
end
```

**图片题注格式化**：

```lua
function Image(el)
    el.attr.attributes["afd-style"] = "caption"
    return el
end
```

**段落首行缩进标记**：

```lua
function Para(el)
    -- 排除列表项和引用中的段落
    if not el.parent or el.parent.t ~= "ListItem" then
        el.attr.attributes["afd-indent"] = "first-line"
    end
    return el
end
```

---

## 7. Pandoc AST JSON 格式

### 7.1 顶层结构

```json
{
  "pandoc-api-version": [1, 23],
  "meta": {
    "title": { "t": "MetaInlines", "c": [{"t": "Str", "c": "文档标题"}] },
    "author": { "t": "MetaInlines", "c": [{"t": "Str", "c": "张三"}] }
  },
  "blocks": [
    { "t": "Header", "c": [1, ["id", [], []], [{"t": "Str", "c": "标题一"}]] },
    { "t": "Para", "c": [{"t": "Str", "c": "这是正文。"}] }
  ]
}
```

### 7.2 解析 AST JSON 的 C# 代码

```csharp
using System.Text.Json;

public class PandocAstParser
{
    public static void ParseAndPrint(string jsonPath)
    {
        var json = File.ReadAllText(jsonPath);
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        // 读取元数据
        if (root.TryGetProperty("meta", out var meta))
        {
            if (meta.TryGetProperty("title", out var title))
            {
                Console.WriteLine($"标题: {ExtractMetaText(title)}");
            }
        }

        // 遍历块级元素
        if (root.TryGetProperty("blocks", out var blocks))
        {
            foreach (var block in blocks.EnumerateArray())
            {
                var type = block.GetProperty("t").GetString();
                Console.WriteLine($"Block: {type}");
            }
        }
    }

    private static string ExtractMetaText(JsonElement meta)
    {
        // MetaInlines → [{t:"Str", c:"文本"}, ...]
        if (meta.TryGetProperty("c", out var inlines))
        {
            return string.Join("", inlines.EnumerateArray()
                .Where(i => i.TryGetProperty("t", out var t) && t.GetString() == "Str")
                .Select(i => i.GetProperty("c").GetString()));
        }
        return "";
    }
}
```

---

## 8. C# 调用 Pandoc 完整封装

```csharp
using System.Diagnostics;

namespace WeaveDoc.Converter;

public class PandocPipeline
{
    private readonly string _pandocPath;

    public PandocPipeline(string? pandocPath = null)
    {
        _pandocPath = pandocPath ?? "pandoc";
    }

    /// <summary>Markdown → DOCX</summary>
    public async Task<string> ToDocxAsync(
        string inputPath, string outputPath,
        string? referenceDoc = null,
        string? luaFilter = null,
        CancellationToken ct = default)
    {
        var args = new List<string>
        {
            inputPath,
            "-f", "markdown+tex_math_dollars+pipe_tables+raw_html",
            "-t", "docx",
            "-o", outputPath,
            "--standalone"
        };

        if (referenceDoc != null)
        {
            args.AddRange(new[] { "--reference-doc", referenceDoc });
        }
        if (luaFilter != null)
        {
            args.AddRange(new[] { "--lua-filter", luaFilter });
        }

        return await RunAsync(args, ct);
    }

    /// <summary>Markdown → PDF（XeLaTeX 中文支持）</summary>
    public async Task<string> ToPdfAsync(
        string inputPath, string outputPath,
        CancellationToken ct = default)
    {
        var args = new List<string>
        {
            inputPath,
            "-f", "markdown+tex_math_dollars+pipe_tables",
            "--pdf-engine", "xelatex",
            "-V", "CJKmainfont=宋体",
            "-o", outputPath
        };

        return await RunAsync(args, ct);
    }

    /// <summary>导出 AST JSON</summary>
    public async Task<string> ToAstJsonAsync(
        string inputPath, CancellationToken ct = default)
    {
        var args = new List<string> { inputPath, "-t", "json" };
        return await RunAsync(args, ct);
    }

    private async Task<string> RunAsync(List<string> args, CancellationToken ct)
    {
        var psi = new ProcessStartInfo
        {
            FileName = _pandocPath,
            Arguments = string.Join(" ", args.Select(a =>
                a.Contains(' ') ? $"\"{a}\"" : a)),
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true
        };

        using var process = Process.Start(psi)!;
        var stdout = process.StandardOutput.ReadToEndAsync(ct);
        var stderr = process.StandardError.ReadToEndAsync(ct);
        await process.WaitForExitAsync(ct);

        if (process.ExitCode != 0)
            throw new Exception($"Pandoc error: {await stderr}");

        return await stdout;
    }
}
```

---

## 9. 参考链接

- **官网**：https://pandoc.org
- **Lua Filter 文档**：https://pandoc.org/lua-filters.html
- **Manual**：https://pandoc.org/MANUAL.html
- **GitHub**：https://github.com/jgm/pandoc
- **Context7 ID**：`/websites/pandoc`
- **本地安装**：`tools/pandoc-3.9.0.2/pandoc.exe`
