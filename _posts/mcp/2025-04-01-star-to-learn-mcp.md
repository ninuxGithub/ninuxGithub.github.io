---
title: Simple Usage of MCP
author: ninuxGithub
layout: post
date: 2025-04-01 14:50:13
description: "MCP"
tag: mcp
---

### Quick Start

    怎么在 Cursor 中添加mcp 服务
    2中方式配置了filesystem
    还有一个firecrawl  (需要安装本地server)

```json

{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "--yes",
        "@modelcontextprotocol/server-filesystem",
        "D:/project/his"
      ],
      "enabled": true
    },
    "filesystemV2": {
      "command": "C:\\Windows\\System32\\cmd.exe",
      "args": [
        "/c",
        "npx",
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "D:\\project\\his"
      ],
      "enabled": true
    },
    "mcp-server-firecrawl": {
      "command": "C:\\Windows\\System32\\cmd.exe",
      "args": [
        "/c",
        "npx",
        "-y",
        "firecrawl-mcp"
      ],
      "env": {
        "FIRECRAWL_API_URL": "http://localhost:3002"
      }
    }
  }
}
```



firecrawl 的安装步骤需要将仓库clone 下来, 然后通过docker desktop 来安装
如何指定文件夹安装windows docker ？

```shell
start /w "" "Docker Desktop Installer.exe" install --backend=wsl-2 --installation-dir=D:\Software\Docker --wsl-default-data-root=D:\Software\wsl --accept-license
```

进入下载好的firecrawl
执行docker-compose up , 启动的api 启动在端口3002

![docker 项目部署](/images/posts/start_docker_firecrawl.png)







    