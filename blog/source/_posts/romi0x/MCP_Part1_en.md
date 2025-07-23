---
title: "[Research] MCP (Model Context Protocol) Part 1 (en)"
author: romi0x
tags: [romi0x, research, mcp, llm]
categories: [Research]
date: 2025-07-24 17:00:00
cc: false
index_img: /2025/07/24/romi0x/MCP_Part1_en/index.png
---
Hello, this is romi0x!

Recently, a new approach to improving **context management** and **interaction efficiency** in AI models, especially in LLM environments, has emerged — it's called **MCP (Model Context Protocol)**. Today, I’ll briefly introduce what MCP is all about.

# What is MCP?

Popular AI models like GPT, Claude, and Gemini are already showing great performance across tasks like coding, document summarization, and Q&A. However, even these models have limitations.

One major limitation is that they **"cannot fully understand or retain context"**.

To address this issue, one of the proposed solutions is the **Model Context Protocol (MCP)**. MCP defines a **context object** that can be shared between multiple AI models, agents, and plugins, and enables them to exchange this context in a standardized way.

MCP was first introduced in **November 2024** by [Anthropic](https://www.anthropic.com/news/model-context-protocol), the creators of Claude. It’s an open protocol designed to help AI interact with various systems and data sources in a standardized manner, maintain consistent context, and become more effective at solving real-world problems.

Still not clear? Let’s borrow an analogy from the introduction of the MCP developer guide:

> **"MCP is the USB-C port for AI applications."**

![Source: https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/](MCP_Part1_en/image.png)  
Source: [https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/](https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/)

> MCP is an open protocol that standardizes how applications provide context to LLMs. Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect your devices to various peripherals and accessories, MCP provides a standardized way to connect AI models to different data sources and tools.  
> — [modelcontextprotocol.io](https://modelcontextprotocol.io/introduction)

Just as USB-C allows you to connect laptops, smartphones, mice, and monitors using one unified port, **MCP lets AI models connect to various data sources** (Google Drive, GitHub, Slack, etc.) using a single unified interface.

In other words, if a developer implements a connection using MCP just once, it can be reused across different LLMs in the future without re-implementation. Similarly, model providers don’t need to build integrations for every tool — they can simply follow the MCP standard.

![Source: https://techblog.woowahan.com/22342/](MCP_Part1_en/image1.png)  
Source: [https://techblog.woowahan.com/22342/](https://techblog.woowahan.com/22342/)

The Wooahan Brothers Tech Blog puts it like this:

> "MCP is a protocol that teaches AI models how to use a program."  
> Suddenly, this complicated term makes a lot more sense, doesn’t it?

---

# Why Do We Need MCP?

LLMs generally maintain context **only within the input token window**. GPT-4, for example, supports up to 128K tokens — which is long, but still limited to a **single session** and lacks long-term memory or external context tracking.

Real-world AI applications often rely on **multiple agents and plugins** working together.

- User → Summary plugin → Email analysis plugin → Calendar input plugin

To ensure each of these components understands the user's intent or workflow, context must be **shared in a standardized format**. That’s where MCP comes in.

---

# How MCP Works

MCP follows a simple **client-server architecture**.

![Source: https://modelcontextprotocol.io/introduction](MCP_Part1_en/image2.png)  
Source: [https://modelcontextprotocol.io/introduction](https://modelcontextprotocol.io/introduction)

Here are the components:

- **MCP Host**: The entity trying to access external data (e.g., Claude Desktop, AI-based IDEs)
- **MCP Client**: Maintains a dedicated 1:1 connection with the MCP server and sends requests
- **MCP Server**: Connects to actual data sources (Google Drive, GitHub, etc.) and provides the required data or services
- **Local/Remote Data Sources**: Files, databases, APIs — the resources the MCP server accesses

To give you an analogy:  
The **host** is like the CEO,  
the **server** is a department,  
the **client** is the department’s assistant,  
and **MCP is the standardized document template**.

The CEO fills out the form, the assistant delivers it to the department, and the department processes and responds — all in a unified and efficient way.

---

# Real-World Examples of MCP

Though still in its early stages, MCP is already being adopted by several companies and developer tools.

## Example 1: Claude + Google Drive Document Summarization

A user types: “Summarize last week’s meeting notes” into Claude Desktop.  
Claude uses the MCP client to connect with a local **Google Drive MCP server**, retrieves the document, and loads it into its context. It then summarizes the content and, if requested, saves the result back to Google Drive.

> In the past, developers had to manually integrate the Google Drive API, implement authentication, parse the file structure, etc. With MCP, all of this happens automatically using a standardized context format.

---

## Example 2: Claude + GitHub Code Review

A developer asks Claude: “What issues do you see in this pull request?”  
Claude accesses GitHub through the **GitHub MCP server**, loads PR changes, review comments, and commit history into context, then provides code review feedback.

> In this case, Claude isn’t just summarizing text — it’s performing a deep analysis based on the full context of the code changes and developer intent.

---

# How Is MCP Different From Existing Technologies?

MCP isn’t a completely new invention — it builds upon and extends limitations of existing tools and frameworks.

Its main focus is **sharing and persisting context across tools and sessions**.

| Category | Function Calling (OpenAI) | LangChain Memory | Vector Search + RAG | **MCP** |
|---------|-----------------------------|------------------|---------------------|---------|
| Purpose | Call external functions     | Maintain short-term memory | Document-based QA | **Standardize and share context** |
| Data Access | Real-time API call       | Past dialogue storage | Vector-based document search | **Files, APIs, databases, etc.** |
| Sharing Scope | One model session      | Within one model | Model-only          | **Across multiple tools/models** |
| Standardization | ❌ Model-dependent     | Framework-dependent | Non-standard formats | ✅ JSON-based spec |
| Security Control | ❌ Manual             | ❌ None           | ❌ None             | ✅ Client-server model |
| Scalability | Low                      | Low              | Medium              | ✅ **High (plugin-friendly)** |

---

Today we explored the overall concept of MCP.  
In the next part of this series, I’ll bring in some papers related to MCP — stay tuned! (Maybe... 😅)

---

# Ref

- [https://techblog.woowahan.com/22342/](https://techblog.woowahan.com/22342/)
- [https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/](https://norahsakal.com/blog/mcp-vs-api-model-context-protocol-explained/)
- [https://channel.io/ko/blog/articles/what-is-mcp-52c77e72](https://channel.io/ko/blog/articles/what-is-mcp-52c77e72)
