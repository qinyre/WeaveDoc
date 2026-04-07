# Markdig 技术参考手册

> **用途**：供 AI 编程助手读取，作为生成 WeaveDoc Markdown 解析模块代码的上下文参考。
> **来源**：Context7 /xoofx/markdig（751 代码片段）
> **版本**：Markdig 0.39.x | **更新日期**：2026-04-06

---

## 1. 概述

Markdig 是 .NET 平台上最流行的 Markdown 处理库，特点是：

- **高性能**：比 CommonMark.NET 快 10 倍以上
- **CommonMark 兼容**：严格遵循 CommonMark 规范
- **可扩展管道**：通过 `MarkdownPipeline` 灵活组合扩展
- **完整 AST**：解析后提供可遍历的文档对象模型

---

## 2. 快速开始

### 2.1 NuGet 安装

```xml
<PackageReference Include="Markdig" Version="0.39.1" />
```

### 2.2 基础用法

```csharp
using Markdig;

// 最简单：直接转 HTML
string html = Markdown.ToHtml("# Hello World");

// 带扩展管道
var pipeline = new MarkdownPipelineBuilder()
    .UseAdvancedExtensions()
    .Build();
string html = Markdown.ToHtml(markdownText, pipeline);
```

---

## 3. Pipeline 构建

### 3.1 使用所有高级扩展

```csharp
var pipeline = new MarkdownPipelineBuilder()
    .UseAdvancedExtensions()  // 包含：表格、脚注、引用、数学公式、任务列表、图表等
    .Build();
```

`UseAdvancedExtensions()` 等价于手动启用以下扩展：
- `UsePipeTables()` — 管道表格
- `UseGridTables()` — 网格表格
- `UseFootnotes()` — 脚注
- `UseCitations()` — 引用 `[@key]`
- `UseEmphasisExtras()` — 高级强调（下划线、删除线等）
- `UseGenericAttributes()` — 通用属性 `{#id .class key=value}`
- `UseAutoIdentifiers()` — 自动生成标题 ID
- `UseTaskLists()` — 任务列表 `- [x]`
- `UseListExtras()` — 列表扩展（字母编号等）
- `UseMathematics()` — 数学公式 `$...$` / `$$...$$`
- `UseEmojiAndSmiley()` — Emoji 支持
- `UseYamlFrontMatter()` — YAML 头部

### 3.2 精细控制扩展

```csharp
var pipeline = new MarkdownPipelineBuilder()
    .UseCitations()       // 学术引用
    .UseFootnotes()       // 脚注
    .UsePipeTables()      // 表格
    .UseMathematics()     // 数学公式
    .UseYamlFrontMatter() // YAML 元数据
    .Build();
```

### 3.3 WeaveDoc 推荐管道配置

```csharp
// 针对学术文档场景的推荐配置
var pipeline = new MarkdownPipelineBuilder()
    .UseAdvancedExtensions()
    .UseYamlFrontMatter()     // 论文元数据
    .UseFootnotes()           // 脚注引用
    .UseCitations()           // BibTeX 引用
    .UseMathematics()         // LaTeX 公式
    .UsePipeTables()          // 表格
    .UseGenericAttributes()   // 自定义属性（用于 AFD 样式标记）
    .UseAutoIdentifiers()     // 标题自动编号
    .Build();
```

---

## 4. AST 结构与遍历

### 4.1 解析为 AST 对象

```csharp
var document = Markdown.Parse(markdownText, pipeline);
// document 类型：MarkdownDocument（继承自 ContainerBlock）
```

### 4.2 核心块级节点（Block）

| Markdig 类型 | Markdown 语法 | 说明 |
|---|---|---|
| `HeadingBlock` | `# H1` ~ `###### H6` | 标题，`Level` 属性表示级别 |
| `ParagraphBlock` | 普通文本 | 段落 |
| `CodeBlock` | `` ```code``` `` | 围栏代码块，`Info` 属性为语言标识 |
| `FencedCodeBlock` | `` ```lang`` | 围栏代码块（更常用） |
| `ListBlock` | `- item` / `1. item` | 列表容器 |
| `ListItemBlock` | 列表中的单个项 | 列表项 |
| `QuoteBlock` | `> text` | 引用块 |
| `TableBlock` | `| a | b |` | 管道表格 |
| `TableRow` | 表格行 | |
| `TableCell` | 表格单元格 | |
| `ThematicBreakBlock` | `---` / `***` | 水平分隔线 |
| `HtmlBlock` | `<div>...</div>` | 原始 HTML 块 |
| `YamlFrontMatterBlock` | `---\nyaml\n---` | YAML 头部 |

### 4.3 核心行内节点（Inline）

| Markdig 类型 | Markdown 语法 | 说明 |
|---|---|---|
| `LiteralInline` | 普通文本 | 原始文本内容 |
| `EmphasisInline` | `*italic*` / `**bold**` | 强调，`DelimiterCount` 区分粗/斜体 |
| `CodeInline` | `` `code` `` | 行内代码 |
| `LinkInline` | `[text](url)` | 链接，`IsImage` 属性区分图片 |
| `AutolinkInline` | `<http://...>` | 自动链接 |
| `LineBreakInline` | 行尾两个空格或 `\` | 换行 |
| `SoftBreakInline` | 普通换行 | 软换行 |
| `HtmlInline` | `<br>` 等 | 行内 HTML |
| `HtmlEntityInline` | `&amp;` | HTML 实体 |
| `MathInline` | `$formula$` | 行内数学公式 |
| `DelimiterInline` | `*`、`**` 等 | 分隔符 |
| `PipeTableDelimiterInline` | `\|` | 表格管道分隔符 |

### 4.4 遍历所有节点

```csharp
// 深度优先遍历所有节点
foreach (var item in document.Descendants())
{
    Console.WriteLine(item.GetType().Name);
}
```

### 4.5 按类型过滤遍历

```csharp
// 只遍历标题
foreach (var heading in document.Descendants<HeadingBlock>())
{
    int level = heading.Level;  // 1-6
    // 获取标题文本
    var text = heading.Inline?.Descendants<LiteralInline>()
        .Select(l => l.Content.ToString());
    Console.WriteLine($"H{level}: {string.Join("", text)}");
}

