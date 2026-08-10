---
layout: post
title: RAG 完整学习笔记：基础原理、检索公式与工程流程
date: 2026-08-10 10:00:00 +0800
description: 从基础原理、检索公式到工程流程的系统化 RAG 学习笔记。
tags: [RAG, retrieval, notes]
categories: [notes]
related_posts: false
toc:
  beginning: true
---

<aside>
🧭

这份笔记从基础原理出发，说明 RAG 每一步为什么存在、输入输出是什么、内部怎样运行、常见公式是什么，以及不同技术适合什么场景。默认排序中带 ✅ 的方案，是大多数项目更适合作为第一版基线的方案，但最终仍要用自己的数据评估。

</aside>

# 0. RAG 总体原理与完整流程

RAG（Retrieval-Augmented Generation，检索增强生成）的核心思想是：**不要只依赖大模型参数中的旧知识，而是在回答前从外部知识库中取回证据，再让 LLM 基于证据回答。**

RAG 主要解决：

- 企业私有资料无法直接进入通用大模型的问题。
- 知识经常更新，重新训练模型成本太高的问题。
- 大模型可能编造事实，答案缺少来源的问题。
- 不同用户只能读取自己有权限资料的问题。

## 0.1 离线知识库构建流程

数据源

→ 数据加载与解析

→ 数据清洗

→ 结构识别

→ 文本切块

→ 添加 Metadata 和权限

→ Embedding 向量化

→ 写入向量数据库、全文索引或其他数据库

→ 建立 ANN 索引

→ 质量检查和版本发布

离线阶段的目标是：把原始文件变成**可以被快速检索、可以追踪来源、可以进行权限过滤**的 Chunk。

## 0.2 在线问答流程

用户问题

→ 查询理解与改写

→ 选择检索通道

→ 关键词、向量、图或 SQL 检索

→ 合并和去重候选结果

→ Rerank 重排

→ 邻居扩展与上下文压缩

→ Prompt 组装

→ LLM 生成

→ 引用检查、事实检查与输出

在线阶段的目标是：在较短时间内找到足够正确的证据，并把证据放进适合 LLM 阅读的上下文。

## 0.3 RAG 系统的三个核心质量

1. **召回率**：正确资料有没有被找到。
2. **排序质量**：最相关资料有没有排在最前面。
3. **生成忠实度**：LLM 有没有只根据证据回答。

---

# 1. 数据加载与解析（Data Loading）

## 1.1 数据加载的作用

数据加载不是简单的“读取文字”。它需要把 PDF、Word、网页、表格、图片等转换成统一的 Document 结构，并尽量保留：

- 标题层级
- 段落顺序
- 页码和坐标
- 表格行列关系
- 图片和图注
- 来源 URL
- 创建时间、作者、版本和权限

如果解析阶段丢失结构，后面的切块、检索和引用都会受到影响。

## 1.2 不同输入数据的处理

### 1. PDF

PDF 分成两类：

- **文本型 PDF**：内部已经保存文字，可以直接抽取。
- **扫描型 PDF**：每页本质是图片，需要 OCR。

主要困难：

- 双栏阅读顺序错误。
- 页眉页脚反复出现。
- 表格被拆成无意义的文本。
- 图片中的信息无法通过普通文本提取获得。

### 2. Word、PPT 和 Markdown

应按标题、段落、列表、表格、文本框和幻灯片进行解析，不要把所有文本直接拼成一段。

### 3. HTML 和网页

处理流程：

网页下载

→ 去除导航栏、广告和页脚

→ 提取正文

→ 保留标题、链接、发布时间和 URL

→ 去除网页模板产生的重复内容

### 4. Excel、CSV 和关系数据库

小表可以转为带表头的文本；大表不适合把每一行都盲目向量化。

更好的方式：

- 表结构和字段解释进入知识库。
- 数值查询通过 SQL 或数据分析工具执行。
- 文字描述列可以单独建立向量索引。

### 5. 图片、音频和视频

图片可以使用 OCR、图像描述模型和视觉 Embedding。音频和视频通常先使用 ASR 转写，并保留时间戳和说话人信息。

## 1.3 常用解析工具

### 1. PyMuPDF4LLM ✅

主要用于把 PDF 转换成适合 LLM 和 RAG 的 Markdown 或结构化文本。

优点：

- PDF 处理速度较快。
- 能保留部分标题、表格和页面结构。
- 适合论文、技术文档和说明书。

缺点：复杂扫描件和特殊版面仍可能需要 OCR 或版面模型。

### 2. Unstructured ✅

通用文档解析框架，可以把 PDF、Word、PPT、HTML 等转换为 Title、NarrativeText、Table 等统一元素。

优点：多格式统一接入，适合企业知识库。

缺点：复杂文档的解析质量需要单独测试，依赖组件较多。

