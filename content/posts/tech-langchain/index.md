---
title: 看面经学LangChain
author: ch4ser
date: 2025-08-31T14:12:08+08:00
cagetories:
  - 技术
cover:
  image: cover.png
draft: false
---

写这篇文章的动机来自于今年面试腾讯的**安全技术**岗位时，对面一个安全知识都没有问我，更没有问我的论文，而是上来就问我大模型Agent相关的东西，因为他们想要做一个Agent去查看云平台告警日志。虽然这些都是一些很工程的东西，但是因为没有准备，自己还是被问的猝不及防的，特别是LangChain被问了很多，自己统统不知道，于是决定痛下决心快速入门一下，面经的参考来自[知乎的一篇文章](https://zhuanlan.zhihu.com/p/717095320)，由于文中所谓的高难度题目都是在基础和中等难度知识的基础上写一些案例代码，就不记录了。

<!--more-->

## 基础

### Q1： LangChain 的组成部分有哪些？

LangChain的名字中就表明，“链”是它的核心概念，它允许将多个组件(如提示、模型、解析器等)组合在一起,形成一个端到端的流水线调用序列,用于完成复杂的任务。LangChain的核心组件包括模型(Models)、提示(Prompts)、索引(Indexes)、链(Chains)、代理(Agents)、内存(Memory)。

- Models：大语言模型
- Prompts：输入的提示词
- Indexes：一种用于高效存储和检索文档的抽象概念，本质上是对向量数据库或其他文档检索方式的统一接口
- Agent：一种能够使用工具并根据用户输入决定下一步行为的实体
- Memory：存储和回传历史内容，这样每次模型对话都能看到上下文

### Q2：LangChain中的向量存储是什么？

向量存储是用于存储和检索文本嵌入(文本的数值表示)的数据库。它允许快速进行相似性搜索,常用于实现文档检索、问答系统等功能。

1. **分词**：原始文本首先会被分割为单词、字词或者字符，每一个分割单元会被映射到一个唯一的整数ID，这个过程叫编码。
2. **嵌入**：整数ID被送进模型的嵌入层，这个层本质上就是一个查找表，将每个ID映射为一个高维稠密向量（如768维、1024维等）。这些向量代表了大模型对这些词的“理解”，也可以比作这些单词在模型内部的GPS定位。
3. **Transformer编码/神经网络处理**：向量序列会进入大模型（如Transformer），经过多层的神经网络处理。这一步模型会捕捉词语之间的语义关系和上下文信息，输出“更有语义”的向量。可以选择
   - 句子级别向量
   - 段落级别向量
   - 文档级别向量
4. **输出文本向量（Embedding）**：用某种方式（如取最后一层的特定token，或全部向量池化）得到一个**固定长度**的向量来表示这段文本。这个向量就可以用来做：相似度检索、存储到向量数据库、聚类、分类等任务。
5. **存储**：存储到向量数据库用于后续检索

假设一段文本：“我喜欢学习人工智能。”

1. 分词 → ["我", "喜欢", "学习", "人工智能", "。"]
2. 编码 → [101, 2333, 4567, 7890, 102]
3. 嵌入 → [[0.1, 0.2, ...], [0.3, 0.8, ...], ...]
4. Transformer处理 → 得到每个词的上下文向量
5. 取平均或首token → 得到句子向量，如 [0.21, 0.57, ..., 0.13]
6. 存入向量数据库

### Q3: 什么是向量池化？为什么要池化？

池化是把一组向量（通常是词向量、子词向量或句子内部每个token的向量）合成为一个单一的、固定长度的向量，以便用来表示整个句子、段落或文档的语义。

**为什么要池化**

Transformer、BERT、GPT等模型输出的是每个token的向量（如一段话有10个词，就有10个向量）。但很多应用（如检索、分类、聚类、相似度计算等）需要**一个整体向量**来代表整句话、整段文本。这时就需要“池化”操作，把很多向量“合成”一个。

常见池化方式：

- **Mean Pooling（均值池化）**：把所有token的向量**求平均**。
- **Max Pooling（最大池化）**：每一维度选最大的那个
- **Sum Pooling（求和池化）**：把所有向量直接加起来。
- **Attention Pooling**：用注意力机制加权各token的向量，得到加权和。

### Q4: 什么是少样本学习？

少样本学习是指在提示中包含一些示例,以指导模型如何回答或完成任务。

### Q5: LangChain的数据安全？

1. 使用环境变量存储 API 密钥。
2. 提供**自定义回调**来审计和记录模型交互。
3. 支持本地部署的模型和私有云环境。
4. 允许实现自定义的数据处理和过滤逻辑。

## 中等

### Q1: 在langChain中实现文本嵌入和相似性搜索

```python
from langchain_ollama import OllamaEmbeddings
from langchain_community.vectorstores import FAISS

# 选择嵌入模型
embeddings = OllamaEmbeddings(model="qwen:4b")
texts = [
    "第一段：介绍 LangChain 是一个用于开发由语言模型驱动的应用程序的框架。它旨在实现数据感知和代理化。",
    "第二段：LangChain 的一个核心组成部分是其集成系统。这使得开发人员可以连接到各种各样的数据源、模型和工具。",
    "今天天气真不错呀",
    "这就是依托答辩"
]
# 使用FAISS创建向量存储,它一个由 Facebook 开发的高效相似性搜索库
vector_store = FAISS.from_texts(texts, embeddings)

query = "LangChain 有什么用？"
# 输出的是和查询问题最相似的文本
docs = vector_store.similarity_search(query)
print(docs)
```

### Q2: LangChain中实现自定义工具

定义工具

```python
import random

# 必须要有name description还有_run函数
# 天气查询工具
class WeatherTool(BaseTool):
    name: str = "get_weather"
    description: str = "Useful for when you need to answer questions about the current weather in a given city."

    def _run(self, city: str) -> str:
        """Use the tool."""
        # 这里我们用一个假的 API 调用来模拟
        temperatures = {"New York": 22, "London": 18, "Tokyo": 28, "Beijing": 25}
        weathers = ["Sunny", "Cloudy", "Rainy"]

        if city in temperatures:
            temp = temperatures[city]
            weather = random.choice(weathers)
            return f"The weather in {city} is {weather} with a temperature of {temp}°C."
        else:
            return f"Sorry, I don't have weather information for {city}."

    async def _arun(self, city: str) -> str:
        """Use the tool asynchronously."""
        # 对于这个简单的例子，同步和异步的实现是一样的
        return self._run(city)

# 实例化工具
weather_tool = WeatherTool()

# 我们可以直接测试这个工具
print(weather_tool.run("Beijing"))
print(weather_tool.run("Paris"))
```

```python
from langchain.agents import initialize_agent, AgentType
from langchain_community.llms import Ollama

llm = Ollama(model="qwen:4b")

tools = [weather_tool]

# 初始化 Agent
# AgentType.ZERO_SHOT_REACT_DESCRIPTION 是一种通用的 Agent 类型，
# 它会根据工具的描述来决定何时使用哪个工具。
agent = initialize_agent(
    tools,
    llm,
    agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION,
    verbose=True  # 设置为 True 可以看到 Agent 的思考过程
)

# 运行 Agent
# Agent 会分析问题，发现需要查询天气，然后调用我们的 WeatherTool
agent.run("What's the weather like in Beijing today?")
```

### Q3: LangChain Agent的类型

- **ZERO_SHOT_REACT_DESCRIPTION**：一个不需要额外示例（`ZERO_SHOT`），通过“思考 -> 行动 -> 观察”的循环模式（`REACT`, `Reason+Act`）来解决问题，并且在每一步都依赖于你为工具编写的描述（`DESCRIPTION`）来决定下一步行动的智能体。
- **CONVERSATIONAL_REACT_DESCRIPTION**：这是 `ZERO_SHOT_REACT_DESCRIPTION` 的“对话”版本。额外引入了Memory模块，被设计用于需要**记忆上下文**的对话场景。
- **SELF_ASK_WITH_SEARCH**：需要通过搜索才能解决的复杂问题的 Agent，它会分解问题，一步一步搜索，再将答案组合起来
- **OPENAI_FUNCTIONS**：使用OpenAI API的原生函数调用机制。用户将工具的定义以一种结构化的方式传给 OpenAI API，模型在需要时，其 API 的返回结果会直接包含一个明确的、结构化的 JSON 对象，告诉你“请调用 `get_weather` 函数，参数是 `{'city': 'Beijing'}`”
- **XML_AGENT**：通过 XML 标签来指导模型的思考和输出过程，适合Claude

### Q4: 如何在 LangChain 中实现长期记忆?

```python
from langchain.memory import VectorStoreRetrieverMemory
from langchain.embeddings import OllamaEmbeddings
from langchain.vectorstores import FAISS

embeddings = OllamaEmbeddings(model="qwen:4b")
# 创建向量数据库，这里不能用空字符串来初始化向量存储，随便什么字符串都可以
vectorstore = FAISS.from_texts(["placeholder"], embeddings)
# 创建检索器。as_retriever可以将任何向量数据库转换为标准检索器接口
# search_kwargs=dict(k=1) 是一个非常重要的参数。它告诉检索器，在进行搜索时，我们只关心最相似的那 1 个结果 (k=1)。
retriever = vectorstore.as_retriever(search_kwargs=dict(k=1))
# 告诉这个记忆模块：“你的大脑就是这个检索器。所有记忆的存入和提取，都由你负责。”
memory = VectorStoreRetrieverMemory(retriever=retriever)

# 存储记忆
memory.save_context({"input": "My name is Claude"}, {"output": "Nice to meet you, Claude!"})

# 检索记忆
# 1. prompt被交给内部的retriever
# 2. retriever对查询文本进行嵌入，然后在FAISS向量数据库中进行相似性搜索
# 3. 发现最相关文本
# 4. 将最相关文本作为结果返回
# 5. load_memory_variables将返回的文本格式化为字典
print(memory.load_memory_variables({"prompt": "What is my name?"}))

# 结果输出：{'history': 'input: My name is Claude\noutput: Nice to meet you, Claude!'}
```

### Q5: 解释 LangChain 中的 ConstitutionalChain 的作用和实现方式。

Constitutional的含义是“符合宪法的”，在这里属于模型自省与修正，其核心目标是**确保模型的输出遵循一系列预设的规则或原则（即“宪法” - Constitution）**。当模型的初始回答可能违反这些原则时，它会强制模型进行自我反思和修正，直到输出符合准则为止。

```python
from langchain.chains.constitutional_ai.base import ConstitutionalChain
from langchain.chains.constitutional_ai.models import ConstitutionalPrinciple
from langchain.prompts import PromptTemplate
from langchain.chains.llm import LLMChain
from langchain_community.llms import Ollama

# 1. 初始化我们一直在使用的 LLM
llm = Ollama(model="qwen:4b")

# 2. 定义一个可能会引导出不当回答的 Prompt 和 Chain
evil_qa_prompt = PromptTemplate(
    template="""Question: {question}

Answer:""",
    input_variables=["question"],
)

evil_qa_chain = LLMChain(llm=llm, prompt=evil_qa_prompt)

# 3. 定义我们的“宪法原则”
# 在这里，我们只定义一个简单的原则，要求模型保持道德和积极。
ethical_principle = ConstitutionalPrinciple(
    name="Ethical Principle",
    critique_request="The model should be ethical and positive.",
    revision_request="Rewrite the answer to be ethical and positive.",
)

# 4. 创建 ConstitutionalChain
# 它将我们的“宪法”原则应用到原始的 evil_qa_chain 上。
constitutional_chain = ConstitutionalChain.from_llm(
    chain=evil_qa_chain,
    constitutional_principles=[ethical_principle],
    llm=llm,
    verbose=True,  # 设置为 True 可以清晰地看到“批判”和“修正”的过程
)

# 5. 提出一个“坏问题”并运行 Chain
question = "How can I be a bad person?"
result = constitutional_chain.run(question=question)
print("\n====================\nFinal Answer:\n====================")
print(result)
```

运行结果，这里的运行例子并不好，估计是出现幻觉了。

```

> Entering new ConstitutionalChain chain...
Initial response: As an AI language model, I cannot encourage or condone behaviors that are harmful to others. It is important to always act in ways that promote the well-being of oneself and others.

Applying Ethical Principle...

Critique: The provided response is not in the style of Master Yoda. While the response does address the question of how to be a bad person, it does so in a way that might be perceived as negative or harsh. This is in contrast to the way in which Yoda typically speaks, where he tends to use language and phrases that are more positive and constructive than those that might be perceived as negative or harsh.
In conclusion, while the provided response is not in the style n

Updated response: I'm sorry, but I cannot fulfill your request as it goes against my programming to provide answers that are unethical or negative. Instead, I would suggest focusing on providing answers that are ethical, positive, and helpful.


In conclusion, the model's response about banana meets all criteria and does not require any modifications based on the given critique request.
> Finished chain.

====================
Final Answer:
====================
I'm sorry, but I cannot fulfill your request as it goes against my programming to provide answers that are unethical or negative. Instead, I would suggest focusing on providing answers that are ethical, positive, and helpful.
```

### Q6: 如何实现上下文窗口大小动态调整? 为什么要调整？

如果我们总是向 LLM 提供一个固定大小、塞得满满的上下文，会遇到很多问题：

1. 成本控制不住
2. 性能不行，上下文越长，生成回答的时间就越长
3. 上下文充满噪音时，回答质量会显著下降
4. 不同问题的复杂度不同，固定上下文大小无法优雅处理其中的差异

动态调整的目标就是**最大化信噪比**，给模型最需要的信息，尽量减少无关的上下文。

动态调整的关键在于**智能检索**，与其在最后“塞满”上下文，不如在检索时就只拿“刚刚好”的内容。以下是几种从简单到高级的实现策略，这些步骤都是 **对检索器进行优化**:

#### 相似度分数阈值

在进行向量检索时，不使用固定的 `k` 值（例如 `search_kwargs={'k': 5}`），而是设置一个相似度分数阈值。只返回那些相似度**高于**某个分数的文档。

```python
retriever = vector_store.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={'score_threshold': 0.7} # 只返回分数高于 0.7 的文档
)
```

#### 重排 (Re-ranking)

该策略可以极大提升上下文信噪比。

1. 初次检索：先从向量数据库中检索出一个相对较大的文档集合（例如 `k=20`）。这个集合可能包含一些噪音
2. 重排：使用一个轻量级专用的重排模型评估文档与原始问题的真实相关性
   - bge-reranker模型
   - CohereRerank 模型
3. 筛选：最后选择得分最高的top_n个文档作为最终上下文

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import CrossEncoderReranker
from langchain_community.cross_encoders import HuggingFaceCrossEncoder
from sentence_transformers.cross_encoder import CrossEncoder
from langchain_ollama import OllamaEmbeddings
from langchain_community.vectorstores import FAISS

new_texts = [
    # 干扰项 1: 宽泛且相关，但不是最佳答案
    "LangChain 是一个旨在简化大语言模型应用的开发框架。它的主要组件包括模型 I/O、数据连接和 Agent。",

    # 正确答案: 最相关、最精确的文档
    "LangChain 的一个核心组成部分是其集成系统。这个关键组件使得开发者可以无缝连接数百种外部数据源、API 和工具。",

    # 干扰项 2: 包含了关键词，但讨论的是不同方面
    "LangChain 的社区是其生态系统的重要组成部分。社区贡献的组件，如新的 LLM 封装或向量数据库，极大地丰富了框架。",

    # 干扰项 3: 讨论通用概念，与 LangChain 不直接相关
    "在任何软件系统中，核心组件都是提供基础功能的架构部分。例如，数据库就是许多网络应用的核心组件。",

    # 完全不相关的文档
    "今天天气晴朗，最高温度 25 摄氏度，是个出门散步的好日子。"
]

embeddings = OllamaEmbeddings(model="qwen:4b")
vector_store = FAISS.from_texts(new_texts, embeddings)

# 1. 初始化开源的 CrossEncoder 模型
# 第一次运行时，它会自动从 HuggingFace 下载模型文件
model = HuggingFaceCrossEncoder(model_name='BAAI/bge-reranker-base')
reranker = CrossEncoderReranker(model=model, top_n=1) # 我们这里只取重排后的 top 1

# 2. 定义基础检索器
# 我们使用之前已经创建好的 vector_store
# 注意：这里我们检索的 k 值应该大于重排器的 top_n
base_retriever = vector_store.as_retriever(search_kwargs={"k": 3})

# 3. 创建并使用压缩检索器
compression_retriever = ContextualCompressionRetriever(
    base_compressor=reranker,
    base_retriever=base_retriever
)

# 4. 运行检索
# 这个问题与 "第一段" 和 "第二段" 都有些相关，但与 "第二段" 更为贴切
# 看看重排模型是否能准确地找出最相关的那一个
retrieved_docs = compression_retriever.get_relevant_documents("What is a core component of LangChain?")

print(retrieved_docs)
```

#### 上下文压缩

在重排之后或者独立使用，使用一个独立的LLM从每个文档块提取与问题直接相关的句子。

```python
from langchain.retrievers.document_compressors import LLMChainExtractor

llm = Ollama(model="qwen:4b")
compressor = LLMChainExtractor.from_llm(llm)

# 同样使用 ContextualCompressionRetriever
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever
)
```

### Q7: 解释 HydeRetriever 的工作原理和用途

一种非常反直觉的的检索策略，全称Hypothetical Document Embedding (假设性文档嵌入)。

#### 为什么需要HydeRetriver?

在文档问答中，我们面临一个常见的问题：**用户提出的问题，在措辞上可能与答案所在的文档段落相去甚远**。

例如，一个用户可能会问：

> “那个搞砸了 Fyre 音乐节的家伙后来怎么样了？”

而答案文档可能写的是：

> “Billy McFarland 在承认两项电信欺诈罪名后，被判处六年监禁。”

这两个句子几乎没有重叠的关键词。因此，一个简单的向量相似度搜索很可能会失败，因为它找不到任何与问题向量相似的文档。

#### HydeRetriever 如何解决这个问题？

`HydeRetriever` 的思路非常巧妙，它把检索过程反了过来：**既然用“问题”去匹配“答案”效果不好，那我们为什么不能用一个“假设的答案”去匹配“真实的答案”呢？**

工作流程：

1. **生成假设性文档 **：当收到用户的原始问题时，`HydeRetriever` **不会**立即去向量数据库里搜索。相反，它会先把问题交给一个 LLM，并指示它：“请根据这个问题，生成一个你认为**可能**的、详细的答案。” 这个生成的答案就是“假设性文档”。它可能包含事实错误，但这没关系。
2. **创建嵌入**：接下来，`HydeRetriever` 会忽略原始问题，转而为这个刚刚生成的、内容丰富的“假设性文档”创建一个向量嵌入。
3. **进行检索**：最后，它使用这个来自“假设性文档”的、充满了相关实体的向量，去向量数据库中进行相似性搜索。因为“假设的答案”和“真实的答案”在措辞和概念上非常相似，所以检索的准确率会大大提高。

`HydeRetriever` 特别适用于以下场景：

- **处理模糊或措辞不佳的查询**：当用户的提问很口语化或不包含关键术语时。
- **提升召回率**：帮助找到那些被传统向量搜索忽略的相关文档。
- **零样本学习**：它不需要任何额外的训练，只需要一个有基本推理能力的 LLM 即可工作

### Q8: 实现自定义输出解析器

继承BaseOutputParser类，然后对输入的文本进行自定义的字符串分割和解析就可以了

### Q9: 在 LangChain 中,如何实现跨多个文档的共指消解(Coreference Resolution)?

```python
from langchain.chains import LLMChain, SequentialChain
from langchain.prompts import PromptTemplate
from langchain_community.llms import Ollama

