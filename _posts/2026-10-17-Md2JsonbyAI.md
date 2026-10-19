---
title: "Markdown存入Neo4j构建图谱"
date: 2026-10-19
categories: [AI]
tags: [Markdown, JSON, Neo4j, Cypher]
---

## 📌 导图
对之前的文章--[解析清洗数据源（PDF行业文档）]({% post_url 2026-10-12-EditPDFbyPy %})里面【2. 文本层外挂】拆解：
![alt text](/assets/images/Md2JsonbyAI/image.png)

### 1. MD转JSON（AI）
#### 1.1 PDF元数据提取
  - 主要利用大模型结合文件名和正文，提取文件名、标题、年份、发布机构、地域范围。
#### 1.2 深度挖掘MD全文
  - 约束大模型归纳固定了枚举范围：文档类型、行业分类（粗、细粒度），尤其针对核心摘要和洞察，必须是高价值干货。

上面两部分的实现和探讨，可以参看文章--[探索如何给PDF文档分类]({% post_url 2026-10-15-ClassifybyAI %})

#### 1.3 JSON标签记录
  - 按照JSON格式化输出提取&挖掘的信息，并记录PDF归档后的绝对路径。（归档文件夹=PDF文件类型）

    <details markdown="1">
    <summary>👉 点击查看这一部分代码</summary>

    ```python
    # ---------- 归档搬运 ----------
    def archive_pdf(pdf_name, doc_type):
        """搬运 PDF，返回绝对路径。找不到文件返回 None"""
        src = PDF_SOURCE_DIR / pdf_name
        print(f"Archive PDF： {src} \n")

        if not src.exists():
            return None

        dest_dir = ARCHIVE_DIR / doc_type
        dest_dir.mkdir(parents=True, exist_ok=True)
        dest = dest_dir / pdf_name

        if not dest.exists():
            shutil.move(str(src), str(dest))

        return str(dest.absolute())
    ```
</details>

### 2. JSON 文件转 图数据库
#### 2.1 构建节点和关系
- 一条记录进入图数据库，会变成一个“中心节点 + 多个周边节点 + 多条连接线”的星型结构。
按照前面输出的JSON体里面的标签，可以归纳为：
    ```
    🔔中心节点：P_Document（1个）
    🔔分类节点：P_DocType、P_IndustryGroup、P_Industry、P_Publisher（4个）
    🔔关键词节点：每个keywords都要作为1个节点（>=5个）
    🧲关系：中心节点->分类节点（4条），中心节点->关键词节点（>=5条）
    💳属性：剩余的字段都是中心节点的属性。
    ```

#### 2.2 数据导入Neo4j
- 写代码（neo4jdataimport.py）,读取JSON文件，使用CYPHER语句导入数据。

    <details markdown="1">
    <summary>👉 点击查看代码</summary>

    ```python
    # ---------- Cypher 语句 ----------
    # 统一使用 P_ 作为节点标签前缀
    CYPHER_QUERY = """
    MERGE (d:P_Document {p_file: $p_file})
    SET d.p_title = $p_title,
        d.p_year = $p_year,
        d.p_region = $p_region,
        d.p_summary = $p_summary,
        d.p_key_findings = $p_key_findings,
        d.p_pdf_path = $p_pdf_path,
        d.p_md_file = $p_md_file

    // 1. 关联文档类型
    MERGE (dt:P_DocType {p_name: $p_doc_type})
    MERGE (d)-[:BELONGS_TO_TYPE]->(dt)

    // 2. 关联行业大类
    MERGE (ig:P_IndustryGroup {p_name: $p_industry_group})
    MERGE (d)-[:BELONGS_TO_GROUP]->(ig)

    // 3. 关联细粒度行业
    MERGE (ind:P_Industry {p_name: $p_industry})
    MERGE (d)-[:BELONGS_TO_INDUSTRY]->(ind)

    // 4. 关联发布机构 (防止因 null 导致建节点失败)
    WITH d
    FOREACH (ignoreMe IN CASE WHEN $p_publisher IS NOT NULL AND $p_publisher <> '' THEN [1] ELSE [] END |
        MERGE (p:P_Publisher {p_name: $p_publisher})
        MERGE (d)-[:PUBLISHED_BY]->(p)
    )

    // 5. 关联关键词
    WITH d
    UNWIND $p_keywords AS kw_name
    MERGE (k:P_Keyword {p_name: kw_name})
    MERGE (d)-[:HAS_KEYWORD]->(k)
    """
    ```