### 3. Docling

强调 PDF、Office、表格和版面结构转换，适合需要较强文档结构保留的场景。

### 4. OCR 工具

常见思路是：页面图片 → 文字检测 → 文字识别 → 版面排序 → 输出文本。

OCR 结果应保留置信度。低置信度页面可以进入人工检查队列，而不是直接写入知识库。

---

# 2. 数据清洗与文本切块（Chunking）

# 2.1 数据清洗

常见清洗步骤：

- 统一 Unicode、编码、换行和空格。
- 去掉重复页眉、页脚、导航和版权声明。
- 修复 PDF 断行、单词连字符和乱码。
- 内容哈希去除完全重复文档。
- 使用 MinHash、SimHash 或向量相似度识别近重复文档。
- 恢复表格表头、单位和行列关系。
- 识别并脱敏手机号、身份证、密钥等敏感信息。
- 保留原始文档 ID、版本、页码和处理日志。

<aside>
⚠️

清洗不能只追求“文字更短”。错误删除标题、限定条件、单位或否定词，会让检索结果失去原意。

</aside>

## 2.2 为什么需要切块？

不能总是把整篇文档直接向量化，原因包括：

- 文档可能超过 Embedding 模型的输入长度。
- 一个长向量会混合多个主题，检索不精确。
- 检索到整篇文档会给 LLM 带来大量无关内容。
- LLM 上下文长度和 Token 成本有限。

切块需要平衡：

- Chunk 太小：检索精确，但信息不完整。
- Chunk 太大：上下文完整，但相似度被无关内容稀释。

## 2.3 常见切块方法

### 1. 结构感知切块 + 递归切块 ✅

先根据标题、章节、段落、列表、表格和代码函数边界切分。如果某一块仍然太长，再使用递归分隔符继续切。

过程：

文档

→ 按标题分章节

→ 按段落或元素分块

→ 超长块依次尝试双换行、换行、句号、空格、字符

→ 生成最终 Chunk

优点：保留语义和结构，同时实现简单稳定。

### 2. Parent-Child Chunking ✅

建立两种粒度：

- 小 Chunk 用于检索，定位更准确。
- 大的父 Chunk 用于送给 LLM，信息更完整。

过程：

父段落 P

→ 切成子块 c1、c2、c3

→ 只为子块建立向量

→ 命中 c2 后，根据 parent_id 取回 P

它解决“检索希望小、生成希望大”的冲突。

### 3. Sentence Window

以句子为索引单位，但命中后取回前后若干句。

例如命中第 10 句，最后返回第 8～12 句。适合事实分散在相邻句子的资料。

### 4. Semantic Chunking

先计算相邻句子的 Embedding。当相邻语义变化明显时切开。

一种简单定义：

```
change_i = 1 - cosine(e_i, e_(i+1))
当 change_i > threshold 时，在句子 i 和 i+1 之间切分
```

优点：主题完整。

缺点：需要额外计算 Embedding，阈值很敏感。

### 5. 固定 Token 切块

例如每 500 Token 切一块。速度最快，但可能把一句话、一个表格或一个知识点切断。

适合结构弱、格式统一的日志或简单文本，可作为基础基线。

## 2.4 Chunk Overlap（切块重叠）✅

Chunk Overlap 也叫 Sliding Window。它不是独立算法，而是让相邻块重复保留一段内容。

例如：

- Chunk Size = 500 Token
- Chunk Overlap = 100 Token

结果：

- Chunk 1：1～500
- Chunk 2：401～900
- Chunk 3：801～1300

步长公式：

```
stride = chunk_size - chunk_overlap
```

作用：

- 避免关键句刚好落在切块边界。
- 保留前后文联系。
- 提高边界知识被召回的概率。

缺点：

- 增加 Embedding、存储和索引成本。
- 多个召回结果可能高度重复。

常见起点：Overlap 设置为 Chunk Size 的 10%～20%，但应通过评估调整。

## 2.5 Chunk 应保存的 Metadata

每个 Chunk 建议保存：

- chunk_id
- document_id
- parent_id
- chunk_index
- title_path
- page_number
- source_url
- created_at 和 version
- tenant_id、department、ACL
- parser_version 和 embedding_model_version

---

# 3. Embedding 模型

## 3.1 Embedding 的基本原理

Embedding 模型把输入文本映射到 d 维向量空间：

```
E(text) -> e ∈ R^d
```

训练目标是让语义相关的 Query 和 Document 向量更接近，让无关内容更远。

例如正样本是问题 q 和正确文档 d+，负样本是无关文档 d-。一种对比学习目标可以写成：

```
L = -log[ exp(sim(q,d+)/τ) / Σ_j exp(sim(q,d_j)/τ) ]
```

其中：

- sim 是相似度函数。
- τ 是温度参数。
- 分母包含正样本和多个负样本。

