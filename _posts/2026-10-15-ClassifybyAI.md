---
title: "探索如何给PDF文档分类"
date: 2026-10-15
categories: [AI]
tags: [PDF, Python, 文档分类, ]
---

## 📌 场景需求还原
在文档处理场景中，PDF 自动分类是一个很常见的需求。每天收集到的PDF 分为原生文本型和图片扫描型，或者图文表格都有，各种模型的处理长短板都不尽相同。暂且按照下面的路径一步一步探索：
![alt text](/assets/images/ClassifybyAI/image-1.png)

### 1. Markdown文档生成
写Python程序(minuertrail.py)，调用mineros命令，对135份PDF文件的前5页内容解析，生成MD。

<details markdown="1">
<summary>👉 点击查看完整代码</summary>

```python
import subprocess
from pathlib import Path
import shutil

src_dir = Path(r"E:\minerU-trail\input")
out_dir = Path(r"E:\minerU-trail\output")
out_dir.mkdir(parents=True, exist_ok=True)
md_out_dir = Path(r"E:\minerU-trail\md_page0-5")
md_out_dir.mkdir(parents=True, exist_ok=True)

pdf_files = sorted(src_dir.glob("*.pdf"))
print(f"共找到 {len(pdf_files)} 个 PDF")

for i, pdf in enumerate(pdf_files, 1):
    print(f"\n[{i}/{len(pdf_files)}] 处理：{pdf.name}")
    subprocess.run(
        [
            "mineros",
            "-p", str(pdf),
            "-o", str(out_dir),
            "-b", "pipeline",
            "-m", "ocr",
            "-l", "ch",
            "-s", "0",
            "-e", "4",     # 只解析前 5 页
        ],
        check=True
    )

md_count = 0
# 遍历新生成的所有 .md 文件
for md_file in out_dir.glob("*.md"):
    # 移动文件到新文件夹，如果重名则替换 (replace=True)
    shutil.move(str(md_file), str(md_out_dir / md_file.name))
    md_count += 1

print(f"\n已成功将 {md_count} 个 Markdown 文件移动至: {md_out_dir}")
print("\n全部完成")
```

</details>

### 2. AI归纳doc_type
- 从文档名称就能大致判断出来是什么文档类型，所以，简单做个一次性的程序让AI根据文件名直接归纳几个Doc_type。再人工收敛并固定下来，就可以建Archive文件夹的子目录了。
- 在写提示词时，为了不让AI直接抄我的举例，又不能让它放飞自由发挥，就写了几个规则：
```
【重要规则】
1. 类别必须是"文档类型"，反映文档的性质和用途，
   不要按行业领域分（如汽车、AI、环保），也不要按主题分。
2. 同义的类别必须合并。比如"行业分析""行业报告""产业报告"
   是同一类，只能保留一个。
3. **最多输出 8 个类别**，宁少勿多。
4. 每个类别给一个 3-8 字的中文名称。
5. 只输出类别列表，每行一个，不要解释，不要编号。
```
- 执行classifybyzhipu-doctype.py后，跑出来结果：

```
共 135 份文件
正在调用智谱 AI 归纳类别...
=== AI 归纳的文档类型 ===
  1. 研究报告
  2. 白皮书
  3. 市场洞察
  4. 行业分析
  5. 趋势报告
  6. 发展报告
  7. 技术分析
  8. 案例研究
已保存：E:\MinerU-trail\classification_v1.json
```

### 3. 人工收敛doc_type
从AI归纳的结果看，很明显文档类型和内容视角混到一起了，市场洞察和行业分析属于内容视角的维度了。
只好我自己人工进行收敛为下面5个，这样互斥清晰。今后AI自动分类就按这个固定list来。
```python
FIXED_DOC_TYPES = [
    "研究报告",    # 含行业研究、市场研究、调研
    "白皮书",      # 含各类白皮书
    "趋势分析",    # 含技术趋势、发展趋势
    "案例研究",    # 含案例分析
    "其他",        # 政策、指南、杂项
]
```

### 4. 按固定List重新分类/抽取字段
- 调整Prompt，新做一个Python程序（classifybyzhipu.py），对这135份MD文件逐一喂给AI。
- AI从MD文件中抽取原始PDF文件的基本元数据（名称、标题、年份、发布机构），另外，还需要从每个MD文件的前5页文字中归纳出：行业、关键词。