</details>

- 为了写入速度和绝对防止重复，启动Docker里面Neo4j，然后打开Neo4j Browser，先跑这四句：

    ```cypher
    CREATE CONSTRAINT FOR (d:P_Document) REQUIRE d.p_file IS UNIQUE;
    CREATE CONSTRAINT FOR (k:P_Keyword) REQUIRE k.p_name IS UNIQUE;
    CREATE CONSTRAINT FOR (i:P_Industry) REQUIRE i.p_name IS UNIQUE;
    CREATE CONSTRAINT FOR (p:P_Publisher) REQUIRE p.p_name IS UNIQUE;
    ```

- 数据导入Neo4j后，打开Browser，可以看到下面爆炸的“震撼感”，中心节点+关系线+聚合效应都体现了。下一步，是不是就可以让Agent上岗了？😎
![alt text](/assets/images/Md2JsonbyAI/image-1.png)

## 📝 总结（隐形工程）

### 🛴MD转JSON
#### 1. 智能截断非常重要（省 Token，防超限）
- 如果 MD 有几万字，全塞进去会报错。读 MD 时自动截取“前 8000 字 + 结尾 2000 字”。开头看背景，结尾看结论，中间直接省略。其实，还可以采取：首 6000 字 + 中间 3000 字 + 尾 2000 字 的智能截断策略，估计效果差不多。
#### 2. 断点续传必不可少（防崩溃白干）
- 对于批量 MD 文件连续处理的场景，万一文档很长或断网或不可预知的错误，要在每处理 5 份就把结果存到 .temp_extracted.json里。这样即使重启任务，也能自动跳过已处理的 MD 文件。
#### 3. JSON 强制清洗
- 长文本下 AI 极容易在 JSON 前后加一句“好的，这是提取结果”。clean_json_string 会自动抠出 { ... } 的部分，防止解析崩溃。
#### 4. 模型的升级与选型
- 一开始用了免费的 glm-4-flash，发现摘要很水，没有数据和具体建议，全都是“探讨了”“分析了”这样的废话，关键字和洞察更是格式混乱、简单得只剩几个字；调整成下面的提示词并切换到 glm-5.3-flash后，质量显著提高。
    ```Python
    - p_summary: 150-300字的干货摘要。必须包含：文档的核心观点、具体的行业数据（如有）、最终的结论。禁止使用“探讨了”、“分析了”、“提出了”等空泛词汇，直接用陈述句说出它到底讲了什么。（英文文档需翻译后再归纳概括）
    - p_key_findings: 3-5 个关键核心结论，必须是具体的事实或结论，不要写套话。（字符串数组，每条不超过30字，中文。英文文档需翻译后归纳）
    ```

### 🛴JSON 文件转 图数据库
#### 1. 图谱思维很重要（从“孤岛”到“网络”的质变）
- “查询的起点”或“独立的检索维度”，用来做分类、共享、找同类的，应该抽离出来作为节点。例如：关键词、机构是独立的主体。
- 对那种有一对一的专有信息的标签，应该作为属性。例如：p_summary（摘要）和 p_title（标题）只属于这一份文档，绝不会共享。
- 仅仅就是一个字符串（比如“页码”），只能作为文档节点的属性。
- 如果某个标签还有它的附属属性（比如“行业”有“行业大类”的属性），应该作为一个节点。
- 可能需要深挖AI归纳的Summary或Keywords等信息时，里面提到的实体也有必要作为节点。
#### 2. 多层级标签隔离
- Neo4j的社区版不分Schema，节点和关系都是平铺在库里面，给节点加专属前缀标签来与本地原有其他图数据，进行物理隔离，互不干扰。从而，倒推到JSON生成时Prompt里面就写好专属前缀。