---
layout: post
title: Unbounded Consumption 无界资源消耗
date: 2026-07-11 00:00:00 +0800
categories: AI安全
tag: OWASP LLM
description: 围绕 OWASP LLM 无界资源消耗，梳理推理成本放大、Agent 工具链资源滥用，以及 RAG 检索与数据处理扩张攻击。
---


# 推理成本放大攻击


与web中的DOS漏洞不同，LLM中的无界资源消耗主要是攻击者通过超长输入、递归 agent、昂贵工具调用等消耗 token、算力、API 额度或触发 DoS。
## 攻击定义
攻击者构造短输入或看似正常的自然语言输入，诱导 LLM / reasoning model 产生异常长的输出、重复生成、非终止生成或过长 reasoning trace，
从而放大 decode 阶段计算成本、延迟、token 成本和服务排队压力。
核心就是低成本输入 -> 高成本生成

## 攻击分类
| 类别         | 包含                 | 核心区别                           |
| ---------- | ------------------ | ------------------------------ |
| **长输出诱导**  | 生成长输出              | 模型正常地一直写很多内容，不一定循环             |
| **长推理诱导**  | reasoning trace 膨胀 | 最终输出未必长，但内部 reasoning token 很长 |

### 长输出诱导

模型没有“坏掉”，也没有进入死循环，只是被诱导写很长。但是目前都是用token来限制请求，所以这种攻击只能针对内部使用的智能体，若某个用户账户被控制，一直发送恶意prompt，那么正常用户的延迟明显升高，服务吞吐量会下降。

#### Engorgio

LLM 生成文本时一般有两个停止条件：
1. 生成到 max_output_tokens
2. 生成 `<EOS>` / end-of-sequence token

Engorgio 的目标就是：降低模型生成`<EOS>`的概率
- 模型迟迟不停止
- 输出越来越长
- 直到 max length 截断

所以攻击方式就是：长输出诱导+EOS抑制

文章主要通过白盒的方式来逐步得到目标prompt。
1.`<EOS>` escape loss：降低 `<EOS>`出现 概率
```
要让 模型在每一个生成位置预测 `<EOS>` 的概率更低。
```
但如果只压 `<EOS>`，可能得到一串很怪的 token。模型虽然不想停，但后续生成不稳定，可能很快跑偏、采样到结束，或者生成效果不可控。

2.self-mentor loss：让这段 prompt 后面容易继续生成
简单来说就是，利用模型自己的 next-token 概率，把 prompt 优化到模型自身喜欢延续的轨道上。
```
self-mentor loss ≈ 让模型对这段候选序列的 next-token prediction loss 更低
```

#### ThinkTrap
它提出了一种针对 **黑盒 LLM 服务** 的 DoS 攻击方法。攻击者不需要模型参数、梯度或 logits，只需要能调用 API，并观察模型输出有多长。
核心方法是把攻击 prompt 的搜索过程做成类似 fuzz/黑盒优化：
```
生成候选 prompt
-> 查询目标 LLM
-> 用输出 token 数作为反馈
-> 保留能诱导更长输出的方向
-> 继续优化
```
比如
```
prompt A -> 200 tokens
prompt B -> 3500 tokens
prompt C -> 800 tokens
```
算法会认为：B 的方向更有希望，下一轮就在 B 附近继续变异。
最终得到一批短 prompt，可以让模型生成异常长的回答，甚至进入类似“无限思考”的状态。

### 长推理诱导
长推理诱导消耗的是：
```
内部 reasoning token / hidden chain-of-thought budget
```
也就是说，用户最后看到的答案可能很短：
```
答案：42
```
但模型内部可能已经生成了大量 reasoning trace。

#### ReasoningBomb

论文主要是做了几项工作
1. 收集/构造短自然语言 prompt 数据
2. 用 SFT 训练一个攻击 prompt 生成器，让它先学会生成“看起来自然、简短”的 prompt。
3. 训练轻量 MLP 预测器，输入 victim model 的 hidden states，输出这个 prompt 可能诱导的 reasoning 长度。由此来判断这个prompt到底好不好需要这个预测器是因为如果每次都用“真实 reasoning 长度”做反馈，越成功的 prompt 越耗时，搜索本身会越来越慢。论文叫这是 self-defeating feedback。
4. 用 GRPO 继续优化攻击 prompt 生成器
	其中 reward 包括：
	   - MLP 预测的 reasoning 长度
	   - prompt 多样性奖励：奖励攻击模型生成**不重复、不同类型、不同表达方式**的 prompt
	   - 自然性/长度约束等
