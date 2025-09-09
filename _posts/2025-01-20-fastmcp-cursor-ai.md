---
layout: post
title: "FastMCP로 자체 MCP 서버 구현하고 Cursor AI에 등록하기"
categories: "AI"
tags: ["MCP"]
sidebar: ['article-menu']
---

이 글에서는 Python의 `FastMCP` 패키지를 사용하여 자신만의 MCP(Model-Centric Programming) 서버를 구축하고, 이를 Cursor AI 에디터에 연동하는 방법을 소개합니다.

### **FastMCP 패키지**

FastMCP는 간단한 데코레이터를 사용하여 MCP 서버를 쉽게 만들 수 있도록 도와주는 파이썬 라이브러리입니다. 자세한 내용은 아래 GitHub 저장소에서 확인할 수 있습니다.

- [https://github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp)

### **로컬 MCP 서버 작성**

다음은 FastMCP를 사용하여 간단한 MCP 서버를 작성하는 예제 코드입니다.

```python
from fastmcp import FastMCP

# MCP 서버 생성
mcp = FastMCP("Demo")


# 'get_message'라는 이름의 도구(tool) 추가
@mcp.tool()
def get_message(message) -> str:
    """동작 구현"""
    return message


# 동적인 greeting 리소스 추가
@mcp.resource("greeting://{name}")
def get_greeting(name: str) -> str:
    """Get a personalized greeting"""
    return f"Hello, {name}!"


# 프롬프트 정의
@mcp.prompt()
def prompt(message: str) -> str:
    return f"""
당신은 데이터의 메시지를 가공하는 AI 어시스턴트입니다.

사용 가능한 도구:
- get_message(message) - 메시지 확인

사용자 메시지: {message}
""" 


if __name__ == "__main__":
    mcp.run()
```

### **로컬 MCP 서버 실행**

작성한 파이썬 파일을 다음과 같이 실행하여 MCP 서버를 시작할 수 있습니다.

```bash
# main.py 파일을 개발 모드로 실행
mcp dev main.py
```

서버가 정상적으로 실행되면 `http://localhost:5173` 주소로 접속하여 콘솔 화면을 확인할 수 있습니다.

### **Cursor AI에 MCP 등록**

이제 로컬에서 실행 중인 MCP 서버를 Cursor AI 에디터에 등록할 차례입니다.

1.  Cursor AI의 설정(Settings) 화면으로 이동합니다.
2.  'add new global MCP Server'를 클릭합니다.
3.  다음과 같이 로컬 MCP 서버 정보를 JSON 형식으로 입력합니다.

    ```json
    {
      "mcpServers": {
        "lower_upper": {
          "command": "uv",
          "args": [
            "run --with mcp mcp run /path/to/your/main.py"
          ]
        }
      }
    }
    ```

![](/assets/images/posts/2024-01-20-fastmcp-cursor-ai-1.png)

이제 Cursor AI에서 직접 만든 MCP 서버의 도구와 리소스를 활용할 수 있습니다.
