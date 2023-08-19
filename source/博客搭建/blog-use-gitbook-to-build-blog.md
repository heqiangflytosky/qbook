---
title: Gitbook 博客搭建
categories: Gitbook
comments: true
keywords: Gitbook, Blog, GitHub
tags: [Gitbook, Blog, GitHub]
description: 使用Gitbook在GitHub上搭建个人博客
date: 2016-01-10 10:00:00
---

在GitHub上托管电子书源码，使用GitBook生成静态的电子书。

# 搭建环境

1.安装Node.js
2.安装 GitBook
`npm install gitbook-cli -g`
查看 GitBook 是否安装成功
`gitbook -V` （大写的 V ）
3.创建项目
`mkdir mybook `
`cd mybook `
`gitbook init`
4.启动项目
`gitbook serve`
不仅生成文件，还会启动网站服务。
如果不想使用 4000 端口，想要使用 9520 端口时
`gitbook serve -p 9520`
5.编译项目
`gitbook build`
6.查看所有可用的 gitbook 版本
`gitbook ls-remote`
7.安装指定的 gitbook 版本
`gitbook fetch beta`（版本号）
8.编译时，输出目录详细的记录包括 debug
`gitbook build ./ --log=debug --debug`
只负责生成静态文件。
9.安装插件
`gitbook install`
终端执行 `gitbook install` 可以安装 book.json 配置的插件，下载的插件会在 node_modules 文件夹。

# 配置

## language

## root

使用子目录(如example-docs/)来存储项目的文档。您可以在 book.json 中通过配置 root 选项告诉 GitBook 在那里找到根目录。

```
"root": "./example-docs"
```

## links
在左侧导航栏添加链接信息：

```
    "links": {
        "sidebar": {
            "我的GitHub": "https://github.com/heqiangflytosky"
        }
    },
```

## plugins

配置使⽤的插件
"plugins": [
    "expandable-chapters"
]
添加新插件之间需要运⾏ `gitbook install` 来安装新的插件

## pluginsConfig



# 插件


plugins 是配置新增或删除插件的位置,而 Gitbook 默认自带有 5 个插件：

 - sharing：右上角分享功能
 - font-settings：字体设置（左上方的"A"符号）
 - livereload：为 GitBook 实时重新加载
 - highlight： 代码高亮
 - search： 导航栏查询功能（不支持中文）

pluginsConfig 是插件配置的地方
特别说明 系统自带插件可通过 在插件名前面加减号的方式去除掉，如 `-sharing`。

```
"plugins": [
  "-search"
]
```

## 安装插件

在book.json中写上插件的名称，然后在目录中执行 
`gitbook install`，会生成node_modules文件夹，配置的插件也会自动下载到该目录下。

```
{
    "plugins":[
        "expandable-chapters"
    ]
}
```

## 插件介绍

下面介绍一些有用的插件：

 - expandable-chapters：目录折叠
 - chapter-fold：和expandable-chapters插件配合使用，使导航目录使用更正常，以免出现导航栏问题。一个支持多层目录，一个是在目录前方加上箭头。使点击两个都有效。可以是目录默认为收起状态。
 - toggle-chapters：默认只在目录导航中显示章的标题，而不会显示小节的标题，点击每一章或者每一节会显示当前章或节的子目录，如果有的话，但是同时会收起其它之前展开的章节。
 - splitter：如果目录内容比较多，左边菜单栏显示不下，也可以使用插件来达到放大菜单栏宽度的目的
 - intopic-toc：装完这个插件可以在文章右侧展示文章目录。
 - splitter：侧边栏宽度调整，添加完插件后，在界面上 侧边栏可自行调整宽度。
 - back-to-top-button: 返回顶部，在页面篇幅过长时，在界面右下角自动添加上返回顶部的按钮。

 - github： 在右上角显示 github 仓库的图标链接  https://github.com/GitbookIO/plugin-github 

### hide-element： 隐藏元素

可以用来隐藏不想看到的元素，例如隐藏GitBook默认提示：Published with GitBook ，在book.json中加入以下内容：

```
{
  "plugins": [
    "hide-element"
  ],
  "pluginsConfig": {
	"hide-element": {
		"elements": [".gitbook-link"]
	}
  }
}
```

### insert-logo 插入logo

在左侧导航栏上方插入logo

```
{
  "plugins": [
    "insert-logo"
  ],
  "pluginsConfig": {
	"insert-logo": {
		"url": "../assets/logo.png",
		"style": "background: none"
	}
  }
}
```

### favicon 修改标题栏图标

设置浏览器选项卡标题栏的小图标。
在book.json中加入以下内容：

```
{
  "plugins": [
    "favicon"
  ],
  "pluginsConfig": {
        "favicon": {
            "shortcut": "images/favicon.ico",
            "bookmark": "https://github.com/heqiangflytosky/qbook/blob/gitbook/images/favicon.ico?raw=true"
        },
	}
  }
}
```

### tbfed-pagefooter 添加页脚

在每个文章下面标注版权信息和文章时间。
在book.json中加入以下内容：