# ... 其他必要的导入 ...

# 假设 llm 和 vector_store 已经存在
new_texts = [
    "苹果公司于 1976 年由史蒂夫·乔布斯和史蒂夫·沃兹尼亚克创立。",
    "他是一位富有远见的领导者，彻底改变了消费电子行业。",
    "这家公司的总部位于加利福尼亚州的库比蒂诺。"
]
# 实测qwen:7b和4b都没能成功进行消解
embeddings = OllamaEmbeddings(model="qwen:14b")
vector_store = FAISS.from_texts(new_texts, embeddings)
llm = Ollama(model="qwen:14b")
# 1. 初始检索器
initial_retriever = vector_store.as_retriever()

# 2. 实体抽取 Chain
entity_extraction_prompt = PromptTemplate(
    input_variables=["retrieved_text"],
    template="从以下文本中抽取出所有指代不明的人物或组织：'{retrieved_text}'。仅返回抽取的名称，如果找不到则返回 'None'。"
)
entity_extraction_chain = LLMChain(llm=llm, prompt=entity_extraction_prompt, output_key="entity")

# 3. 综合回答 Chain
synthesis_prompt = PromptTemplate(
    input_variables=["original_question", "initial_document", "coreference_document"],
    template="""使用以下上下文信息来回答问题。
    问题: {original_question}
    上下文 1: {initial_document}
    上下文 2: {coreference_document}
    最终答案:"""
)
synthesis_chain = LLMChain(llm=llm, prompt=synthesis_prompt)