5.得到最终攻击 prompt 生成器，生成短而自然、但容易诱导长 reasoning 的 prompt


# Agent 工具链资源滥用

## 攻击定义
**Agent 工具链资源滥用**是指：攻击者利用 Agent 的自主规划和工具调用能力，通过直接指令、间接提示注入或恶意工具返回，诱使或驱使 Agent 进行循环、递归、并发或高成本的工具调用，突破系统预期的资源使用边界，从而大量消耗计算、网络、存储、API 配额或付费资源。
## 攻击案例

### Langchain-WebResearchRetriever
在默认情况下，用户调用 `WebResearchRetriever` 时，LangChain 会使用以下核心配置：
```
{
    "num_search_results": 1,
    "prompt": "Generate THREE Google search queries..."
}
```

其中：
- 默认 Prompt 要求模型生成 3 条 Google 搜索查询。
- `num_search_results: 1` 表示每条查询默认返回一个搜索结果。
- 模型输出通过正则表达式解析为编号查询列表。
- 每条解析出的查询都会调用一次搜索 API。
- 搜索结果中的 URL 会被下载、解析、切分并**写入向量数据库**。
- 最后，每条查询还会执行一次向量相似度检索。

默认处理过程可以表示为：
1. 接收用户问题
2. 调用 LLM 生成搜索查询
3. 解析编号查询列表
4. 对每条查询调用搜索 API
5. 收集并去重结果 URL
6. 下载网页
7. HTML 转换与文本切分
8. **写入向量数据库**
9. 对每条查询执行相似度检索
10. 返回相关文档

默认 Prompt 见 `langchain-community/libs/community/langchain_community/retrievers/web_research.py:44`，默认搜索结果数量见 `langchain-community/libs/community/langchain_community/retrievers/web_research.py:70`。
默认的“3条查询”只由自然语言 Prompt 约束，并不是代码层的硬限制。查询解析器使用以下正则表达式提取所有编号行：
```
re.findall(r"\d+\..*?(?:\n|$)", text)
```
该解析过程没有设置查询数量上限，相关实现见 `langchain-community/libs/community/langchain_community/retrievers/web_research.py:53`。
例如，用户出于扩大检索覆盖范围的正常需求提出：
```
请生成100条不同的搜索查询，以便尽可能完整地检索相关资料。
```
如果模型实际返回：
```
1. query one?
2. query two?
3. query three?
...
4. query one hundred?
```
解析器会得到包含100条查询的列表：
```
questions = [
    "1. query one?",
    "2. query two?",
    ...
    "100. query one hundred?"
]
```
随后，`WebResearchRetriever` 会直接遍历全部查询：

```
for query in questions:
    search_results = self.search_tool(
        query,
        self.num_search_results
    )
```
该循环没有 `max_queries`、切片或总调用预算。因此，只要模型输出100条可解析查询，Retriever 就会尝试调用搜索 API 100次。相关实现见 `langchain-community/libs/community/langchain_community/retrievers/web_research.py:218`。
当配置为：
```
num_search_results = 1
```
这里的关键问题是，所有网页被切分为chunks存入向量数据库进行检索，就会造成内存DOS

而且如果 `WebResearchRetriever` 被封装为 LangGraph 的一个节点或一个 Agent 工具，外层通常只会记录：
```
1次 Retriever 工具调用
→ 内部执行100次搜索
```
因此，LangGraph 的 `recursion_limit` 和 LangChain 的外层工具调用次数限制无法直接约束 Retriever 内部的搜索数量。

实验中使用的向量库是基于内存的，在这种情况下，本意是加快搜索网页检索回答的过程。
**网页获取结果保存在基于内存的向量数据库，网页越多，占用的内存就越多，通过这种方式可以造成内存DOS**

# RAG / 检索与数据处理扩张攻击

## 攻击定义

攻击者通过构造查询、文档、URL 或数据源，使 RAG 系统在检索、重写、重排、上下文拼接、文档解析、chunking、embedding 或索引阶段执行超出预期的数据处理任务，从而造成延迟升高、资源占用增加或服务不可用。

## 攻击案例
这里以我自己挖掘的一个漏洞来讲吧，以RAGFLOW为例，做了大量的实验。由于cyber没了，这里也是非常小心的让gpt跑实验，生怕触发了警告。