模型通过大量这样的样本学习“什么内容应该相似”。

## 3.2 Bi-Encoder 编码结构

普通 Dense Embedding 通常使用 Bi-Encoder：Query 和 Document 分开编码。

```
q_vec = E_q(query)
d_vec = E_d(document)
score(q,d) = similarity(q_vec, d_vec)
```

文档向量可以提前计算和保存，因此能搜索大规模文档库。

优点：速度快、可以预计算。

缺点：Query 和 Document 在编码阶段没有逐 Token 交互，因此精细判断能力低于 Cross Encoder。

## 3.3 相似度公式

### 1. Cosine Similarity ✅

```
cos(q,d) = (q · d) / (||q|| × ||d||)
```

它主要比较方向，值越大越相似。文本向量检索中最常见。

### 2. Dot Product

```
dot(q,d) = q · d = Σ_i q_i d_i
```

当向量都做了 L2 归一化后，点积和余弦相似度等价。

### 3. Euclidean Distance

```
distance(q,d) = sqrt[ Σ_i (q_i - d_i)^2 ]
```

距离越小越相似。

必须按照模型训练说明选择距离函数，不能随意更换。

## 3.4 Embedding 的主要类型

### 1. Dense Embedding ✅

每个文本输出一个稠密向量。它擅长理解同义词、改写和整体语义。

弱点：对型号、数字、缩写、专有名词的精确匹配可能不稳定。

### 2. Sparse Embedding

输出高维稀疏向量，只有少量词或概念的权重非零。它保留更多关键词信号。

### 3. Dense + Sparse ✅

同时保存 Dense 和 Sparse 表示，分别检索后融合。适合既需要语义理解，又需要精确术语匹配的企业 RAG。

### 4. Multi-Vector / Late Interaction

不把整段文本压缩成一个向量，而是保留多个 Token 向量。以 ColBERT 的 MaxSim 为例：

```
Score(q,d) = Σ_i max_j (q_i · d_j)
```

意思是：对 Query 中的每个 Token，寻找 Document 中最匹配的 Token，再把这些最大匹配分数相加。

优点：比单向量表达更精细。

缺点：存储和计算成本更高。

### 5. Multimodal Embedding

将文本、图片等映射到可比较的空间，用于图文搜索、商品搜索和 PDF 图表检索。

## 3.5 常见模型与选择

1. ✅ **BGE-M3 / BGE 系列**：中文和多语言能力好；BGE-M3 可以支持 Dense、Sparse 和 Multi-vector 思路，适合中文企业 RAG。
2. ✅ **Qwen Embedding 系列**：中文、多语言和不同模型规模选择较丰富。
3. **OpenAI Embedding**：API 使用简单、无需部署，适合快速开发，但需要考虑费用和隐私。
4. **Multilingual-E5 / E5**：开源、多语言检索能力强；部分模型要求给 Query 和 Passage 添加指定前缀。
5. **Jina Embeddings**：覆盖长文本、多语言和多模态方向。
6. **Cohere Embed**：托管 API，区分 Query 和 Document 输入类型。
7. **Sentence-BERT**：经典句向量生态，适合学习、轻量项目和自定义微调。

选择时要比较：

- 业务语言和专业领域。
- Recall@K、MRR、nDCG 等检索指标。
- 输入长度和向量维度。
- 单条和批量编码速度。
- GPU 显存、API 费用和隐私要求。
- 是否要求 Query/Passage 前缀。
- 是否支持 Dense、Sparse 或多向量。

<aside>
⚠️

文档和查询通常必须使用同一个 Embedding 模型及相同预处理方式。更换模型、向量维度或归一化规则后，一般要重新生成全部文档向量并重建索引。

</aside>

---

# 4. 向量数据库与 ANN 索引

## 4.1 向量数据库保存什么？

通常同时保存：

- 文本 Chunk
- Dense 或 Sparse 向量
- document_id、chunk_id、parent_id
- 标题、页码、来源和时间
- 租户、用户、部门和权限字段
- 模型版本、数据版本和删除标记

查询过程：

Query

→ Embedding

→ Metadata 预过滤

→ ANN 向量搜索

→ Top-N 候选

→ Rerank

→ Top-K

## 4.2 为什么需要 ANN？

精确搜索需要把 Query 向量与所有 N 个文档向量逐一计算，复杂度大约是：

```
O(N × d)
```

当 N 很大时速度太慢。ANN（Approximate Nearest Neighbor）牺牲少量准确率，快速找到“近似最近”的向量。

## 4.3 常见索引

### 1. HNSW ✅

HNSW 把向量组织成多层邻接图：

- 上层节点少，用于快速跳到大致区域。
- 下层节点多，用于局部精细搜索。