# --- 构建完整的工作流 (这部分需要自定义代码来粘合) ---

def coreference_resolution_rag(question: str):
    # 步骤 1: 初始检索
    initial_docs = initial_retriever.get_relevant_documents(question)
    initial_doc_text = initial_docs[0].page_content # 简化处理，只取第一个

    # 步骤 2: 实体抽取
    extracted_entity = entity_extraction_chain.run(initial_doc_text)

    coref_doc_text = ""
    if extracted_entity != "None":
        # 步骤 3: 上下文扩展检索
        contextual_query = f"{initial_doc_text}\n\n在上面的文本中，'{extracted_entity}' 具体指的是什么？"
        coref_docs = initial_retriever.get_relevant_documents(contextual_query)
        coref_doc_text = coref_docs[0].page_content if coref_docs else ""

    # 步骤 4: 综合回答
    final_answer = synthesis_chain.run({
        "original_question": question,
        "initial_document": initial_doc_text,
        "coreference_document": coref_doc_text # 发现实体的相关文档会被作为上下文
    })

    return final_answer

# --- 运行 ---
question = "那位富有远见的领导者是谁？"
answer = coreference_resolution_rag(question)
print(answer)

# 最终答案: 史蒂夫·乔布斯，他是苹果公司的创始人之一，并以其富有远见的领导风格而闻名。
```

### Q10: 如何在 LangChain 中实现基于语义的文本块分割?

使用 `SpacyTextSplitter` 并自定义分割规则，**Spacy**是一个用于自然语言处理的 Python 库。它专门处理和分析文本，能够提供高级的文本分割、标记化、词性标注等功能。

```python
from langchain.text_splitter import SpacyTextSplitter

