---
title: 大模型推理框架 llama.cpp
categories: AI
comments: true
tags: [AI]
description: 大模型推理框架 llama.cpp
date: 2024-6-5 10:00:00
---


[llama](https://github.com/ggerganov/llama.cpp)是一个C++编写的轻量级开源大模型推理框架，可以支持在消费级普通设备上本地部署运行大模型。      
llama.cpp的核心是一个优化的量化推理引擎。这个引擎能够高效地在CPU上执行量化模型的推理任务。它通过一系列的优化技术，如使用定点数代替浮点数进行计算、批量处理和缓存优化等，来提高推理速度并降低功耗。    
## GGUF 格式文件

GGUF 文件全称是 GPT-Generated Unified Format，是由 llama.cpp 的创始人 Georgi Gerganov 定义发布的一种大模型文件格式。         
GGUF 就是一种二进制格式文件的规范，原始的大模型预训练结果经过转换后变成GGUF格式可以更快地被载入使用，也会消耗更低的资源。原因在于GGUF采用了多种技术来保存大模型预训练结果，包括采用紧凑的二进制编码格式、优化的数据结构、内存映射等。     

 - PyTorch中使用的pickle格式加载文件时可能会执行任意代码，从而带来安全风险，safetensors提供了一个更安全的选择。     
 - safetensors 重点关注于安全性，对性能和跨平台交换等方面关注较少。虽然safetensors也被很多厂商采用，但是在大模型高效序列化、数据压缩、量化等方面不足。而且它只保存了张量数据，没有任何关于模型的元数据信息。     
 - GGUF格式因为良好的设计，通过各种优化手段实现了快速的模型加载，这对于需要频繁载入不同模型的场景尤为重要，同时将模型的元数据信息压缩到二进制文件中，对于跨平台操作和传输都非常有价值。     

https://www.datalearner.com/blog/1051710596054633     

很多厂商也会开源 GGUF 格式的大模型，可以在 huggingface 上Libraries 里面选中 GGUF 来过滤 GGUF 格式的模型。     

[GGUF格式的大模型文件是什么意思？gguf是什么格式？如何使用？为什么有GGUF格式的大模型文件？GGUF大模型文件与GGML的差异是啥？](https://www.datalearner.com/blog/1051705718835586)     
[Llama.cpp量化简明手册](http://www.baidu.com/baidu?tn=34046034_10_dg&ie=utf-8&wd=llmma.cpp)     
llama.cpp 官方提供了转换脚本，可以将pt格式的预训练结果以及 safetensors 模型文件转换成 GGUF 格式的文件。转换的时候也可以选择量化参数，降低模型的资源消耗。     

## ggml

https://github.com/ggerganov/ggml     
ggml是一个用 C 和 C++ 编写、专注于 Transformer 架构模型推理的机器学习库。该项目完全开源，处于活跃的开发阶段，开发社区也在不断壮大。ggml 和 PyTorch、TensorFlow 等机器学习库比较相似。      

## 本地部署

下载 GGUF 格式的大模型文件。     

huggingface 有很多开源的 GGUF 格式的大模型，可以在搜索时在 Libraries 里面选择 GGUF 来过滤 GGUF 格式大模型文件。         

https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/tree/main     

里面有很多量化格式的文件，我们选择 qwen2.5-3b-instruct-q8_0.gguf 下载。     

编译：     

根目录下执行 `make`。     

运行：     

具体参数参考：https://github.com/ggerganov/llama.cpp/blob/master/examples/main/README.md     

文本生成模式，输入提示补全对话：     
```
./llama-cli -m ../models/Qwen2.5-3B-Instruct-GGUF/qwen2.5-3b-instruct-q8_0.gguf --prompt "很久很久以前，有一个国王"
```

对话模式：     

```
./llama-cli -m ../models/Qwen2.5-3B-Instruct-GGUF/qwen2.5-3b-instruct-q8_0.gguf -cnv -p "你是一位诗人，擅长写七言绝句，能够根据主题要求写出优美的七言绝句"
```

## 端侧部署

llama.cpp 提供了 Android 端侧部署的例子 llama.android。直接 AS 导入就可以简单运行。     

[Android Build](https://github.com/ggerganov/llama.cpp/blob/master/docs/android.md)     

## 推理流程介绍

推理流程可以分为5个子功能模块：     
初始化：模型和系统提示词初始化。     
用户输入：等待用户输入文本信息。     
推理预测：这个是大语言模型的核心能力之一，它需要分析上下文（系统提示词、用户输入、已推理的内容）再进一步完成下一个词语（token）的预测。     
采样：这个是大语言模型的另一个核心能力，它需要从分析预测的结果中选择一个token，并将它作为输入反向发送给分析预测模块继续进行，直到输出结束（EOS）。     
输出：把大模型的输出转换为自然语言输出。     
     
```
main()
    // 初始化
    common_params_parse // 解析参数
        common_params_parse
            common_params_parser_init
    common_init()
    llama_backend_init()
    llama_numa_init
    common_init_from_params // 初始化
        llama_load_model_from_file() // 创建模型
            llama_model_load()
                llama_model_loader // 创建 llama_model_loader 对象
                    gguf_init_from_file()
                llm_load_arch()
                llm_load_hparams() //超参数解析
                llm_load_vocab() // 根据模型配置 llama_vocab 信息，包括编码类型等
                llm_load_print_meta() // 打印加载的模型信息
                llm_load_tensors() // 加载模型张量信息
        llama_new_context_with_model //创建推理上下文
        llama_lora_adapter_init() //
    ggml_threadpool_new// 创建线程池
    llama_attach_threadpool
    // 初始化系统提示词
    chat_add_and_format // 格式化system prompt
    common_tokenize // 按 token 进行分词编码，比如QWen采用 BPE (LLAMA_VOCAB_TYPE_BPE) 分词编码算法
    common_sampler_init
    
    // 开始循环推理流程
    common_sampler_accept(false)
    // 推理
    llama_batch_get_one
    llama_decode
        llama_decode_internal
            llama_build_graph //构建推理计算图
            llama_graph_compute //根据计算图调用对应算子
    //采样
    common_sampler_sample
    common_sampler_accept(true)
    //转换
    common_token_to_piece //将token转成自然语言
```



1.初始化:

初始化这部分主要包括大模型的初始化和系统提示词初始化。     
大模型初始化：     
这部分可以看上面的调用流程，主要包括参数解析，系统初始化，创建模型和推理上下文，创建ggml线程池等。     

```
    if (!common_params_parse(argc, argv, params, LLAMA_EXAMPLE_MAIN, print_usage)) {
        return 1;
    }

    common_init();
    ......
    llama_backend_init();
    llama_numa_init(params.numa);

```

```
    // load the model and apply lora adapter, if any
    LOG_INF("%s: load the model and apply lora adapter, if any\n", __func__);
    common_init_result llama_init = common_init_from_params(params);

    model = llama_init.model;
    ctx = llama_init.context;
```

```
    struct ggml_threadpool * threadpool = ggml_threadpool_new(&tpp);
    if (!threadpool) {
        LOG_ERR("%s: threadpool create failed : n_threads %d\n", __func__, tpp.n_threads);
        return 1;
    }

    llama_attach_threadpool(ctx, threadpool, threadpool_batch);
```

系统提示词初始化：     
把终端输入的提示词进行格式化，然后进行分词编码，存放到 embd_inp 容器中。         

```
    {
        auto prompt = (params.conversation && params.enable_chat_template && !params.prompt.empty())
            ? chat_add_and_format(model, chat_msgs, "system", params.prompt) // format the system prompt in conversation mode
            : params.prompt;
        if (params.interactive_first || !params.prompt.empty() || session_tokens.empty()) {
            LOG_DBG("tokenize the prompt\n");
            embd_inp = common_tokenize(ctx, prompt, true, true);
        } else {
            LOG_DBG("use session tokens\n");
            embd_inp = session_tokens;
        }

        LOG_DBG("prompt: \"%s\"\n", prompt.c_str());
        LOG_DBG("system tokens: %s\n", string_from(ctx, embd_inp).c_str());
    }
```

大模型 prompt 格式：     

```
<|im_start|>system
You are a helpful assistant<|im_end|>
<|im_start|>user
Hello<|im_end|>
<|im_start|>assistant
Hi there<|im_end|>
```

比如格式化后的 system prompt 是：     

```
<|im_start|>system
你是一位诗人，擅长写七言绝句，能够根据主题要求写出优美的七言绝句<|im_end|>
```

进行分词编码后就是：     

```
'<|im_start|>':151644, 'system':8948, '':198, '':56568, '':109182, '':106926, '':3837, '':107618, '':61443, '':99612, '':77144, '':99631, '':99700, '':3837, '':100006, '':100345, '':100220, '':101882, '':112672, '':90172, '':101607, '':99612, '':77144, '':99631, '':99700, '<|im_end|>':151645, '':198
```

token 对应编码：     

```
151644 -> '<|im_start|>'
  8948 -> 'system'
   198 -> '
'
 56568 -> '你'
109182 -> '是一位'
106926 -> '诗人'
  3837 -> '，'
107618 -> '擅长'
 61443 -> '写'
 99612 -> '七'
 77144 -> '言'
 99631 -> '绝'
 99700 -> '句'
  3837 -> '，'
100006 -> '能够'
100345 -> '根据'
100220 -> '主题'
101882 -> '要求'
112672 -> '写出'
 90172 -> '优'
101607 -> '美的'
 99612 -> '七'
 77144 -> '言'
 99631 -> '绝'
 99700 -> '句'
151645 -> '<|im_end|>'
   198 -> '
'
```

将 system prompt 载入大模型：     
开启第一次循环：     
首先判断是否需要推理，embd 容器中放入的就是需要推理预测的 tokens。此时是空的，并不需要进行推理。     
```
    while ((n_remain != 0 && !is_antiprompt) || params.interactive) {
        // predict
        if (!embd.empty()) {/
```

由于此时还没有进行推理，下面的条件不满足，n_consumed 表示 embd_inp 中已经放入 embd 容器中进行推理的 token 个数。         

```
        if ((int) embd_inp.size() <= n_consumed && !is_interacting) {
```

然后就会把 embd_inp 输入 prompt 放到 embd 中。     

```
            // some user input remains from prompt or interaction, forward it to processing
            while ((int) embd_inp.size() > n_consumed) {
                embd.push_back(embd_inp[n_consumed]);

                // push the prompt in the sampling context in order to apply repetition penalties later
                // for the prompt, we don't apply grammar rules
                common_sampler_accept(smpl, embd_inp[n_consumed], /* accept_grammar= */ false);

                ++n_consumed;
                if ((int) embd.size() >= params.n_batch) {
                    break;
                }
            }
```

这里会将 prompt  载入上下文中，用户的 system 不需要执行推理，只需要调用 common_sampler_accept(false) 即可。     
此时的 embd 容器中为：     

```
<|im_start|> system 
 你 是一位 诗人 ， 擅长 写 七 言 绝 句 ， 能够 根据 主题 要求 写出 优 美的 七 言 绝 句 <|im_end|> 
```


第二次循环：     

由于前面已经将 system prompt 装入到 embd 中，因此，embd 不为空，可以执行推理操作。         

embd 为：     
```
<|im_start|> system 
 你 是一位 诗人 ， 擅长 写 七 言 绝 句 ， 能够 根据 主题 要求 写出 优 美的 七 言 绝 句 <|im_end|> 
```

批量处理 token llama_batch_get_one()，并进行预测 llama_decode()。     

```
            for (int i = 0; i < (int) embd.size(); i += params.n_batch) {
                int n_eval = (int) embd.size() - i;
                if (n_eval > params.n_batch) {
                    n_eval = params.n_batch;
                }

                LOG_DBG("eval: %s\n", string_from(ctx, embd).c_str());

                if (llama_decode(ctx, llama_batch_get_one(&embd[i], n_eval, n_past, 0))) {
                    LOG_ERR("%s : failed to eval\n", __func__);
                    return 1;
                }
                // 表示已经执行推理的tokens个数
                n_past += n_eval;

                LOG_DBG("n_past = %d\n", n_past);
                // Display total tokens alongside total time
                if (params.n_print > 0 && n_past % params.n_print == 0) {
                    LOG_DBG("\n\033[31mTokens consumed so far = %d / %d \033[0m\n", n_past, n_ctx);
                }
            }
```

清理 embd。     

```
embd.clear();
```

由于 满足下面两个条件     

```
if ((int) embd_inp.size() <= n_consumed) {
    if (n_past > 0 && is_interacting) {
```

等待用户输入。     

```

```

输入完成后调用`chat_add_and_format(model, chat_msgs, "user", std::move(buffer))`格式化 user prompt。     

```
<|im_start|> system 
 你 是一位 诗人 ， 擅长 写 七 言 绝 句 ， 能够 根据 主题 要求 写出 优 美的 七 言 绝 句 <|im_end|> 
 
 <|im_start|> user 
 春天 
 <|im_end|> 
 <|im_start|> assistant 
```

把还没有喂给大模型的  token 填入 embd 中，并且载入上下文中。         


```
            while ((int) embd_inp.size() > n_consumed) {
                embd.push_back(embd_inp[n_consumed]);

                // push the prompt in the sampling context in order to apply repetition penalties later
                // for the prompt, we don't apply grammar rules
                common_sampler_accept(smpl, embd_inp[n_consumed], /* accept_grammar= */ false);

                ++n_consumed;
                if ((int) embd.size() >= params.n_batch) {
                    break;
                }
            }
```

由于 embd_inp 中有大模型没有推理完成的 token ，所以这一次循环不会等待用户输入。直接开始下一次循环。     
开始推理逻辑：     

embd 信息：     
```
 <|im_start|> user 
 春天 
 <|im_end|> 
 <|im_start|> assistant 
```

```
                if (llama_decode(ctx, llama_batch_get_one(&embd[i], n_eval, n_past, 0))) {
                    LOG_ERR("%s : failed to eval\n", __func__);
                    return 1;
                }
```

推理     

```
                if (llama_decode(ctx, llama_batch_get_one(&embd[i], n_eval, n_past, 0))) {
                    LOG_ERR("%s : failed to eval\n", __func__);
                    return 1;
                }
```

采样     

```
            const llama_token id = common_sampler_sample(smpl, ctx, -1);

            common_sampler_accept(smpl, id, /* accept_grammar= */ true);
```

转换     

转换成自然语言     

```
                const std::string token_str = common_token_to_piece(ctx, id, params.special);
```

这个时候输出了第一个 token "春"，     
此时 `(int) embd_inp.size() == n_consumed)`，因此不需要等待用户输入，开启下一次循环。     

把“春”输送给分析预测进行下一个token的预测，输出 “风吹”，直至推理完成，发出 eos-token，表示本轮推理结束。     

```
'<|im_end|>':151645
```

然后等待用户输入。     