查询从顶层入口开始，每层寻找更接近 Query 的邻居，再向下移动。

重要参数：

- M：每个节点保存的邻居数。越大，Recall 和内存通常越高。
- efConstruction：构建索引时搜索范围。
- efSearch：查询时候选范围。越大，Recall 更高但延迟更大。

优点：在线查询快、召回高。

缺点：内存占用较高，删除和大量更新需要注意维护。

### 2. Flat / Brute Force

与所有向量精确比较，没有近似误差。

优点：结果最准确，适合小数据和建立评估基线。

缺点：大数据时很慢。

### 3. IVF_FLAT

先使用聚类把向量分到多个中心区域。查询时只搜索最接近的若干区域。

重要参数：

- nlist：聚类中心数量。
- nprobe：查询时搜索多少个区域。

nprobe 越大，Recall 越高，速度越慢。

### 4. IVF_PQ

在 IVF 基础上，使用 Product Quantization 压缩向量。

优点：节省内存，适合海量向量。

缺点：量化会损失精度。

### 5. DiskANN 类索引

让大量向量保存在磁盘，只把关键结构放在内存中，适合内存放不下的超大规模数据。

## 4.4 Metadata Filter

例如：

```
tenant_id = 当前公司
AND department = finance
AND year >= 2025
AND access_level <= 当前用户权限
```

权限过滤应在检索阶段执行。不能先取回其他用户的私密文档，再在回答阶段删除。

## 4.5 常见数据库选择

1. ✅ **PostgreSQL + pgvector**：关系数据、事务、权限和向量在同一个系统，适合已有 PostgreSQL 的中小型企业 RAG。
2. ✅ **Qdrant**：开源、向量搜索和 Metadata Filter 能力强，API 清晰，适合中型到大型自建服务。
3. ✅ **Milvus**：专业分布式向量数据库，索引选择多，适合海量向量和高吞吐。
4. **Elasticsearch / OpenSearch**：全文检索、BM25、过滤和向量检索可放在同一搜索系统，适合混合检索。
5. **ChromaDB**：简单、Python 友好，适合学习、Demo 和小型知识库。
6. **Pinecone**：托管服务，减少运维，但需要考虑持续费用和厂商锁定。
7. **Weaviate**：提供向量、过滤和多模态扩展，功能丰富但概念较多。
8. **FAISS**：高性能向量搜索库，但不是完整数据库；权限、服务、备份、持久化和分布式需要自己实现。

选择数据库时要检查：

- 数据规模和向量维度。
- QPS、P95/P99 延迟。
- Metadata Filter 能力。
- 是否支持混合检索。
- 实时写入、更新和删除。
- 备份、恢复、高可用和多租户。
- 索引构建时间、内存和磁盘成本。

---

# 5. 检索系统（Retrieval）

## 5.1 检索的目标

检索阶段通常不是直接找最后 5 个结果，而是先从大量资料中快速找出 Top-N 候选，例如 Top-50 或 Top-100，再交给更慢的 Reranker。

检索需要平衡：

- Recall：不要漏掉正确文档。
- Precision：不要取回太多无关文档。
- Latency：用户不能等待太久。
- Cost：查询扩展和多路检索会增加计算。

## 5.2 倒排索引和关键词检索

倒排索引记录“每个词出现在哪些文档中”。

例如：

```
Python -> 文档1、文档3、文档8
RAG -> 文档2、文档3
```

查询包含 Python 时，不需要扫描所有文档，只读取 Python 的 Posting List。

### TF-IDF 基础

TF 表示一个词在文档中出现的频率；IDF 表示这个词在整个集合中有多稀有。

```
TF-IDF(t,d) = TF(t,d) × IDF(t)
IDF(t) = log[N / df(t)]
```

常见词出现在很多文档中，IDF 较低；稀有词更有区分度。

## 5.3 BM25 关键词检索 ✅

BM25 在 TF-IDF 基础上增加：

- 词频饱和：同一个词出现 20 次，不应该比出现 10 次重要两倍。
- 文档长度归一化：长文档更容易包含关键词，需要进行惩罚。

公式：

```
BM25(q,d) = Σ_(t∈q) IDF(t) ×
            [ f(t,d)(k1+1) /
              ( f(t,d) + k1(1-b+b×|d|/avgdl) ) ]
```

其中：

- f(t,d)：词 t 在文档 d 中出现次数。
- 
    
    
    | d | ：文档长度。 |
    | --- | --- |
- avgdl：平均文档长度。
- k1：控制词频饱和程度，常见起点约 1.2～2.0。
- b：控制长度归一化，常见起点约 0.75。

IDF 的一种形式：

```
IDF(t) = log[ (N - df(t) + 0.5) / (df(t) + 0.5) + 1 ]
```

BM25 流程：

Query 分词

→ 在倒排索引中找到包含查询词的文档