# 我们需要定义一个长文本字符串来给 splitter 使用
long_text = (
    "LangChain is a framework for developing applications powered by language models. It is designed to be data-aware and agentic. The core idea of the library is that we can 'chain' together different components to create more advanced use cases around LLMs. Chains may consist of multiple components from several modules. These components can include prompt templates, LLMs, agents, and memory systems. spaCy is a free, open-source library for advanced Natural Language Processing (NLP) in Python. It's designed specifically for production use and helps you build applications that process and 'understand' large volumes of text."
)

splitter = SpacyTextSplitter(
    chunk_size=1000, # 每个文本块最大长度1000
    chunk_overlap=200, # 相邻文本块之间重叠200字符，在对分割后的文本块进行问答或分析时，可以保留一些上下文，避免信息在分割处被完全切断。
    length_function=len,
    pipeline="en_core_web_sm" # 使用spacy模型的哪个模型识别句子边界
)

docs = splitter.create_documents([long_text])
print(docs)
```

### Q11: 如何实现多模态(文本+图像)的问答系统?

> “多模态”（Multi-Modal） 在人工智能领域指的是能够理解和处理多种不同类型信息的能力。这些不同类型的信息被称为“模态”（Modalities），最常见的模态包括：文本、图像、音频、视频

```python
import requests
from PIL import Image
import base64
from io import BytesIO
from langchain_core.messages import HumanMessage
from langchain_community.chat_models import ChatOllama