```python
    type_list = "\n".join(f"- {t}" for t in FIXED_DOC_TYPES)

    prompt = f"""请根据下面的文档信息，抽取字段并输出 JSON。

文档文件名：{filename}

文档开头内容：
{md_head}

请输出以下字段：

- file: 文件名（原样返回）
- title: 完整标题。请保留原文件名里的所有核心信息，只去掉明显的冗余修饰（如重复的年份、发布机构、无关的后缀）。不要压缩成短语。
- year: 年份（整数，找不到填 null）
- doc_type: 文档类型，**必须从下面 5 个里选一个**，不要自己造词：
{type_list}
  判断依据：
  - 含"白皮书" → 白皮书
  - 含"趋势" → 趋势分析
  - 含"案例" → 案例研究
  - 含"研究""报告""调研""分析" → 研究报告
  - 都不符合 → 其他
- industry: 主行业领域（用你判断最准确的中文名称，只选最主要的一个，自由生成）
- publisher: 发布机构（找不到填 null）
- keywords: 5-8 个核心关键词（字符串数组）

要求：
- 只输出 JSON 对象，不要解释，不要代码块标记
- 不确定的字段填 null，不要编造
```

- 执行classifybyzhipu.py后，135份文档的doc_type分类正确，但是industry出现"长尾分布"：少数领域（人工智能 14 份）集中，剩下 100 多个领域各 1 份。Keywords因为只喂了5页文本而归纳得乱七八糟。
暂且保留原始 industry，另加一个固定list的粗粒度的 industry_group。这样今后分级检索起来也可以更精准。

```python
INDUSTRY_GROUPS = [
    "科技与互联网",    # AI、芯片、半导体、量子、云计算、网络安全
    "制造业与工业",    # 工业自动化、机器人、激光雷达、半导体设备、电子元器件、智能制造装备
    "消费与零售",      # 电商、零售、餐饮、食品饮料、美容个护、宠物
    "医疗与健康",      # 医疗器械、AI制药、健康保险、口腔、远程医疗
    "金融与保险",      # 金融科技、财富管理、资本市场、租赁、支付
    "能源与环保",      # 能源、储能、锂电、环保、循环经济
    "交通与物流",      # 汽车、新能源汽车、低空经济、无人机、航运、物流、跨境电商、智能座舱、交通传媒
    "文化传媒",        # 电竞、游戏、大文娱、广告营销、数字营销
    "农业与食品",      # 海洋渔业、食品、白酒、健康食品
    "企业服务",        # 人力资源、企业数字化、法律、管理咨询
    "其他",            # 兜底
]
```

- 接下来写代码（appendIndustryGroup.py），直接把Json文件给AI追加粗粒度的 industry_group的值。执行后输出如下。看上去分布是均衡的，可以相信AI可以按照给定的规则进行准确的映射。

```
=== industry_group 分布 ===
  科技与互联网: 28
  消费与零售: 22
  企业服务: 21
  金融与保险: 15
  医疗与健康: 11
  能源与环保: 10
  文化传媒: 10
  制造业与工业: 9
  交通与物流: 8
  农业与食品: 1
```

- 至此，PDF->MD->JSON的workflow总算手工串联起来了。虽然只是试跑了135份PDF文档的前5页内容，但是，doc_type和industry_group的list固定下来了。
- 后面正式程序再照葫芦画瓢，喂全量文本给AI提取元信息和归纳摘要等信息。全量文本的MD文件生成，只需要去掉下图这两行就可以。

![alt text](/assets/images/ClassifybyAI/image.png)

## 📝 总结（跨过了多少坑）
### 1. Prompt写规则非常重要。
- AI大模型擅长的是基于海量文本规律做语言层面的生成、归纳、转换、推理模拟。所以，其实很容易出现幻觉，而且自带 “讨好倾向”，顺着人输入的信息来回答，根本没有严谨的逻辑引擎。这样就只能调Prompt约束AI。
- 在对文档分类时，到底是文档类型优先（如：市场分析报告、产品说明书 etc.），还是行业领域优先（如：汽车智能座舱、生鲜电商 etc.），很难取舍定夺。后来考虑到PDF最终也是要归档的，那就一定要有互斥的角度来约束AI自动分类。简化为直接让AI使用这135份PDF的文件名来归纳建议都有哪些文档分类。至于行业领域则会随视角而无法绝对的互斥统一，其实是可以通过Prompt约束AI归纳出不同粗细颗粒度的特征字段来标识。

### 2. 分步探索
- 前5页内容给到AI归纳行业领域，无论是百花齐放的industry还是粗颗粒的industry_group，其实都是试验探索一个大致的分布情况。如果结果还行，暂且按照目前固定的分类执行。隔一个月看看分类不准确需要重新调整Prompt的话，再调即可。
  
### 3. 让AI专注做它擅长的事
- 无论用多少份MD喂给AI来归纳doc_type、industry都不能完全准确。其实，人一眼就能分辨标识出来。所以可以说，要用AI代替人的话，一定要规则清晰。
- 将所有页转MD文件后的全量文本喂给AI，就是为了让它能专注写Summary、Key_findings并提取Keywords。但是全文太长（例如1万字以上），也还是需要智能截断，中间加上...，防止token超限。