→ 计算每个词的 BM25 分数

→ 所有查询词分数求和

→ 按分数返回 Top-N

优点：

- 型号、姓名、数字、缩写和专业术语匹配强。
- 速度快、结果较容易解释。

缺点：

- 不理解同义词和改写。
- Query 与文档用词不同时可能漏召回。

## 5.4 Dense Vector Retrieval ✅

原理：分别用 Bi-Encoder 把 Query 和文档转换成向量，再搜索相似向量。

过程：

```
q_vec = E(query)
对每个候选文档计算 score(q,d)
通过 HNSW / IVF 等 ANN 索引返回 Top-N
```

常用分数：

```
score(q,d) = cosine(q_vec,d_vec)
```

优点：

- 能识别同义词、改写和隐含语义。
- 用户表述与文档用词不同也可能命中。

缺点：

- 对精确关键词、罕见实体和数字可能不稳定。
- 一个向量压缩整段文本，可能丢失细粒度信息。

## 5.5 Sparse Neural Retrieval

通过神经网络给词项生成稀疏权重，例如模型可能自动把“汽车”扩展到“车辆”等相关词。

它仍可以使用倒排索引，但比传统 BM25 具有更强的语义扩展能力。

优点：关键词信号强、可使用倒排索引。

缺点：索引可能更大，训练和部署更复杂。

## 5.6 混合检索（Hybrid Search）✅

混合检索并行执行：

- BM25 / Sparse：负责精确关键词。
- Dense Vector：负责整体语义。

过程：

Query

→ BM25 Top-N

→ Dense Top-N

→ 分数融合或排名融合

→ 去重

→ 候选池

→ Rerank

### 方法 1：加权分数融合

先把不同检索器的分数归一化：

```
normalized_score = (score - min) / (max - min)
```

再加权：

```
final_score(d) = α × dense_score(d) + (1-α) × bm25_score(d)
```

问题：不同通道的分数分布可能不稳定，归一化方式会影响结果。

### 方法 2：RRF 排名融合 ✅

RRF 不直接比较原始分数，只看每个结果在各检索器中的排名：

```
RRF(d) = Σ_r 1 / (k0 + rank_r(d))
```

其中 rank_r(d) 是文档 d 在检索器 r 中的排名，k0 是平滑常数。

优点：不需要把 BM25 和向量分数校准到同一范围，通常比较稳健。

## 5.7 Metadata Filtering ✅

Metadata Filter 根据确定条件缩小搜索范围，例如时间、文档类型、部门和权限。

过程：

用户问题

→ 从用户身份和问题中提取过滤条件

→ 在满足条件的子集合中执行 BM25 或向量检索

注意：

- Filter 太严格会漏掉正确文档。
- Filter 太宽会增加无关候选。
- 日期和权限等确定条件适合硬过滤；不确定主题适合软排序。

## 5.8 Query Rewrite 与 Query Expansion

### 1. 查询清洗

修正拼写、统一简称、去掉无意义语气词、恢复上下文中的代词。

例如：

“那它多少钱”

→ 根据对话历史改写为

“iPhone 16 Pro 当前价格是多少”

### 2. 查询分解

复杂问题可以拆成多个子问题。

例如：

“比较 A 和 B 的价格、性能和续航”

→ A 的价格、性能、续航

→ B 的价格、性能、续航

→ 汇总比较

适合多跳问题，但会增加检索和生成次数。

## 5.9 高级检索策略（对应 HelloAgents 8.3.5）

### 5.9.1 多查询扩展（MQE）✅

MQE（Multi-Query Expansion）的核心是：**同一个问题可以有多种表达，不同表达可能命中不同文档。**

例如原始 Query：

“如何学习 Python”

LLM 扩展为：

- Python 入门教程
- Python 学习方法
- Python 编程指南

完整过程：

1. 把原始 Query 交给 LLM。
2. LLM 生成 n 个语义等价或互补的查询。
3. 原始查询和扩展查询分别执行检索。
4. 合并所有候选文档。
5. 根据 document_id 或 chunk_id 去重。
6. 使用最大分数、加权分数、RRF 或 Reranker 排序。

扩展集合：

```
Q_expanded = { q, q1, q2, ..., qn }
```

候选集合：

```
C = union[ Retrieve(q_i, per_k) ]
```

简单最大分数融合：

```
Score(d) = max_i score(q_i,d)
```

也可以使用 RRF：

```
Score(d) = Σ_i 1 / (k0 + rank_i(d))
```

优势：

- 处理用户和文档用词不同的问题。
- 对模糊查询、简称和专业术语有效。
- 提升 Recall。

风险：

- LLM 可能生成偏离原意的扩展查询，引入噪声。
- 检索次数增加，延迟和成本上升。
- 扩展过多会让候选池失控。

