---
title: MCP 协议
categories: AI
comments: true
tags: [AI]
description: MCP 协议
date: 2016-7-2 10:00:00
---

## MCP 基础知识
[MCP协议官网](https://modelcontextprotocol.io/introduction)     
https://mcp-docs.cn     
### MCP 概念
MCP（Model Context  Protocol，模型上下文协议） 是由 Anthropic  推出的一种开放标准，旨在统一大型语言模型（LLM）与外部数据源和工具之间的通信协议。MCP 使得 AI 应用能够安全地访问和操作本地及远程数据，为 AI 应用提供了连接万物的接口。    
可以把将 MCP 想象成用于 AI 应用的 USB-C 端口。正如 USB-C 提供了一种将设备连接到各种外围设备和配件的标准化方法一样，MCP 也提供了一种将 AI 模型连接到不同数据源和工具的标准化方法。    
目前对于 MCP Client 和 Server 开发提供了 Kotlin、Python 和 TypeScript 三种版本的 SDK。    
### MCP 通用架构    
MCP 的核心遵循客户端-服务器架构，其中主机应用程序可以连接到多个服务器：    

<img src="/images/ai_mcp_introduce/1.png" width="750" height="515"/>    

 - MCP Hosts：像 Claude 桌面 APP、IDEs 或想要通过 MCP 访问数据的 AI 工具的程序 (这里可以理解集成了 MCP Client 的 LLM 的端侧应用，比如: Claude 的客户端， Cline 等)
 - MCP Client：与 MCP Server 保持 1:1 连接的协议客户端。(通常不用用户去实现，可以和 MCP Hosts 合在一起看作为 MCP Client)
 - MPC Servers：轻量级的程序，每个程序通过 MCP 规范暴露指定的功能；（需要实现各种能力，比如访问数据库，请求网络服务等）
 - Local Data Sources: 您的计算机文件、数据库以及 MCP 服务器可以安全访问的服务；
 - Remote Services：外部系统是指可以通过互联网（例如通过API接口）访问的系统，MCP服务器可以连接到这些系统以获取或交换数据。
### 传输机制
MCP 支持多种传输机制：    
1. Stdio ：使用标准输入/输出进行双向通信，一个服务器进程只能与启动它的客户端通信（1:1 关系），适用于本地进程    
<img src="/images/ai_mcp_introduce/2.png" width="512" height="577"/>    

2. SSE：使用服务器发送事件进行服务器到客户端的消息传递，适用于远程通信。服务端作为独立进程运行，可以处理多个客户端连接。    
<img src="/images/ai_mcp_introduce/3.png" width="430" height="523"/>    

Server-Sent Events (SSE) 是一种服务器推送技术，允许服务器向客户端发送实时更新的数据流。与传统的客户端请求-服务器响应模式不同，SSE 允许服务器在有新数据时主动向客户端推送数据，而不需要客户端不断地轮询服务器。     
主要特点：     
- 单向通信：服务器向客户端推送数据
- 基于 HTTP：
- 长链接
- 自动重连

https://zhuanlan.zhihu.com/p/11457484657

### MCP Server
MCP Server 提供 3 种主要类型的功能：    
1. 资源（Resources）：类似文件的数据，可以被客户端读取，如文件内容、数据库记录、图片、日志等等。
2. 工具（Tools）：可以被 LLM 调用的函数（需要用户批准）。
3. 提示（Prompts）：预先编写的模板，帮助用户完成特定任务。

目前传递MCP Server都是和Client运行在本地，MCP计划支持远程 MCP Server的计划。允许客户端通过互联网安全地连接到 MCP 服务器。    
https://modelcontextprotocol.io/development/roadmap    
## MCP 实践
### MCP Server 的应用和开发
我们使用 VSCode 的Cline 插件来作为 MCP Host来进行MCP Server 的开发。     
#### Remote Services
目前有很多社区贡献了大量的MCP Server来提供服务对LLM进行扩展。
https://github.com/modelcontextprotocol/servers
我们以 Obsidian Markdown Notes 来介绍一下 MCP Server 的使用。它提供了 Markdown 文件的搜索和读取功能。    
配置 MCP 客户端：    
点击 Cline 插件的 “Configure MCP Servers” 打开配置文件，配置后我们存放本地文档的目录。    
```
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-obsidian",
        "/home/meizu/test/test/"
      ]
    }
  }
}
```
那么该服务就出现了 Cline 插件的服务列表中：      

<img src="/images/ai_mcp_introduce/4.png" width="539" height="507"/>

它提供了两个Tools：read_notes和search_notes来提供对文档的读取和搜索功能。     
在插件中配置对MCP Server的功能介绍：      

<img src="/images/ai_mcp_introduce/5.png" width="597" height="189"/>

下面来执行一个任务：    

<img src="/images/ai_mcp_introduce/6.png" width="586" height="860"/>

<img src="/images/ai_mcp_introduce/7.png" width="531" height="717"/>


#### Local Services

下面基于 Python SDK 来实现一个本地 MCP 服务。     
执行下面命令来创建一个 python 工程。    
```
# python 版本需要大于 3.10
uv init weather -p 3.11
cd weather
# Create virtual environment and activate it
uv venv
source .venv/bin/activate
# Install dependencies
uv add "mcp[cli]" httpx
# Create our server file
touch weather.py
```
如果参考 quickstart ，使用 FastMCP 来构建服务器，在 Cline 中运行会有下面的报错：      
```
INFO Processing request of type server.py:432 ListToolsRequest INFO Processing request of type server.py:432 ListResourcesRequest INFO Processing request of type server.py:432 ListResourceTemplatesRequest
```

是由Cline插件引起的。需要使用 low level api 来规避这个问题。https://github.com/cline/cline/issues/1272    
因此参考这里的代码：https://github.com/modelcontextprotocol/python-sdk/tree/main/examples    

```
import anyio
import sys
import click
import httpx
import mcp.types as types
from mcp.server.lowlevel import Server


async def fetch_website(
    city: str,
) -> list[types.TextContent | types.ImageContent | types.EmbeddedResource]:
    return [types.TextContent(type="text", text=city+"目前天气晴，温度20度，微风")]


@click.command()
@click.option("--port", default=8000, help="Port to listen on for SSE")
@click.option(
    "--transport",
    type=click.Choice(["stdio", "sse"]),
    default="stdio",
    help="Transport type",
)
def main(port: int, transport: str) -> int:
    app = Server("mcp-website-fetcher")

    @app.call_tool()
    async def fetch_tool(
        name: str, arguments: dict
    ) -> list[types.TextContent | types.ImageContent | types.EmbeddedResource]:
        if name != "getWeather":
            raise ValueError(f"Unknown tool: {name}")
        if "city" not in arguments:
            raise ValueError("Missing required argument 'url'")
        return await fetch_website(arguments["city"])

    @app.list_tools()
    async def list_tools() -> list[types.Tool]:
        return [
            types.Tool(
                name="getWeather",
                description="Get weather forecast for a location.",
                inputSchema={
                    "type": "object",
                    "required": ["city"],
                    "properties": {
                        "city": {
                            "type": "string",
                            "description": "the city want to get weather info",
                        }
                    },
                },
            )
        ]

    if transport == "sse":
        from mcp.server.sse import SseServerTransport
        from starlette.applications import Starlette
        from starlette.routing import Mount, Route

        sse = SseServerTransport("/messages/")

        async def handle_sse(request):
            async with sse.connect_sse(
                request.scope, request.receive, request._send
            ) as streams:
                await app.run(
                    streams[0], streams[1], app.create_initialization_options()
                )

        starlette_app = Starlette(
            debug=True,
            routes=[
                Route("/sse", endpoint=handle_sse),
                Mount("/messages/", app=sse.handle_post_message),
            ],
        )

        import uvicorn

        uvicorn.run(starlette_app, host="0.0.0.0", port=port)
    else:
        from mcp.server.stdio import stdio_server

        async def arun():
            async with stdio_server() as streams:
                await app.run(
                    streams[0], streams[1], app.create_initialization_options()
                )

        anyio.run(arun)

    return 0
    
if __name__ == "__main__":
    print("Hello from weather!")
    sys.exit(main())
```

配置 Client ：     
点击Cline 插件中的 “Configure MCP Servers”，就会打开 MCP 客户端的配置文件，将 weather 服务添加进去。     
```
{
  "mcpServers": {
    "obsidian": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-obsidian",
        "/home/meizu/test/test/"
      ]
    },
    "weather": {
      "command": "/home/meizu/.local/bin/uv",
      "args": [
        "--directory",
        "/home/meizu/Documents/Cline/MCP/weather1",
        "run",
        "weather-simple.py"
      ],
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

MCP 客户端列出可用的 Server以及 Tools 列表和资源列表。     
<img src="/images/ai_mcp_introduce/8.png" width="585" height="536"/>

<img src="/images/ai_mcp_introduce/9.png" width="591" height="607"/>

执行任务进行测试：    
<img src="/images/ai_mcp_introduce/10.png" width="600" height="675"/>


<img src="/images/ai_mcp_introduce/11.png" width="592" height="866"/>

### MCP Client

https://blog.csdn.net/cr7258/article/details/145433427    

## MCP 调试

使用 MCP Inspector 调试 MCP Server。MCP Inspector 是一个交互式的开发者工具，专门用于测试和调试 MCP 服务器。它提供了一个图形化界面，让开发者能够直观地检查和验证 MCP 服务器的功能。     
执行以下命令可以启动 MCP Inspector：    
```
$ mcp dev weather1.py
```
会安装 inspector 工具，然后打开 `http://localhost:5173` 进行调试。    
点击 connect 进行链接服务器。    

<img src="/images/ai_mcp_introduce/12.png" width="640" height="327"/>

点击 List Tools展示可用工具列表：     
<img src="/images/ai_mcp_introduce/13.png" width="640" height="181"/>

输入参数进行调试：    

<img src="/images/ai_mcp_introduce/14.png" width="640" height="125"/>

## MCP 方案落地场景 

https://zhuanlan.zhihu.com/p/16948315081    
可以提供了统一的 API 构建Agent和工作流，相对于对于目前的一些 Agent 平台来说更加开放，可以更好第三方的支持。    

## 相关文章

[AI重要发展趋势：MCP 技术科普](https://xiangyangqiaomu.feishu.cn/wiki/PXAKwEgLQir9rkkV1zjcGnMHntg?fromScene=spaceOverview)    