// 只遍历图片链接
foreach (var img in document.Descendants<LinkInline>().Where(x => x.IsImage))
{
    string url = img.Url;       // 图片路径
    string alt = img.FirstChild?.ToString();  // alt 文本
}

// 只遍历行内代码
foreach (var code in document.Descendants<CodeInline>())
{
    string content = code.Content.ToString();
}

// 只遍历数学公式
foreach (var math in document.Descendants<MathInline>())
{
    string formula = math.Content.ToString();
}
```

### 4.6 复合查询

```csharp
// 查找列表项中的所有强调
var items = document.Descendants<ListItemBlock>()
    .Select(block => block.Descendants<EmphasisInline>());

// 查找父级是引用块的所有强调
var quoted = document.Descendants<EmphasisInline>()
    .Where(inline => inline.ParentBlock is QuoteBlock);
```

---

## 5. 自定义扩展

### 5.1 IMarkdownExtension 接口

```csharp
public interface IMarkdownExtension
{
    void Setup(MarkdownPipelineBuilder pipeline);    // 解析阶段
    void Setup(MarkdownPipeline pipeline, IMarkdownRenderer renderer);  // 渲染阶段
}
```

### 5.2 最小自定义扩展示例

```csharp
public class MyExtension : IMarkdownExtension
{
    public void Setup(MarkdownPipelineBuilder pipeline)
    {
        // 注册自定义 Block 解析器
        pipeline.BlockParsers.AddIfNotAlready<MyBlockParser>();
    }

    public void Setup(MarkdownPipeline pipeline, IMarkdownRenderer renderer)
    {
        // 注册自定义渲染器（可选）
    }
}

// 使用
var pipeline = new MarkdownPipelineBuilder()
    .Use<MyExtension>()
    .Build();
```

### 5.3 扩展现有解析器（以 EmphasisInline 为例）

```csharp
public class BlinkExtension : IMarkdownExtension
{
    public void Setup(MarkdownPipelineBuilder pipeline)
    {
        var parser = pipeline.InlineParsers.FindExact<EmphasisInlineParser>();
        if (parser is not null && !parser.HasEmphasisChar('%'))
        {
            // 注册 %%%text%%% 语法
            parser.EmphasisDescriptors.Add(
                new EmphasisDescriptor('%', 3, 3, false));
        }
    }

    public void Setup(MarkdownPipeline pipeline, IMarkdownRenderer renderer)
    {
        if (renderer is not HtmlRenderer htmlRenderer) return;

        var emphasisRenderer = htmlRenderer.ObjectRenderers
            .FindExact<EmphasisInlineRenderer>();
        if (emphasisRenderer is null) return;

        var previousTag = emphasisRenderer.GetTag;
        emphasisRenderer.GetTag = inline =>
            (inline.DelimiterCount == 3 && inline.DelimiterChar == '%'
                ? "blink" : null)
            ?? previousTag(inline);
    }
}
```

---

## 6. 提取纯文本

```csharp
// 从 MarkdownDocument 提取纯文本
public static string ExtractPlainText(MarkdownDocument doc)
{
    var sb = new StringBuilder();
    foreach (var block in doc)
    {
        if (block is ParagraphBlock para)
        {
            sb.AppendLine(para.Inline?.Descendants<LiteralInline>()
                .Aggregate(new StringBuilder(), (s, l) => s.Append(l.Content))
                .ToString());
        }
        else if (block is HeadingBlock heading)
        {
            sb.AppendLine(heading.Inline?.Descendants<LiteralInline>()
                .Aggregate(new StringBuilder(), (s, l) => s.Append(l.Content))
                .ToString());
        }
    }
    return sb.ToString();
}
```

---

## 7. WeaveDoc 场景映射

| WeaveDoc 需求 | Markdig 用法 |
|---|---|
| 解析 Markdown 得到结构化数据 | `Markdown.Parse(text, pipeline)` |
| 提取所有标题层级 | `document.Descendants<HeadingBlock>()` |
| 识别图片和链接 | `document.Descendants<LinkInline>()` |
| 提取脚注内容 | `document.Descendants<FootnoteBlock>()` |
| 解析 YAML 元数据（标题、作者等） | `document.Descendants<YamlFrontMatterBlock>()` |
| 识别数学公式 | `document.Descendants<MathInline>()` + `MathBlock` |
| 将 AST 转为自定义 IR | 遍历 + 映射到 WeaveDoc 自定义数据结构 |

---

## 8. 参考链接

- **GitHub**：https://github.com/xoofx/markdig
- **文档**：https://github.com/xoofx/markdig/tree/master/doc
- **NuGet**：https://www.nuget.org/packages/Markdig
- **Context7 ID**：`/xoofx/markdig`