控制方法：限制扩展数量、保留原始 Query、设置主题一致性检查，并在最后 Rerank。

### 5.9.2 假设文档嵌入（HyDE）✅

HyDE（Hypothetical Document Embeddings）的核心是“用可能的答案去找真实答案”。

传统 Dense Retrieval 使用问题向量搜索文档：

```
E(question) -> search documents
```

但问题通常是疑问句，知识库通常是陈述句，它们在向量空间中可能存在表达差异。

HyDE 过程：

1. LLM 根据 Query 生成一段假设性答案 h。
2. 不把 h 当成最终事实，只把它当作检索查询文档。
3. 对 h 计算 Embedding。
4. 使用 h 的向量搜索真实文档。
5. 真实文档经过 Rerank 后交给最终 LLM。

公式：

```
h = LLM_generate(query)
h_vec = E(h)
Score(d) = cosine(h_vec, E(d))
```

如果生成多个假设文档，可以取平均：

```
hyde_vec = (1/m) × Σ_j E(h_j)
```

为什么有效：

- 假设答案和真实文档都是陈述式文本，语体更接近。
- 假设答案可能包含正确的领域术语和相关概念。
- 即使具体事实有错误，语义方向仍可能帮助找到正确文档。

风险：

- LLM 生成严重错误或错误领域内容时，会把检索引向错误方向。
- 增加一次生成调用，延迟和费用较高。
- 简单精确关键词查询不一定需要 HyDE。

适合：专业领域、概念解释、问题与文档表达差异较大的 Query。

### 5.9.3 统一扩展检索框架 ✅

统一框架把原始 Query、MQE 和 HyDE 放进同一个“扩展—检索—合并”流程。

第一步：构建扩展列表

```
expansions = [original_query]
如果 enable_mqe：加入 q1...qn
如果 enable_hyde：加入 hypothetical_document
去除空值和重复扩展
```

第二步：扩大候选池

```
pool_size = max(top_k × candidate_pool_multiplier, 20)
per_query = max(1, pool_size / number_of_expansions)
```

例如 top_k=8，candidate_pool_multiplier=4，则总候选池至少约 32。

第三步：分别检索

对每个扩展文本 e：

```
e_vec = Embedding(e)
hits_e = VectorSearch(e_vec, limit=per_query, filter=where)
```

第四步：Metadata 和权限过滤

例如只检索：

```
memory_type = rag_chunk
is_rag_data = true
data_source = rag_pipeline
rag_namespace = 当前知识库
```

第五步：去重与合并

使用 memory_id、document_id 或 chunk_id 作为唯一键。同一文档被多条扩展命中时，可以：

- 保留最高分。
- 累加不同查询的命中信号。
- 使用 RRF。

第六步：排序并返回 Top-K

```
merged = deduplicate(all_hits)
merged = sort_by_score(merged)
return merged[:top_k]
```

推荐：

- 一般自然语言查询：原始检索 + MQE。
- 专业领域、语义鸿沟明显：MQE + HyDE。
- 延迟敏感：只使用基础混合检索，或只生成少量 MQE。

## 5.10 Parent-Child Retrieval 与邻居扩展

小 Chunk 命中后，取回父段落、同标题内容或相邻 Chunk。

过程：

检索子块 c_i

→ 根据 parent_id 取回父块 P

→ 根据 chunk_index 取回 c_(i-1)、c_i、c_(i+1)

→ 去重并限制总 Token

它不是提高初始召回，而是提高最后上下文的完整性。

## 5.11 Multi-Vector / ColBERT Retrieval

Query 和 Document 都保留 Token 级向量，使用 MaxSim 进行局部匹配：

```
Score(q,d) = Σ_i max_j(q_i · d_j)
```

它比单一 Dense 向量更能识别“Query 中每个词分别对应文档中的哪部分”，但索引更大、检索更复杂。

## 5.12 Graph Retrieval / GraphRAG

适合关系型、多跳问题，例如：

“A 公司投资了哪些公司，这些公司的创始人毕业于哪里？”

过程：

实体识别

→ 实体链接

→ 在图中搜索邻居、路径或子图

→ 取回节点和边的来源文本

→ LLM 汇总

图检索没有一个统一的固定公式。常见路径分数可以综合边权、节点相关性和路径长度，例如：

```
Score(path) = Σ edge_weight + Σ node_relevance - λ × path_length
```

优点：关系清晰、多跳推理强。

缺点：建图和实体消歧成本高，图谱不完整会漏信息。

## 5.13 SQL / Text-to-SQL Retrieval

适合精确数值和结构化条件问题，例如：

“2026 年销售额最高的三个地区是什么？”

过程：

用户问题

→ 检索数据库 Schema 和字段解释

→ LLM 生成只读 SQL

→ SQL 安全校验

→ 数据库执行

