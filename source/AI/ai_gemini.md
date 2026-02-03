---
title: Android Gemini介绍
categories: AI
comments: true
tags: [AI]
description: Android Gemini介绍
date: 2016-7-2 10:00:00
---

## Gemini 概述
Gemini 是 Google 发布的 AI 大模型，目前推出了三个不同体量的模型，分别应用于不同的场景：     

 - Gemini Pro：在成本和延迟方面性能优化的模型，可在广泛的任务范围内提供较优性能。该模型表现出强大的推理性能和广泛的多模态能力。Gemini Pro在谷歌的数据中心运行。
 - Gemini Nano：用于在数码设备上运行的最高效模型，目前应用于谷歌的智能手机系统 Google  Pixel  8，旨在执行需要高效AI处理但不需要连接到外部服务器的本地任务，比如在聊天应用中建议回复或总结文本，目前应用于Pixel  8录音机应用中的自动摘要功能，以及Gboard键盘的智能回复部分。谷歌训练了两个Nano版本，参数分别为18亿（Nano-1）和32.5亿（Nano-2）。
 - Gemini Ultra：虽然目前还没有被广泛使用，但谷歌将Gemini Ultra描述为其最强大的模型。可在各种高度复杂的任务（包括推理和多模态任务）中提供最先进的性能。

在 Android 设备上目前我们可以有两种方式来在应用程序中使用 Gemini。通过API来访问云端 Gemini Pro 模型以及通过 AICore 来访问终端设备运行的 Gemini Nano 模型（目前仅支持Pixel 8 Pro）。    