```
{
  "plugins": [
    "tbfed-pagefooter"
  ],
  "pluginsConfig": {
	"tbfed-pagefooter": {
		"copyright": "Copyright &copy ruarua 2020",
		"modify_label": "该文章修订时间：",
		"modify_format": "YYYY-MM-DD"
	}
  }
}
```

### sharing-plus 分享页面

GitBook默认只有Facebook、Google+、Twiter、Weibo、Instapaper，插件可以有更多分享方式，也可关闭指定分享方式。
在book.json中加入以下内容：

```
{
  "plugins": [
    "-sharing","sharing-plus"
  ],
  "pluginsConfig": {
	"sharing": {
		  "facebook": "false",
		  "google": "false",
	      "twiter": "false",
		  "qq": "true",
		"all": [
			"facebook","google","twiter","qq"
		]
	}
  }
}
```

### code 代码

为代码块添加行号和复制按钮，复制按钮可关闭
单行代码无行号。

```
"code": {
        "copyButtons": false
      }
```

# 基于 GitHub Actions 配置自动部署

我们的源文件是放在 gitbook 目录，生成的静态文件放在 main 目录，基于 GitHub HomePage 来生成个人主页。
现在我们来配置一下自动部署，这里选择免费好用的 GitHub Actions。

在gitbook分支编写 `.github/workflows/gitbook.yml`，.yml 文件缩进很重要，YAML、YML在线编辑(校验)器可以检验yml格式是否正确。
如下供参考：

```
name: auto-generate-gitbook
on:                                 #在 gitbook 分支上进行push时触发
  push:
    branches:
    - gitbook

jobs:
  gitbook-to-main:
    runs-on: ubuntu-latest

    steps:
    - name: checkout gitbook
      uses: actions/checkout@v2
      with:
        ref: gitbook

    - name: install nodejs
      uses: actions/setup-node@v1

    - name: configue gitbook
      run: |
        npm install -g gitbook-cli
        gitbook install
        npm install -g gitbook-summary

    - name: generate _book folder
      run: |
        #book sm
        gitbook build
        #cp SUMMARY.md _book

    - name: push _book to branch main
      env:
        TOKEN: ${{ secrets.TOKEN }}
        REF: github.com/${{github.repository}}
        MYEMAIL: heqiangfly@163.com                  # ！！记得修改为自己github设置的邮箱
        MYNAME: ${{github.repository_owner}}
      run: |
        cd _book
        git config --global user.email "${MYEMAIL}"
        git config --global user.name "${MYNAME}"
        git init
        git remote add origin https://${REF}
        git add .
        git commit -m "Updated By Github Actions With Build ${{github.run_number}} of ${{github.workflow}} For Github Pages"
        git branch -M gitbook
        git push --force --quiet "https://${TOKEN}@${REF}" gitbook:main
```

还需要生成一个token，按照[官网生成](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)，注意要勾选 repo 即可。
然后进入到该仓库，点击Settiings -> Secrets -> New repository，添加TOKEN，将其命名为TOKEN。
当我们修改 gitbook 分支并push上去后，就自动构建并发布到了 main 分支。

# Github


# GitBook

注册 [GitBook](www.gitbook.com) 账号。

所有在 Gitbook.com 上的书的 http 地址为 http://{author}.gitbooks.io/{book}/，而书内容的地址是 http://{author}.gitbooks.io/{book}/content/。
但是你也可以使用你自定义的域名（GitBook 的免费功能）。域名可以绑定到你的主页或者内容上（或两者都）。

自定义域名：home -> Settings -> Account -> Publishing -> Bitbook subdomian -> edit，输入你喜欢的域名。




https://www.cnblogs.com/wukongnotnull/archive/2021/09/01/15216123.html
https://docs.gitbook.com/
https://blog.csdn.net/weixin_41815063/article/details/86504905
https://www.cnblogs.com/levywang/p/13569619.html

https://baijiahao.baidu.com/s?id=1666708486591555944&wfr=spider&for=pc
https://chrisniael.gitbooks.io/gitbook-documentation/content/index.html
Gitbook配置目录折叠 http://t.zoukankan.com/vielat-p-10217153.html
https://pudongping.com/posts/b00f410f.html

https://www.w3cschool.cn/gitbook/gitbook-px813ewg.html

https://blog.csdn.net/u013545389/article/details/123907541
https://blog.csdn.net/simplehouse/article/details/78766513?spm=1001.2101.3001.6661.1&utm_medium=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-78766513-blog-123907541.pc_relevant_antiscanv2&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-1-78766513-blog-123907541.pc_relevant_antiscanv2
https://blog.csdn.net/u013545389/article/details/123907541

https://blog.csdn.net/qq_46067720/article/details/110518499
https://www.jianshu.com/p/53fccf623f1c?utm_campaign=maleskine&utm_content=note&utm_medium=seo_notes&utm_source=recommendation
https://www.it610.com/article/1488978674559508480.htm
https://wenku.baidu.com/view/1eb32e55f142336c1eb91a37f111f18583d00c90.html

https://juejin.cn/post/6844903848914452488

http://www.voycn.com/article/dazaowanmeixiezuoxitonggitbookgithub-pagesgithub-actions
