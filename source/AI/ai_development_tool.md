---
title: AI 开发工具
categories: AI
comments: true
tags: [AI]
description: AI 开发工具
date: 2016-7-2 10:00:00
---

## Hugging face

[Hugging face 官网](https://huggingface.co/)    
可以说是AI开发者的GitHub，提供了模型、数据集（文本|图像|音频|视频）、类库（比如transformers|peft|accelerate）、教程等。目前大概有50多万个针对多个领域的开源预训练模型和12万多个数据集。    
缓存路径：`~/.cache/huggingface/modules/transformers_modules/`    
我们使用from_pretrained加载模型时，如果用的路径是repo id，那么就会从huggingface上下载并缓存到本地。    
https://zhuanlan.zhihu.com/p/675078682    

## Colab

[Colaboratory](https://colab.research.google.com/)简称 Colab）,是Google提供的一个免费的云服务，可以在浏览器中编写和执行 Python 代码，并且无需任何配置可以免费使用 GPU（虽然免费的只能用T4GPU，如果想要使用A100或者V100就需要付费了）。借助 Colab，只需使用几行代码，即可导入图像数据集、用图像数据集训练图像分类器，以及评估模型等等。Colab 笔记本会在 Google 的云服务器中执行代码，也就是说，无论您所用机器的功能如何，您都可以利用 Google 硬件（包括 GPU 和 TPU）的强大性能。只要有个浏览器即可。在 Hugging face 以及Langchain上提供的一些示例代码，可以直接在 Colab 上运行。    

## Google AI Studio

[Google AI Studio 快速入门](https://ai.google.dev/tutorials/ai-studio_quickstart?hl=zh-cn)    

Google AI Studio 是一个基于浏览器的 IDE，用于使用生成模型进行原型设计。通过 Google AI Studio，可以快速试用模型并针对不同的 Prompt 和模型参数进行实验。在构建出满意的结果后，可以将对应代码导出到自己的应用程序中。    

## Jupyter

1.安装 Anaconda，在 Anaconda 环境中启动 `jupyter notebook`    
2.通过`pip install jupyterlab`安装。    

安装后两种使用方式：    

 - 直接启动 jupyterlab，
 - VSCode 的 Jupyter插件启动，在VSCode插件库中搜索Jupyter安装即可。    

创建 hello.ipynb 文件，然后用VSCode打开即可编辑运行python代码了。    
anaconda jupyter TAB 键代码提示    

## Anaconda

下载对应版本并且安装：进入 https://www.anaconda.com/download# 页面会根据当前的环境选择合适的下载。或者进入 https://www.anaconda.com/download#downloads 自行选择。    
安装：    

```
bash Anaconda3-2024.02-1-Linux-x86_64.sh
```

下面命令可以进入 anaconda 的UI界面。    

```
~/install/anaconda3/bin$ ./anaconda-navigator
```

 - 查看安装结果：conda info    
 - 查看版本：conda -V    
 - 查看已安装虚拟环境列表 conda env list     
 - 设置conda 的基础环境在启动终端时不被激活：conda config --set auto_activate_base false    
 - 创建新环境 conda create -n <your_env_name> python=x.x    
 - 删除已有的虚拟环境 conda remove -n <your_env_name> --all    
 - 激活环境：conda activate <your_env_name>    
 - 退出环境：conda deactivate <your_env_name>    
 - 复制环境：conda create -n <new_env_name> --clone <origin_env_name>     
 - 查看已安装的包：conda list    
 - 搜索包：conda search <package_name1>    
 - 安装包：conda install <package_name1> <package_name2> 或者 pip install <package_name1> <package_name2>    
 - 卸载包：conda remove <package_name> 或者 pip remove <package_name>     

Anaconda卸载：     
删除Anaconda3文件夹：     

```
 rm -rf ~/anaconda3
```

删除相关隐藏文件：     
```
rm -rf ~/.condarc ~/.conda ~/.continuum
```

在环境变量中删除anaconda：     
```
 打开 ~/.bashrc (例如: vim ~/.bashrc)，找到与conda 相关的，注释掉即可。
```