##  Google AI Studio
[Google AI Studio 快速入门](https://ai.google.dev/tutorials/ai-studio_quickstart?hl=zh-cn)
Google AI Studio 是一个基于浏览器的 IDE，用于使用生成模型进行原型设计。通过 Google AI Studio，可以快速试用模型并针对不同的 Prompt 和模型参数进行实验。在构建出满意的结果后，可以将对应代码导出到自己的应用程序中。    
Google AI Studio 可以针对以下模型参数进行设置：    

 - 模型
 - temperature
 - topK
 - topP
 - 最大输出
 - 安全设置

如需了解其中每个参数的信息，请参阅“LLM 简介”指南的[模型参数](https://ai.google.dev/docs/concepts?hl=zh-cn#model_parameters)部分。    

##  Gemini API

Google 提供了 Gemini API 来访问在 Google 服务器上运行的 Genmini Pro 模型。     
[快速入门：开始在 Android 应用中使用 Gemini API（客户端 SDK）](https://ai.google.dev/tutorials/android_quickstart?hl=zh-cn)     
[Github SDK 开源 ：generative-ai-android ](https://github.com/google/generative-ai-android)    
[Gemini API 参考文档](https://ai.google.dev/api?hl=zh-cn)    

使用 Gemini API 首先需要获取 API KEY，然后 下载 Android Studio 最新预览版，在 New Project 向导中访问 Gemini API Starter 模板，可以快速搭建一个 AI 项目。具体可以参考[在 Android Studio 中构建您的首个生成式 AI 应用](https://developer.android.com/studio/preview/gemini-template?hl=zh-cn)。    
API KEY的获取以及 Gemini API的访问需要使用国外代理，具体可以查看 [Google AI Studio 和 Gemini API 支持的语言和地区](https://ai.google.dev/available_regions?hl=zh-cn)。    
代码示例：     
```
    import com.google.ai.client.generativeai.GenerativeModel

    val model = GenerativeModel(
        "gemini-1.0-pro",
        BuildConfig.apiKey,
        generationConfig = generationConfig {
            temperature = 0.9f
            topK = 1
            topP = 1f
            maxOutputTokens = 2048
        },
        safetySettings = listOf(
            SafetySetting(HarmCategory.HARASSMENT, BlockThreshold.MEDIUM_AND_ABOVE),
            SafetySetting(HarmCategory.HATE_SPEECH, BlockThreshold.MEDIUM_AND_ABOVE),
            SafetySetting(HarmCategory.SEXUALLY_EXPLICIT, BlockThreshold.MEDIUM_AND_ABOVE),
            SafetySetting(HarmCategory.DANGEROUS_CONTENT, BlockThreshold.MEDIUM_AND_ABOVE),
        ),
    )

    val chatHistory = listOf(
        content("user") {
            text("中国足球水平如何")
        },
        content("model") {
            text("**2023 年中国足球水平评估**\n\n**总体排名：**\n\n* 国际足联世界排名：第 85 位（截至 2023 年 3 月）\n* 亚洲足联排名：第 12 位\n\n**近期表现：**\n\n* 2022 年东亚杯：亚军\n* 2023 年亚洲杯预选赛：以小组第一出线\n\n**优势：**\n\n* **身体素质：**中国球员普遍身体强壮，耐力较好。\n* **纪律性：**中国足球强调纪律性和团队合作。\n* **进攻潜力：**中国队拥有多名具有爆发力和创造力的进攻球员。\n\n**劣势：**\n\n* **技术能力：**中国球员的技术能力相对较弱，特别是传球和控球方面。\n* **战术素养：**中国队在战术安排和应变方面仍需提高。\n* **国际经验不足：**中国球员缺乏在顶级国际比赛中的经验。\n\n**发展方向：**\n\n* 提高球员的技术能力，尤其是传球和控球。\n* 加强战术训练，提升球队在不同情境下的应变能力。\n* 增加球员的国际比赛经验，以提高他们的信心和竞争力。\n* 完善青训体系，培养更多有天赋的年轻球员。\n\n**结论：**\n\n中国足球处于一个逐步复苏的阶段。球队拥有身体素质和进攻优势，但仍需在技术、战术和国际经验方面提升。通过持续的努力和投资，中国足球有潜力在未来取得更高的成就。")
        },
    )

    val chat = model.startChat(chatHistory)
    val response = chat.sendMessage("YOUR_USER_INPUT")
    //val response = generativeModel.generateContent("YOUR_USER_INPUT")

    // 下面两种方式来获取回复内容
    response.text?.let { Log.d("Gemini", it) }
    response.candidates.first().content.parts.first().asTextOrNull()?.let { Log.d("Gemini", it) }
```

##  Android AI Core
[Android AICore 概览](https://developer.android.com/ml/aicore?hl=zh-cn)      
[在 Android（设备端）上开始使用 Gemini Nano](https://ai.google.dev/tutorials/android_aicore?hl=zh-cn)      
Gemini  Nano 是针对终端设备的最有效模型，可直接在移动芯片上运行，为一系列重要应用场景提供支持。在端侧运行可以保证数据不离开设备。      
Gemini Nano 是从较大的 Gemini 模型中提取精髓进行训练，并针对移动芯片加速器做了专门的优化。      
Android 提供了通过 AICore (com.google.android.aicore)来支持访问在设备上运行的 Gemini Nano ，目前仅支持在Pixel 8 Pro 设备上使用， 仅支持文本到文本模式。可以使用 Android 版 Google AI Edge SDK 来访问 AICore，目前此 SDK 目前处于抢先体验预览版 (EAP) 阶段，并未完全公开，如果要使用需要向 Google 申请。       

AI Core 是一种新的系统级后台服务，可让用户访问设备端执行的基础模型从而为其它应用程序提供 AI 功能。通过在设备上本地运行 Gemini Nano 模型，Google 可以减少对云处理的依赖，支持离线访问，从而节省流量、提高性能和隐私性。      
AICore 负责管理 Gemini Nano 的分发以及更新，应用无需关心大模型的下载和更新，以及运行时对内存的影响。      
AI Core 架构：      

<img src="/images/ai_gemini/1.png" width="616" height="385"/>

AICore 可以通过 LoRA 微调 Gemini Nano，让应用开发者能够基于自己的训练数据创建小型LoRA适配器，然后由AICore 加载针对应用本身的用例进行微调的 LoRA 适配器，形成强大的大语言模型。通过安全模块可调整安全设置，基于[Googles AI 原则](https://ai.google/principles/?hl=zh-cn)为帮助开发者打造安全、负责任的应用。     
针对 Google AI Edge SDK 的使用由于 EAP 申请未获得通过，因此目前还无法调研 API 的使用。       
一些应用案例：      
 - Pixel 录音机应用使用 Gemini Nano 和 AICore 来为设备端摘要功能提供支持。
 - Gboard 将 Gemini Nano 与 AICore 结合使用，在设备上提供准确的智能回复功能。


