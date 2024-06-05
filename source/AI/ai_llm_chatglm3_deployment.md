---
title: ChatGLM 3 本地安装部署以及快速入门
categories: AI
comments: true
tags: [AI]
description: ChatGLM 3 本地安装部署以及快速入门
date: 2024-6-5 10:00:00
---

## 概述

ChatGLM3 是智谱AI和清华大学 KEG 实验室联合发布的对话预训练模型。ChatGLM3-6B 是 ChatGLM3 系列中的开源模型，在保留了前两代模型对话流畅、部署门槛低等众多优秀特性的基础上，ChatGLM3-6B 引入了如下特性：    
 - 更强大的基础模型： ChatGLM3-6B 的基础模型 ChatGLM3-6B-Base 采用了更多样的训练数据、更充分的训练步数和更合理的训练策略。在语义、数学、推理、代码、知识等不同角度的数据集上测评显示，* ChatGLM3-6B-Base 具有在 10B 以下的基础模型中最强的性能*。
 - 更完整的功能支持： ChatGLM3-6B 采用了全新设计的 Prompt 格式 ，除正常的多轮对话外。同时原生支持工具调用（Function Call）、代码执行（Code Interpreter）和 Agent 任务等复杂场景。
 - 更全面的开源序列： 除了对话模型 ChatGLM3-6B 外，还开源了基础模型 ChatGLM3-6B-Base 、长文本对话模型 ChatGLM3-6B-32K 和进一步强化了对于长文本理解能力的 ChatGLM3-6B-128K。以上所有权重对学术研究完全开放 ，在填写 问卷 进行登记后亦允许免费商业使用。