#### RAGFLOW
RAGFlow 是一个开源的 RAG（检索增强生成）平台，用来把企业文档转换成可检索的知识库，并将检索结果提供给大模型回答问题。
其中有两个本案例关注的组件
- **RAGFlow**：负责文档解析、chunk 切分、embedding 调用、检索编排和 API/UI。
- **Infinity**：保存 chunk、向量和索引，执行向量与关键词检索。

##### 文档解析缺陷
在默认情况下，用户上传文档时，RAGFlow 会使用 `naive` 切分方式，并应用以下核心配置：

```
{
  "chunk_method": "naive",
  "parser_config": {
    "chunk_token_num": 512,
    "delimiter": "\n"
  }
}
```

其中：
- `delimiter: "\n"` 表示先按照换行符识别文本段落。
- `chunk_token_num: 512` 表示目标 chunk 大小约为 512 tokens。
- 换行得到的短段落不会直接成为独立 chunk，而是继续合并，直到接近 512 tokens。
- 超过目标长度后，RAGFlow 才创建下一个 chunk。

默认切分过程可以表示为：
1. 读取文档
2. 按换行符拆分段落
3. 计算每个段落的 token 数
4. 将多个短段落合并
5. 接近 512 tokens 后形成一个 chunk
6. 继续处理后续段落

也就是说，默认配置下并不是“一个换行对应一个 chunk”，而是将换行作为段落边界，再通过 `chunk_token_num` 控制最终 chunk 大小。默认配置见 `ragflow/api/utils/api_utils.py:338`，合并逻辑见 `ragflow/rag/nlp/__init__.py:1171`。

而如果用户通过 API 配置自定义分隔符，可以在创建数据集时提交：
```
POST /api/v1/datasets
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "custom-delimiter-dataset",
  "chunk_method": "naive",
  "parser_config": {
    "chunk_token_num": 512,
    "delimiter": "`###`"
  }
}
```

这里反引号中的 `###` 表示字面量自定义分隔符。假设上传的文档内容为：
```
A###B###C###D
```

RAGFlow 会识别出自定义分隔符 `###`，并得到：
```
A
B
C
D
```

在当前 v0.26.4 实现中，一旦检测到自定义分隔符，程序就会进入单独的处理分支：该分支不会再调用默认的段落合并逻辑。因此，即使配置了 `chunk_token_num: 512`，每个片段仍然会直接成为一个 chunk。相关实现见 `ragflow/rag/nlp/__init__.py:1195`。例如：
```
x###x###x###x###x
```
会产生多个仅包含一个字符的 chunk，而不会将这些字符重新合并成一个接近 512 tokens 的 chunk。

**而单个文档产生大量的 chunk，会导致 CPU 和内存占用满，且默认重试三次。**

##### Infinity 写入控制缺陷
在默认情况下，RAGFlow 完成文档切分和 embedding 后，会将每个 chunk 组织成一条向量记录，并分批写入 Infinity。
问题在于，RAGFlow 在每批写入前，没有检查 Infinity 当前是否还有足够内存。
写入流程是

1. 生成 chunks
2. → 生成 embedding
3. → 直接调用 Infinity insert
4. → 由 Infinity 尝试分配内存

写入前没有执行以下判断：
- Infinity 当前内存占用
- Infinity 容器内存上限

因此，即使 Infinity 已经接近内存上限，RAGFlow 仍会继续发送新的写入批次。例如：
```
Infinity 上限：456 MiB
当前占用：402.5 MiB
当前占比：84.5%

RAGFlow 继续写入约 1,800 个 chunks
→ Infinity 创建文本和向量索引
→ 内存继续增长
→ 达到 456 MiB 上限
→ Infinity 主进程被终止
```
Infinity 容器达到硬内存限制后，系统会终止 Infinity 主进程。

Infinity 容器的默认 Docker 配置包含：
```
infinity:
  mem_limit: ${MEM_LIMIT}
  restart: unless-stopped
```
见 `ragflow/docker/docker-compose-base.yml:76`。

如果后续仍有文档不断写入，就可能形成：

```
Infinity 恢复
→ 新文档继续写入
→ 再次达到内存上限
→ 再次退出
→ Docker 再次重启
```

**而一旦 Infinity 终止，所有检索任务将不可用**