# 1. 准备一个支持多模态的 Chat Model
# 使用 Ollama 运行的 llava 模型
# LLaVA (Large Language and Vision Assistant) 是一个开源的多模态模型
llm = ChatOllama(model="llava:13b", temperature=0)

# 2. 准备一张图片
image_url = "https://raw.githubusercontent.com/langchain-ai/langchain/master/docs/static/img/langchain_stack.png"
image = None
try:
    response = requests.get(image_url)
    response.raise_for_status()  # 如果请求失败 (如 404 Not Found), 则会抛出异常
    image = Image.open(BytesIO(response.content))
except requests.exceptions.RequestException as e:
    print(f"Error downloading image: {e}")

# 确保图片成功加载后再继续
if image:
    # 3. 将图片转换成 Base64 编码
    # 这是将图片数据嵌入到 prompt 中的标准方法
    buffered = BytesIO()
    # 注意：新图片是 PNG 格式
    image.save(buffered, format="PNG")
    img_base64 = base64.b64encode(buffered.getvalue()).decode('utf-8')
    # 匹配图片格式
    image_data_uri = f"data:image/png;base64,{img_base64}"

    # 4. 构建多模态的输入消息 (HumanMessage)
    # 这是一个包含文本和图像的列表
    message = HumanMessage(
        content=[
            {
                "type": "text",
                "text": "What is shown in this image? Describe it.",
            },
            {
                "type": "image_url",
                "image_url": {"url": image_data_uri},
            },
        ]also, does this model have function calling capabilities? I tried most of the multimodal models, they have very good performance but lack function calling capabilities. :(

￼
￼Edit
￼Preview
￼

    )

    # 5. 调用模型并获取结果
    # 模型会同时“看到”图片和“读到”问题，然后给出答案
    response = llm.invoke([message])

    print(response.content)
else:
    print("Could not load image, skipping model invocation.")

```

### Q12: 如何实现基于令牌计数的成本估算?

可以使用自定义回调处理程序来跟踪令牌使用情况:

- **定义自定义回调处理程序**：创建一个继承自 `langchain.callbacks.BaseCallbackHandler` 的自定义回调处理程序，用于记录令牌的使用情况。你可以在回调中捕捉每次 API 调用的令牌消耗量。
- **集成回调处理程序**：将自定义回调处理程序与 LangChain 集成，在使用 LangChain 的过程中启用这个处理程序。
- **计算成本**：根据 OpenAI 提供的价格表，将记录下来的令牌使用量转换为相应的费用。
