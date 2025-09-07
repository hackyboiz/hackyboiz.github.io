---
title: "[Research] MCP (Model Context Protocol) Part 2 (en)"
author: romi0x
tags: [romi0x, research, mcp, llm]
categories: [Research]
date: 2025-09-02 17:00:00
cc: false
index_img: /2025/09/02/romi0x/MCP_Part2_en/index.png
---

Hello! This is romi0x~

Following up on the last article, this is Part 2 of the MCP series~

In the previous article, we covered the concept of MCP, its background, basic structure, and adoption cases. In this article, I've focused on how MCP works internally.

### Core Components of MCP (Host / Client / Server)

As mentioned in the last article, MCP is a protocol that allows various LLMs to define context objects and exchange them in a standardized way. I also briefly introduced the simple working mechanism of MCP in the previous article!

The components were added as an **MCP Host, an MCP Client, and an MCP Server.** Now, let's examine exactly how these components operate and what their structure is.

A simple summary of the three components is as follows:

| Component | Description | Examples |
| --- | --- | --- |
| **MCP Server** | The backend that holds actual resources | Google Drive MCP, GitHub MCP, etc. |
| **MCP Client** | An intermediary for communication between Host and Server | OpenAI's Function Router, etc. |
| **MCP Host** | An LLM or LLM app (Claude, GPT, etc.) | Claude Desktop, AI IDE |

Let's break down each of these components!

### (1) MCP Server

The **MCP Server** is a backend component that provides functionality to enable AI applications to connect with and operate in the external world. This server offers specialized functions for specific domains, such as file systems, email, travel itineraries, and databases.

For example, there can be a file system server for document management, an email server for message processing, and a travel server for trip planning.

Through the MCP server, an AI model can access resources, run tools, and perform various actions according to a specified interface.

This structure extends the LLM beyond a simple language model into an environment that can function like an app.

The MCP server provides its functionality through three main components: **Tools**, **Resources**, and **Prompts**.

| Component | Purpose | Controller | Example |
| --- | --- | --- | --- |
| **Tool** | Actions the model can perform | Model | Search for flights, send emails, add calendar events |
| **Resource** | Provides context data | Application | Calendars, documents, travel history, weather data |
| **Prompt** | Provides interaction templates | User | "Plan a vacation for me," "Summarize this meeting" |

Each element follows a standardized schema and URI structure and communicates via JSON-RPC.

### (1-1) Tools: Actions executed by AI

A **Tool** is an executable function that an AI model calls to perform tasks in the external world.

For example, actions like searching for flights, sending an email, or adding a calendar event can all be implemented as an MCP Tool.

### Structure

Each Tool has a clearly defined input/output structure using **JSON Schema**.

This helps the LLM generate accurate inputs without format errors when using a tool.

```
{
  "name": "searchFlights",
  "description": "Search for available flights",
  "inputSchema": {
    "type": "object",
    "properties": {
      "origin": { "type": "string" },
      "destination": { "type": "string" },
      "date": { "type": "string", "format": "date" }
    },
    "required": ["origin", "destination", "date"]
  }
}
```

### Protocol Methods

| Method | Description |
| --- | --- |
| `tools/list` | View the list of available tools |
| `tools/call` | Request to execute a specific tool |

Tool execution always requires the user's explicit approval. Before execution, the user can review the tool's description, input values, and expected results and decide whether to approve it. Automatic approval can also be set for trusted tools.

### (1-2) Resources – Providing Context Data

A **Resource** is an external data source provided in a format that an LLM can understand. It can be a document, calendar event, weather information, email, or a database record, and the model can accept and process it as context.

### Structure

A Resource is identified by a unique URI, using schemes like `file://`, `calendar://`, or `travel://`. It also includes metadata such as MIME type, title, and description.

```
{
  "uriTemplate": "weather://forecast/{city}/{date}",
  "name": "weather-forecast",
  "title": "Weather Forecast",
  "description": "Get weather forecast for any city and date",
  "mimeType": "application/json"
}
```

### Protocol Methods

| Method | Description |
| --- | --- |
| `resources/list` | View available resources |
| `resources/templates/list` | View resource templates |
| `resources/read` | Read data from a specific resource |
| `resources/subscribe` | Subscribe to resource changes (optional) |

Resources can generally be explored through a file browser, search bar, or category filtering UI. The LLM can automatically load resources selected by the application, or the user can select them manually. The selected resources are passed to the AI as context for use in question-answering, analysis, etc.

### (1-3) Prompts – Interaction Templates

A **Prompt** is a template that defines a standardized format for tasks users frequently perform, allowing them to be reused. Through a prompt, the model can receive more structured requests than natural language input and be guided toward a workflow appropriate for a specific domain.

### Structure

```
{
  "name": "plan-vacation",
  "title": "Plan a vacation",
  "description": "Guide through vacation planning process",
  "arguments": [
    { "name": "destination", "type": "string", "required": true },
    { "name": "duration", "type": "number" },
    { "name": "budget", "type": "number" },
    { "name": "interests", "type": "array", "items": { "type": "string" } }
  ]
}
```

### Protocol Methods