→ 返回结果和来源

这里不是根据文本相似度直接得到答案，因此没有统一相似度公式。关键是 Schema Linking、SQL 正确性、权限和只读限制。

## 5.14 多模态检索

文本 Query 可以检索图片，图片也可以检索文本。基本过程是使用多模态 Embedding 得到可比较的向量，再通过向量相似度搜索。

---

# 6. 重排（Rerank）

## 6.1 为什么检索后还要重排？

第一阶段检索追求速度和 Recall，会返回较多候选。Rerank 使用更强但更慢的模型，对候选进行精细判断。

典型两阶段结构：

```
大规模知识库
→ BM25 / Dense / Hybrid 取 Top-50 或 Top-100
→ Reranker 重新计算相关性
→ 返回 Top-5 或 Top-10
```

## 6.2 Bi-Encoder 和 Cross Encoder 的区别

### Bi-Encoder

Query 和 Document 分开编码：

```
q_vec = E_q(q)
d_vec = E_d(d)
score = cosine(q_vec,d_vec)
```

文档向量可以预计算，所以适合搜索全库。

### Cross Encoder ✅

把 Query 和一个候选 Document 拼成一个序列，一起输入 Transformer：

```
[CLS] Query [SEP] Document [SEP]
```

内部过程：

1. Query Token 和 Document Token 进入同一个 Transformer。
2. Self-Attention 允许每个 Query Token 直接关注每个 Document Token。
3. 模型获得细粒度交互信息，例如否定词、数字、实体关系和条件是否对应。
4. 取 [CLS] 隐状态或分类头输出一个相关性分数。

简化公式：

```
h_cls = Transformer([CLS], q, [SEP], d, [SEP])
score(q,d) = W · h_cls + b
```

如果需要 0～1 概率：

```
p(relevant|q,d) = sigmoid(score(q,d))
```

为什么更准确：Query 与 Document 在编码过程中进行了完整 Token 交互，而不是先压缩成两个独立向量。

为什么不能直接搜索全库：每一个 Query-Document 对都要重新运行一次 Transformer。假设有 100 万个文档，就要进行 100 万次成对推理，成本无法接受。因此 Cross Encoder 只处理第一阶段取回的小候选集。

## 6.3 Cross Encoder 的训练方式

### 1. Pointwise

每个 Query-Document 对有标签 y，相关为 1，不相关为 0。

```
L = -[ y log(p) + (1-y) log(1-p) ]
```

### 2. Pairwise

希望正确文档分数高于错误文档：

```
L = -log sigmoid( score(q,d+) - score(q,d-) )
```

### 3. Listwise

同时考虑一组候选的整体排序，训练目标更接近最终排名，但实现更复杂。

## 6.4 Cross Encoder 重排过程

1. Retriever 返回候选 d1...dN。
2. 构造 N 个输入对 `(q, d_i)`。
3. Reranker 分批推理，得到 s1...sN。
4. 按分数降序排序。
5. 应用 score_threshold 和 Top-K。
6. 把最终 Chunk 交给上下文组装。

优点：准确率高，能识别细节和否定关系。

缺点：延迟高，候选越多越慢；长文档可能超过模型输入长度。

## 6.5 LLM Rerank

让 LLM 阅读 Query 和候选文档，输出相关性评分或排序。

优点：能够理解复杂要求和多条件问题。

缺点：成本高、速度慢、输出稳定性较弱。

适合少量高价值候选，不适合作为大规模第一阶段检索。

## 6.6 MMR 多样性重排

普通排序可能返回很多内容相似的 Chunk。MMR（Maximal Marginal Relevance）同时考虑相关性和多样性。

```
MMR(d) = λ × sim(q,d)
         - (1-λ) × max_(d'∈Selected) sim(d,d')
```

第一项奖励与 Query 相关，第二项惩罚与已选结果重复。

λ 越大越重视相关性，越小越重视多样性。

## 6.7 Rerank 的推荐顺序

1. ✅ Cross Encoder：通用效果最好，生产常用。
2. ColBERT / Late Interaction：速度和精度之间的折中。
3. LLM Rerank：复杂高价值任务。
4. 规则排序：时间、新旧版本、权威来源等业务规则作为补充。

---

# 7. 上下文组装与生成

## 7.1 候选去重

去除：

- 相同 chunk_id。
- Overlap 产生的高重复 Chunk。
- 同一内容的不同文件版本。

可以使用文本哈希、Jaccard、MinHash 或向量相似度识别重复。

## 7.2 邻居扩展和父块回填

命中小块后，可以补充：

- 前后 Chunk。
- 父章节。
- 表格标题和表头。
- 图片图注。

但要控制 Token，防止上下文越来越大。

## 7.3 Context Compression

目标是从召回 Chunk 中保留与问题有关的句子。

