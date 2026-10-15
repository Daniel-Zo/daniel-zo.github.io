---
title: "解析清洗数据源（PDF行业文档）"
date: 2026-10-12
categories: [AI]
tags: [Python, PyMuPDF, OpenCV]
---

## 📌 导图
### 1. 摘除广告页
- 广告内容不仅影响阅读原文，也污染索引查询，通过关键字和广告模板来命中，进行PDF删页处理。
![alt text](/assets/images/EditPDFbyPy/image.png)

### 2. 文本层外挂
- OCR 错误的话，虽然回写透明文本层，而且也不会污染用户看到的内容。但是，确实影响检索准确性。
所以，OCR 识别的文本和原本PDF的文本，都进行外挂处理，建立PDF->MD->JSON的关系（三者用同一个文件名前缀关联）
```
原始 PDF（不动）
    ↓ MinerU 解析
Markdown（外挂文本）
    ↓ 抽取字段
JSON（结构化标签）
    ↓ 写入
Neo4j（节点 + 关系）
    ↓ 检索时
查 Neo4j / 查 MD，不查 PDF
```

MinerU 解析：参考前面的文章-[实践大模型Mineros]({% post_url 2026-10-10-1stDay %})

## 📌 代码实现
### 1. Conda环境准备
启动Conda下的ExamRGA环境，并安装pymupdf和opencv库。
- PyMuPDF 负责“打开 PDF、读取页面、提取图片/文本，转 Markdown等”。
- OpenCV 负责“拿到图片后做去水印、二值化、裁剪、OCR 预处理等图像处理”。
```
pip install pymupdf opencv-python pillow
```

### 2. Python代码实现
代码也就300行，借助Deepseek多次调整修改，一天时间搞定。🎉🎉🎉
部分代码如下图：
![alt text](/assets/images/EditPDFbyPy/image-1.png)

## 📝 总结（跨过了多少坑）
### 1. 灰度图Size归一化
- 广告模板页尺寸（横版）和其他正常页（有些文档是竖版）尺寸不一致，导致PDF每页的渲染图全都和模板不一致。改为对每页都在渲染灰度图后，用cv2.resize进行归一化到相同尺寸了再比较

### 2. 模板匹配的阈值设定
- 随PDF的版式变化，取opencv返回的最佳匹配点max_val进行阈值比较的话，无论阈值怎么设定，都存在与模板图不匹配的风险。改为增加关键词命中的匹配逻辑。模板匹配 or 关键词命中，then判定为嫌疑页。

<details>
<summary>👉 点击查看完整代码</summary>

```python
# ---------- 检测疑似页 ----------

def find_suspect_pages(pdf_path, template_path, ad_keywords, threshold=0.25, dpi=150):
    """
    逐页渲染、归一化，与模板图做匹配 + 关键词双重锁定，
    返回 (疑似广告页页码列表, 每页得分字典)，页码 0-indexed。
    """
    template = cv2.imread(str(template_path), cv2.IMREAD_GRAYSCALE)
    if template is None:
        raise FileNotFoundError(f"模板图读取失败：{template_path}")
    template = normalize(template)
    th, tw = template.shape[:2]

    doc = fitz.open(pdf_path)
    suspect_pages = []
    scores = {}
    for page_num in range(doc.page_count):
        page = doc[page_num]

        # --- 条件 1：模板匹配 ---
        pix = page.get_pixmap(dpi=dpi, colorspace=fitz.csGRAY)
        img = np.frombuffer(pix.samples, dtype=np.uint8).reshape(pix.h, pix.w)
        img = normalize(img)

        if th > img.shape[0] or tw > img.shape[1]:
            scores[page_num] = 0.0
            continue

        result = cv2.matchTemplate(img, template, cv2.TM_CCOEFF_NORMED)
        _, score, _, _ = cv2.minMaxLoc(result)
        scores[page_num] = score

        # --- 条件 2：关键词 ---
        text = page.get_text()
        hit_keyword = any(kw in text for kw in ad_keywords)

        # --- 双重锁定 ---
        if score >= threshold or hit_keyword:
            suspect_pages.append(page_num)

    doc.close()
    return suspect_pages, scores
```

</details>



