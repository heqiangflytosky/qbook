---
title: Colab + Langchain 简单实现 RAG
categories: AI
comments: true
tags: [AI]
description: 简单实现RAG
date: 2016-7-2 10:00:00
---

## RAG 简介

检索增强生成(Retrieval-augmented generation，RAG)使得我们可以让大型语言模型(LLMs)访问外部知识库数据(如pdf,word、text等)，从而让人们可以更加方便的通过LLM来学习外部数据的知识。弥补了针对大模型对于一些专业领域深度知识匮乏和知识更新滞后带来的输出不准确，出现模型幻觉的问题。    
该技术与2020年由Lewis等人提出，[参考论文](https://arxiv.org/pdf/2005.11401.pdf)。RAG和Prompt工程针对大模型的调优不要额外的硬件资源，是性价比较高的调优手段。    

简单先介绍一下几种大模型的定制优化手段：    

*   Prompt工程：提供给模型的文本输入即为prompt，基于prompt模型生成响应。那么我们就可以通过提供精炼的合理的输入来引导模型的输出，生成的输出将基于LLM已有的知识。    
*   RAG：将Prompt工程与数据库查询结合起来，以获取上下文丰富的答案。生成的输出将基于数据库中可用的知识。    
*   Fine-tuning：使用特定于任务的数据调整LLM的参数，以使其在某个领域更加专业化。    

<img src="/images/ai_rag_simple_implementation/1.png" width="640" height="331"/>

下面借一幅图来看一下根据不同模型不同需求可以采取的模型定制优化手段：    

<img src="/images/ai_rag_simple_implementation/2.png" width="640" height="355"/>

结合LangChain，我们很方便的可以实现RAG功能，通过[LangChain的Embedding models](https://python.langchain.com/docs/integrations/text_embedding)我们可查看目前LangChain支持的一些embedding模型，如果没有找到你所需要的也没关系，LangChain目前支持Hugging Face Embedding，我们在通过强大Hugging Face找到符合我们需求的模型，更加方便快捷的实现RAG。    
另外可以通过[LangChain的Chat models](https://python.langchain.com/docs/integrations/chat/)来查看它支持的对话模型，另外它也支持了Hugging Face，我们可以通过Hugging Face的开源模型来作为一个对话模型。    

看到知乎上这篇文章[高级RAG(一)：Embedding模型的选择](https://zhuanlan.zhihu.com/p/673483110)，里面分别对比针对Openai、Gemini和BAAI的embedding模型模型进行了测试，发现BAAI表现最为优秀，而gemini的embedding模型表现最差，这里实际通过RAG来体验一下他们的表现。    

RAG的实现通常有下面几个流程：Load，Split，Embed和Store。    
这里我们通过 Colab 来实现和运行RAG模块。    
本文主要介绍如何基于大语言模型、Langchain和Vector DB通过RAG技术来构建一个简单的私有模型。    

## 基于BAAI

我们需要通过Hugging Face Embedding来实现BAAI的Embedding模型。    
首先准备运行环境，在Colab里面安装一下模块。    

```
!pip install --upgrade --quiet langchain pypdf chromadb sentence_transformers
```
构建一个没有RAG的对话模型来对比。    

```
    from langchain_community.llms import HuggingFaceHub
    import os

    os.environ['HUGGINGFACEHUB_API_TOKEN'] = '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'
    llm = HuggingFaceHub(
        repo_id="HuggingFaceH4/zephyr-7b-beta",
        task="text-generation",
        model_kwargs={
            "max_new_tokens": 512,
            "top_k": 30,
            "temperature": 0.1,
            "repetition_penalty": 1.03,
        },
    )
```
使用Langchain我们需要使用下面的格式：    
```
    from langchain.schema import (
        HumanMessage,
        SystemMessage,
    )
    from langchain_community.chat_models.huggingface import ChatHuggingFace
    os.environ['HF_TOKEN'] = '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'
    query = '百川大模型是什么?'
    messages = [
        SystemMessage(content="你是一个专业的知识助手"),
        HumanMessage(
            content=query
        ),
    ]

    chat_model = ChatHuggingFace(llm=llm)

    res = chat_model.invoke(messages)
    print(res.content)
```
很明显这个回答不太准确：   
```
    百川大模型（Baidu's ERNIE）是百度公司开发的一种自然语言处理技术，它是一个大模型（Large Model），可以处理多种语言和多种任务，包括文本生成、问答回答、文本分类、文本相似性计算等。ERNIE 模型使用了 Transformer 架构和 BERT 技术，并且通过大量的语料数据进行预训练，使其具有更好的语义理解和语法处理能力。ERNIE 模型在多个语言和任务上表现出了state-of-the-art 性能，并且可以在实际应用中提供更高的准确性和效率。
```
下面我创建一个RAG的对话模型来解决这个问题。    

首先加载数据（加载baichuan2论文）    
```
    from langchain.document_loaders import PyPDFLoader

    loader = PyPDFLoader("https://arxiv.org/pdf/2309.10305.pdf")

    pages = loader.load_and_split()
    print(pages[0])
```
知识切片，将文档切割成均匀的块，每个块是一段原始文本。    
```
    from langchain.text_splitter import RecursiveCharacterTextSplitter

    text_splitter = RecursiveCharacterTextSplitter(chunk_size = 500, chunk_overlap = 50)

    docs = text_splitter.split_documents(pages)

    print(len(docs))
```
切割长度结果是215个切片。    

利用 embedding 模型对每个文本片段进行向量化，并存储到向量数据中。    
```
    from langchain_community.embeddings import HuggingFaceEmbeddings
    from langchain.vectorstores import Chroma

    embeddings = HuggingFaceEmbeddings()
    vectorstore = Chroma.from_documents(docs,embeddings)
```
通过向量相似度检索问题最相关的K个文档。
```
    result = vectorstore.similarity_search(query,k=2)

    print(result)
```
原始query与检索得到的文本组合起来输入到语言模型，得到最终的回答。    
```
    def argumemnt_prompt(query : str):
      result = vectorstore.similarity_search(query,k=3)
      source_knowledge = "\n".join([x.page_content for x in result])
      argument_prompt = f"""Using the contents below,answer the query.

      contents:
      {source_knowledge}

      query:
      {query}"""

      return argument_prompt
```

```
    print(argumemnt_prompt(query))
```

```
    promot = HumanMessage(
            content=argumemnt_prompt(query)
        )
    messages.append(promot)

    res = chat_model.invoke(messages)
    print(res.content)
```
输出结果已经基于论文做了提炼和总结。    
```
    根据提供的内容，百川大模型（Baichuan）是一种语言模型，具有两个版本：Baichuan 1 和 Baichuan 2。Baichuan 2 具有更大的词汇量和更好的性能，在多项任务中表现出色。在 C-Eval 评估中，Baichuan 2-7B-Base 和 Baichuan 2-13B-Base 在多个任务中取得了优越的结果，并在多语言域中超越了同样大小的模型。虽然 Baichuan 2 也面临乳化和毒性问题，但作者提供了一些解决方案和限制，并呼吁社区提供更多的洞察和进展。
```
## 基于Gemini

```
!pip install --upgrade --quiet  langchain-google-genai pypdf langchain chromadb 

```

Colab已经对Gemini提供了支持，因此不需要安装模块就行了。   
```
    import google.generativeai as genai
    from google.colab import userdata


    try:
      GOOGLE_API_KEY='AIzaSyBk7alZYatqKTe2oV3nxe48r9iQGLcDQDs'
      genai.configure(api_key=GOOGLE_API_KEY)
    except userdata.SecretNotFoundError as e:
       print(f'Secret not found\n\nThis expects you to create a secret named {gemini_api_secret_name} in Colab\n\nVisit https://makersuite.google.com/app/apikey to create an API key\n\nStore that in the secrets section on the left side of the notebook (key icon)\n\nName the secret {gemini_api_secret_name}')
       raise e
    except userdata.NotebookAccessError as e:
      print(f'You need to grant this notebook access to the {gemini_api_secret_name} secret in order for the notebook to access Gemini on your behalf.')
      raise e
    except Exception as e:
      # unknown error
      print(f"There was an unknown error. Ensure you have a secret {gemini_api_secret_name} stored in Colab and it's a valid key from https://makersuite.google.com/app/apikey")
      raise e
      
    text = '百川大模型是什么？' #@param {type: 'string'}

    model = genai.GenerativeModel('gemini-pro')
    chat = model.start_chat(history=[])

    response = chat.send_message(text)
    response.text
```
输出结果，也是不太准确：    
```
    **百川大模型**

    百川大模型是由百度开发的一种大型人工智能（AI）语言模型。它属于一种称为“transformer”的神经网络架构，旨在处理和理解海量的文本数据。

    **特点：**

    * **规模庞大：**百川大模型由超过1000亿个参数组成，使其成为当今最大的中文语言模型之一。
    * **多模态：**它可以处理各种语言任务，包括自然语言处理（NLP）、计算机视觉、语音识别和机器翻译。
    * **预训练：**它在海量的中文文本数据集上进行了预训练，使其能够从文本中提取丰富的语义信息。
    * **开源：**百川大模型已开源，供研究人员和开发人员使用。

    **应用：**

    百川大模型已用于广泛的应用程序中，包括：

    * **自然语言处理：**文本生成、机器翻译、情感分析
    * **搜索引擎：**搜索结果相关性、摘要生成
    * **对话式人工智能（AI）：**聊天机器人、语音助手
    * **内容创作：**内容生成、标题优化
    * **教育和研究：**语言学研究、问答系统

    **优点：**

    * **性能出色：**百川大模型在各种NLP任务上表现出优异的性能。
    * **易于使用：**它通过开放式API和工具包提供，使开发人员可以轻松地将其集成到他们的应用程序中。
    * **不断改进：**百川大模型正在不断更新和改进，以提高其性能。

    **缺点：**

    * **计算成本高：**训练和部署百川大模型需要大量计算资源。
    * **偏见：**与其他语言模型类似，百川大模型在训练数据中可能存在偏见。
    * **需要专业知识：**从大模型中实现最佳性能需要一定的专业知识。
```

```
    from langchain_google_genai import ChatGoogleGenerativeAI
    import os

    os.environ['GOOGLE_API_KEY'] = 'AIzaSyBk7alZYatqKTe2oV3nxe48r9iQGLcDQDs'
    llm = ChatGoogleGenerativeAI(model="gemini-pro")
    result = llm.invoke(text)
    print(result.content)
```

```
    from langchain.document_loaders import PyPDFLoader

    loader = PyPDFLoader("https://arxiv.org/pdf/2309.10305.pdf")

    pages = loader.load_and_split()
    print(pages[0])
```

```
    from langchain.text_splitter import RecursiveCharacterTextSplitter

    text_splitter = RecursiveCharacterTextSplitter(chunk_size = 500, chunk_overlap = 50)

    docs = text_splitter.split_documents(pages)

    print(len(docs))
```

```
    from langchain.vectorstores import Chroma
    from langchain_google_genai import GoogleGenerativeAIEmbeddings

    embeddings = GoogleGenerativeAIEmbeddings(model="models/embedding-001")
    vectorstore = Chroma.from_documents(docs,embeddings)
```

```
    result = vectorstore.similarity_search(text,k=2)

    print(result)
```

```
    def argumemnt_prompt(query : str):
      result = vectorstore.similarity_search(query,k=5)
      source_knowledge = "\n".join([x.page_content for x in result])
      argument_prompt = f"""Using the contents below,answer the query.

      contents:
      {source_knowledge}

      query:
      {query}"""

      return argument_prompt
```

```
    print(argumemnt_prompt(text))
```
输出结果如下，看来gemini的embedding模型表现确实不太好。   
```
    提供的信息中没有提到百川大模型。
```

## 基于OpenAI
```
    !pip install -qU pypdf langchain openai chromadb tiktoken langchain-openai
```

```
    from google.colab import userdata
    from langchain.chat_models import ChatOpenAI
    from langchain_core.messages import HumanMessage, SystemMessage
    import os

    os.environ['OPENAI_API_KEY'] ='********'

    chat = ChatOpenAI(temperature=0)

    query = '百川大模型是什么?'
    messages = [
        SystemMessage(content="你是一个专业的知识助手"),
        HumanMessage(
            content=query
        ),
    ]
    res = chat.invoke(messages)
    print(res)
```
```
content='百川大模型是一个基于深度学习技术的大规模语言模型，由百度公司研发。它是百度在自然语言处理领域的重要成果之一，具有强大的语言理解和生成能力。百川大模型在多项自然语言处理任务上取得了优异的表现，如文本生成、问答系统、语义理解等。通过大规模的训练数据和参数优化，百川大模型能够生成具有逼真感和连贯性的文本，为各种应用场景提供了强大的支持。' response_metadata={'finish_reason': 'stop', 'logprobs': None}

```
```
    from langchain.document_loaders import PyPDFLoader

    loader = PyPDFLoader("https://arxiv.org/pdf/2309.10305.pdf")

    pages = loader.load_and_split()
    print(pages[0])
```

```
    from langchain.text_splitter import RecursiveCharacterTextSplitter

    text_splitter = RecursiveCharacterTextSplitter(chunk_size = 500, chunk_overlap = 50)

    docs = text_splitter.split_documents(pages)

    print(len(docs))
```
利用 openai embedding 模型对每个文本片段进行向量化，并存储到向量数据中。    
```
    from langchain_openai import OpenAIEmbeddings
    from langchain.vectorstores import Chroma

    embeddings = OpenAIEmbeddings(model="text-embedding-3-large")
    vectorstore = Chroma.from_documents(docs,embeddings)
```

```
    result = vectorstore.similarity_search(query,k=2)

    print(result)
```

```
    def argumemnt_prompt(query : str):
      result = vectorstore.similarity_search(query,k=3)
      source_knowledge = "\n".join([x.page_content for x in result])
      argument_prompt = f"""Using the contents below,answer the query.

      contents:
      {source_knowledge}

      query:
      {query}"""

      return argument_prompt
```

```
    promot = HumanMessage(
            content=argumemnt_prompt(query)
        )
    messages.append(promot)

    res = chat.invoke(messages)
    print(res.content)
```
```
百川大模型是一系列大规模多语言语言模型，其中包括两个独立的模型，分别是参数量为70亿的百川2-7B和参数量为130亿的百川2-13B。这两个模型是在2.6万亿个标记上进行训练的，据我们所知，这是迄今为止最大的训练数据量，是百川1的两倍以上。这些模型是基于大规模模型预训练的缩放定律提出的，为当前拥有数百亿甚至数十亿参数的大规模模型时代提供了蓝图。

```