[GitHub 地址](https://github.com/THUDM/ChatGLM3)

模型下载：
HF：https://huggingface.co/THUDM
魔搭：https://modelscope.cn/models/ZhipuAI

## 环境准备

1.查看 GPU 信息：nvidia-smi

2.安装 Anaconda

3.安装GPU版本的PyTorch

先在conda环境中启动 jupyter，通过`import torch`的方式来看有没有安装 PyTorch，如果没有报错，则表示安装了。如果没有安装则安装 PyTorch。     
去 https://pytorch.org/get-started/locally/ 找到合适的安装命令进行安装。Pytorch Build版本选择 Stable(2.2.1)，CUDA版本根据刚才硬件查看的结果`nvidia-smi`来选择对应版本。    
接下来确认当前PyTorch的版本：    

```
import torch
print(torch.__version__)
```

如果输出结果是 `2.2.1+cu118` 则表示当前PyTorch版本符合要求，其中cu代表cuda，是GPU的版本。如果返回的结果是`1.x.x+cpu`，则说明当前PyTorch版本过低，且只支持cpu运行模式。那么就要重新安装。
如果PyTorch版本符合要求，那么再来测试一下是否能调用cuda完成gpu运算。

```
print(torch.cuda.is_available())
```

如果返回True，那么表示和cuda兼容，如果fase则表示当前cuda不兼容，需要重新安装或者升级cuda版本。

如果重新安装PyTorch那么可以先删除现有版本：

```
pip uninstall torch torchvision torchaudio
```

4.cuda 安装：

去[CUDA官网](https://developer.nvidia.com/cuda-downloads) 选择平台安装。

5.下载 ChatGLM3

[GitHub地址](https://github.com/THUDM/ChatGLM3)。

进入目录，conda环境中安装依赖：

```
pip3 install -r requirements.txt
```

```
ERROR: pip's dependency resolver does not currently take into account all the packages that are installed. This behaviour is the source of the following dependency conflicts.
anaconda-cloud-auth 0.1.4 requires pydantic<2.0, but you have pydantic 2.6.4 which is incompatible.
```


## 下载模型到本地

第一次加载模型需要下载模型，会耗费一点事件，因此我们可以把大模型文件预先下载到本地，然后执行

```
export MODEL_PATH=/path/to/model
```


可以通过 `python web_demo_streamlit.py ` 以及 `python cli_demo.py` 来体验网页端和命令行和 ChatGLM3进行对话。


## 代码调用方法



示例代码参考：https://huggingface.co/THUDM/chatglm3-6b

```
from transformers import AutoTokenizer, AutoModel

path = "/home/*********/ChatGLM3/chatglm3-6b"
tokenizer = AutoTokenizer.from_pretrained(path, trust_remote_code=True)
model = AutoModel.from_pretrained(path, trust_remote_code=True).quantize(8).cuda()
model = model.eval()
```

`AutoTokenizer.from_pretrained` 加载模型文件可以是repo id，也可以是本地路径。
如果是 "THUDM/chatglm3-6b" 就会从huggingface下载并缓存，缓存路径是 ~/.cache/huggingface/modules/transformers_modules/THUDM/chatglm3-6b/。
如果是"/path/to/model/"这种本地路径，就可以避免访问http://huggingface.co，从而迅速加载模型。

```
response, history = model.chat(tokenizer, "你好", history=[])
print(response)
```

```
你好👋！我是人工智能助手 ChatGLM3-6B，很高兴见到你，欢迎问我任何问题。
```

```
response, history = model.chat(tokenizer, "什么是机器学习呢？", history=history)
print(response)
```

```
机器学习是一种人工智能的分支，它使用算法和统计模型来让计算机从数据中学习，从而改进其性能。在机器学习中，计算机可以自动调整其内部参数，以便在给定任务上取得更好的成绩。这些算法可以用于许多不同的任务，例如语音识别、图像识别、自然语言处理、推荐系统等。机器学习的目标是让计算机能够像人类一样学习，并最终实现自主学习。
```

```
response, history = model.chat(tokenizer, "我该怎么去学习它呢？", history=history)
print(response)
```

```
学习机器学习，你可以按照以下步骤进行：

1. 学习基础数学知识：机器学习需要一定的数学基础，如线性代数、概率论、微积分等。
2. 学习编程语言：Python 是机器学习最流行的编程语言，你需要掌握 Python 的基本语法和常用库，如 NumPy、Pandas 和 Matplotlib 等。
3. 学习机器学习基础知识：学习机器学习的基本概念和算法，如监督学习、无监督学习、强化学习等。
4. 实践项目：通过实践项目来巩固和应用所学知识，例如使用 Keras 或 TensorFlow 等深度学习框架进行图像分类或自然语言处理等任务。
5. 参加课程或培训：参加在线或线下的课程或培训，以获得更深入的知识和技能。

总之，学习机器学习需要一定的数学基础和编程技能，并通过实践来巩固和应用所学知识。
```

可以看到，加上history参数它是有上下文理解能力的。
如果不加history参数它的回答是这样的：

```
response, history1 = model.chat(tokenizer, "我该怎么去学习它呢？", history=[])
print(response)
```

```
您好！请问您想学习哪方面的内容？您可以提供更具体的信息，以便我为您提供更有针对性的建议。
```


## OpenAI 风格 API调用方法

ChatGLM3 还提供了 OpenAI 风格的 API调用方法，可以让 ChatGLM3 模型无缝接入 OpenAI 开发生态。     
ChatGPT 虽然是闭源的，但是它的SDK是开源可用的，我们通过在本地模拟一个server来接收 API调用，实际还是访问的是本地的 ChatGLM3 模型。    

首先需要安装openai库：    

```
pip install openai
```

由于依赖 https://huggingface.co/BAAI/bge-large-zh-v1.5/ ，为了减少下载时间，可以clone到本地。然后设置路径。

```
export EMBEDDING_PATH=/home/***/BAAI/bge-large-zh-v1.5
```

然后进入openai_api_demo运行 api_server.py 启动一个服务。

```
python api_server.py
```

可以首先运行一下他们提供的测试脚本：

```
python openai_api_request.py
```

```
当然可以！让我给您讲一个关于森林里的小动物们的故事。

在一片广阔的森林里，住着各种各样的小动物。有一天，小动物们都在森林里玩耍，突然它们听到了一阵悦耳的声音。原来是一只美丽的鸟儿在唱歌！

歌声吸引了所有的小动物们前来欣赏。它们都聚在一起，静静地听着那美妙的歌曲。当歌曲结束时，小动物们为那位鸟儿的美丽歌声而欢呼雀跃。

从那天起，小动物们与那只鸟儿成为了好朋友。它们经常一起玩耍、分享快乐时光。有时候，这只鸟儿还会教其他小动物们如何唱歌，让森林里充满了欢声笑语。

这就是一个关于森林里小动物们的故事。希望您喜欢这个温馨、快乐的故事！如果您还有其他问题或想听其他故事，请随时告诉我！

从前，在一个遥远的国度里，有一个美丽的村庄。这个村子四面环山，风景如画，绿树成荫，小溪潺潺。村民们勤劳善良，生活和谐美满。

村子里有一位聪明、勇敢的少年，名叫小明。他乐于助人，对待每个人都充满友善。有一天，小明听说村子外的一座山上有一个神奇的宝箱，里面装满了各种珍宝。他心想：“如果能得到那个宝箱，我们的生活会变得更加美好。”

于是，在一天清晨，小明带着他的竹筏，踏上了寻找神奇宝箱的征程。他穿过了青山绿水，克服了重重困难，终于来到了那座神秘的山上。

在山脚下，小明发现了一个巨大的石碑，上面刻着“勇者之路”。他知道这是通往山顶的必经之路，于是，他鼓起勇气，开始攀爬。在攀爬过程中，他遇到了许多挑战和危险，但他从未放弃。每当遇到困难，他想起了村子里的亲朋好友，他们的鼓励和支持给了他力量。

经过艰苦努力，小明终于到达了山顶。在那里，他找到了那个神秘的宝箱。宝箱里面确实装满了各种珍宝，应有尽有。然而，小明并没有

嵌入完成，维度： 1024
```

还可以体验在 jupyter 里面代码编写。

```
from openai import OpenAI

base_url = "http://127.0.0.1:8000/v1/"
client = OpenAI(api_key="EMPTY", base_url=base_url)

chat_completion = client.chat.completions.create(
    messages=[
        {
            "role": "user",
            "content": "你好",
        }
    ],
    model="chatglm3-6b",
)


print(chat_completion)
```

输出：

```
ChatCompletion(id='', choices=[Choice(finish_reason='stop', index=0, logprobs=None, message=ChatCompletionMessage(content='你好👋！我是人工智能助手 ChatGLM3-6B，很高兴见到你，欢迎问我任何问题。', role='assistant', function_call=None, tool_calls=None, name=None))], created=1712140058, model='chatglm3-6b', object='chat.completion', system_fingerprint=None, usage=CompletionUsage(completion_tokens=30, prompt_tokens=8, total_tokens=38))
```

## 定制

如果运行时遇到 `torch.cuda.OutOfMemoryError: CUDA out of memory.` 的问题，可以限制一下模型的加载精度：

```
#api_server.py

#model = AutoModel.from_pretrained(MODEL_PATH, trust_remote_code=True, device_map="auto").eval()
#改成
model = AutoModel.from_pretrained(MODEL_PATH, trust_remote_code=True).quantize(8).cuda()
model = model.eval()
```

## 相关文章
[20分钟本地部署ChatGLM3-6B](https://blog.csdn.net/xiangxiang613/article/details/134965097?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-134965097-blog-135192585.235%5Ev43%5Epc_blog_bottom_relevance_base5&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-134965097-blog-135192585.235%5Ev43%5Epc_blog_bottom_relevance_base5&utm_relevant_index=6)