过程：

候选 Chunk

→ 句子切分

→ 根据 Query 进行相关性判断

→ 删除无关句子

→ 保留来源信息

风险：压缩模型可能删除限定条件，因此高风险场景应保留原文引用。

## 7.4 上下文排序

常见顺序：

1. 最相关和最权威证据放前面。
2. 相关证据可以按文档顺序排列。
3. 避免最重要证据全部放在很长上下文中间。
4. 冲突资料要同时保留，并标出时间和来源。

## 7.5 Prompt 结构

```
System：角色、回答边界和安全规则
Instructions：只能基于 Context；证据不足时说明不知道
Context：带编号的检索结果和来源
Question：用户问题
Output Format：答案格式、引用格式
```

生成要求：

- 每个关键事实给出来源。
- 不使用证据中不存在的数字和结论。
- 资料冲突时明确说明冲突。
- 没有足够证据时拒绝猜测。

---

# 8. RAG 评估

# 8.1 评估集

至少需要：

- 用户问题。
- 正确文档或正确 Chunk。
- 参考答案。
- 问题类型和难度。
- 权限、时间和来源要求。

评估集应包含：简单事实、同义改写、精确关键词、多跳、时间敏感、无答案和对抗问题。

## 8.2 检索指标

### 1. Hit@K

Top-K 中只要出现一个正确文档就记为 1。

```
Hit@K = 1 if TopK contains relevant document else 0
```

### 2. Recall@K ✅

```
Recall@K = TopK 中正确文档数量 / 全部正确文档数量
```

适合检查是否漏召回。

### 3. Precision@K

```
Precision@K = TopK 中正确文档数量 / K
```

适合检查候选中无关内容是否过多。

### 4. MRR

只关注第一个正确结果的位置：

```
RR = 1 / rank_of_first_relevant
MRR = 所有 Query 的 RR 平均值
```

正确文档排第 1 得 1，排第 5 得 0.2。

### 5. MAP

对每个正确文档出现的位置计算 Precision，再取平均，适合一个 Query 有多个正确结果。

### 6. nDCG@K

同时考虑相关性等级和排名位置：

```
DCG@K = Σ_i [ (2^rel_i - 1) / log2(i+1) ]
nDCG@K = DCG@K / IDCG@K
```

高相关文档排得越靠前，分数越高。

## 8.3 Rerank 评估

比较 Rerank 前后：

- MRR 是否提升。
- nDCG 是否提升。
- 正确文档是否进入最终 Top-K。
- 增加了多少延迟和成本。

不要只看准确率，还要计算每提升一点指标需要多少额外延迟。

## 8.4 生成评估

### 1. Faithfulness（忠实度）✅

答案中的结论是否能被 Context 支持。

### 2. Answer Relevance

答案是否真正回答用户问题，而不是只复述文档。

### 3. Answer Correctness

与人工参考答案相比是否正确。

### 4. Context Precision

提供给 LLM 的 Context 中，有多少内容真正相关。

### 5. Context Recall

回答问题所需要的证据是否都被检索到。

### 6. Citation Precision / Recall

引用是否真的支持对应结论，以及重要结论是否都有引用。

## 8.5 系统指标

- P50、P95、P99 延迟。
- Embedding、检索、Rerank 和 LLM 各阶段耗时。
- 每次请求 Token 和费用。
- 缓存命中率。
- QPS、错误率和超时率。
- 索引大小、内存和磁盘。
- 用户满意度和人工抽检通过率。

常用工具包括 RAGAS、LangSmith、Arize Phoenix 和自建评估脚本，但工具不能替代真实业务测试集。

---

# 9. 推荐的第一版 RAG 方案

1. ✅ 使用格式专用解析器，保存页码、标题和来源。
2. ✅ 结构感知切块 + Recursive Splitter，设置 10%～20% Overlap。
3. ✅ 小块检索 + Parent-Child 回填。
4. ✅ 选择支持业务语言的 Embedding，并建立私有评估集。
5. ✅ PostgreSQL + pgvector、Qdrant 或 Elasticsearch，根据现有系统选择。
6. ✅ BM25 + Dense Hybrid Retrieval，优先使用 RRF 融合。
7. ✅ 初始取 Top-50 左右候选，再使用 Cross Encoder 重排到 Top-5～10。
8. 对用词差异明显的问题启用 MQE；专业概念问题可增加 HyDE。
9. 对最终 Context 去重、扩展邻居、压缩并保留引用。
10. 分开评估 Retrieval、Rerank、Generation、延迟和成本。

<aside>
🧪

最重要的方法不是一次加入所有高级模块，而是先建立简单可评测的基线。每增加 MQE、HyDE、Rerank、GraphRAG 等模块，都要用同一评估集证明它确实提高了效果，而不是只增加复杂度。

</aside>