| Method | Description |
| --- | --- |
| `prompts/list` | View the list of available prompts |
| `prompts/get` | Get details of a specific prompt |

Users can execute a prompt via a slash command like `/plan-vacation` or a UI button. The prompt appears as a structured input form (text, numbers, dropdowns, etc.), and the entered values are passed directly to the model. This allows for consistent results, autocompletion, and workflow automation.

---

### (2) MCP Client

The **MCP Client** is a component created by host applications like Claude or AI IDEs to communicate with the MCP Server. In other words, it is not an object that the user directly sees or handles, but every server connection must pass through the client.

Whether reading data from the server, executing a tool, or requesting additional information from the user, all requests must go through the client.

### Why the MCP Client is Important

The most significant role of the MCP client is to prevent the server from directly accessing the LLM or the user's environment. All data access and tool execution are designed to go through an approval process in front of the user, allowing them to clearly see and control what tasks the AI is attempting.

For potentially risky functions like file system access, the client establishes **roots** to set boundaries, restricting access to "only up to here," ensuring safety. Additionally, when a server needs to call a model, it doesn't execute it directly; it delegates the task through the client, putting cost, security, and approval procedures all under user control.

### Key Features

| Feature Name | Description |
| --- | --- |
| **Sampling** | The server delegates AI model calls to the client |
| **Roots** | Defines the file system boundaries that the server can access |
| **Elicitation** | Supports the server's ability to request additional input from the user |

Let's look at each of the three key features.

### (2-1) Sampling: Delegating Model Calls

**Sampling** is a feature that allows the server to delegate AI model calls to the client, rather than directly calling the LLM API.

### Example Flow

![https://modelcontextprotocol.io/docs/learn/client-concepts](MCP_Part2_en/image.png)
https://modelcontextprotocol.io/docs/learn/client-concepts

1. The server sends a request to the client, such as "Please analyze this data."
2. The client clearly shows the request to the user and asks for their approval.
3. If the user approves, the client calls the LLM and sends the response back to the server.

All responses are passed only after user review.

### Sample Request Example

```
{
  "messages": [
    {
      "role": "user",
      "content": "Analyze the following flight options and recommend the best choice..."
    }
  ],
  "modelPreferences": {
    "hints": [{ "name": "claude-3-5-sonnet" }],
    "costPriority": 0.3,
    "speedPriority": 0.2,
    "intelligencePriority": 0.9
  },
  "systemPrompt": "As a travel expert, please recommend flights",
  "maxTokens": 1500
}
```

### (2-2) Roots: Setting File System Boundaries

**Roots** is a feature that defines the scope (directories) of the file system that the server can access. All access is configured with `file://` URIs, and the client explicitly communicates these boundaries to the server.

### Sample Request Example

```
{
  "uri": "file:///Users/agent/travel-planning",
  "name": "Travel Planning Workspace"
}
```

The server can only access files within this root; external paths are blocked according to client policy. When a user opens a folder, the client can automatically update the roots and notify the server with a `roots/list_changed` event.

### (2-3) Elicitation: Requesting Additional User Input

**Elicitation** is a feature that allows the server to dynamically request information from the user during a task.

### Usage Flow

![https://modelcontextprotocol.io/docs/learn/client-concepts](MCP_Part2_en/image1.png)

https://modelcontextprotocol.io/docs/learn/client-concepts

1. The server creates a request for user input.
2. The client provides a message and input form via the UI.
3. Once the user completes the input, the information is sent to the server, and the task continues.

### Sample Request Example

```
{
  "method": "elicitation/requestInput",
  "params": {
    "message": "Please confirm the booking for your Barcelona trip:",
    "schema": {
      "type": "object",
      "properties": {
        "confirmBooking": { "type": "boolean" },
        "seatPreference": {
          "type": "string",
          "enum": ["window", "aisle", "no preference"]
        },
        "roomType": {
          "type": "string",
          "enum": ["sea view", "city view"]
        }
      },
      "required": ["confirmBooking"]
    }
  }
}
```

---

### (3) Overall Communication Structure

![본인제작](MCP_Part2_en/image2.png)

The **User** interacts with the AI model through a host application (e.g., Claude, a GPT-based IDE, etc.).
The **AI Model (Host)** interprets the user's request and decides what tasks are needed.
The **MCP Client (Client)** acts as an intermediary between the Host and the MCP Server.
All communication is done in JSON-RPC format, and the client manages the connection with the server while handling user approval, security control, and model call delegation (sampling).
- The **MCP Server (Server)** provides the actual functionality. The server exposes three core building blocks:
    - **Tool**: Specific actions that the AI can execute (e.g., search for flights, send an email, add a calendar event).
    - **Resource**: Context data provided to the AI (e.g., files, calendars, weather information).
    - **Prompt**: A structured interaction template that the user can reuse (e.g., "Plan a vacation for me").

Through this structure, MCP allows AI models to access external data and tools safely and consistently, and it provides a trust-based environment where the user can directly approve and control all actions.

---

This research article has covered MCP in more detail!

I think it would be much clearer if I could actually use it!

Here is a tutorial to easily use MCP, which would be great to check out together~

### Ref

https://modelcontextprotocol.io/docs/getting-started/intro