# Mistral Vibe Web API Specification

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Authentication](#authentication)
4. [Core API Endpoints](#core-api-endpoints)
5. [WebSocket API](#websocket-api)
6. [Data Models](#data-models)
7. [Tool Execution Workflow](#tool-execution-workflow)
8. [Error Handling](#error-handling)
9. [Rate Limiting & Security](#rate-limiting--security)

---

## Overview

This specification defines a RESTful HTTP API with WebSocket support to expose all Mistral Vibe CLI capabilities through a web interface. The API enables:

- Session management (create, resume, list, delete)
- Interactive conversation with AI agent
- Tool execution with approval workflow
- Real-time streaming responses
- Configuration management
- File operations, code search, shell execution
- Task/todo management

### Design Principles
- **Stateful Sessions**: Each conversation maintains state across multiple requests
- **Async-First**: Support for both streaming (WebSocket) and polling (HTTP)
- **Tool Approval**: Client controls whether tools execute automatically or require approval
- **Multi-Tenancy**: Support multiple users with isolated sessions
- **Compatible with ACP**: Align with existing Agent Client Protocol where possible

---

## Architecture

### High-Level Components

```
┌─────────────────┐
│   Web Client    │
│  (Browser/App)  │
└────────┬────────┘
         │
         │ HTTPS/WSS
         │
┌────────▼────────┐
│   API Gateway   │
│  (Auth, Routing)│
└────────┬────────┘
         │
┌────────▼────────────────────────┐
│   Mistral Vibe Web Server       │
│                                  │
│  ┌──────────────────────────┐  │
│  │  Session Manager         │  │
│  │  - Create/Resume/Delete  │  │
│  │  - Session State Store   │  │
│  └──────────────────────────┘  │
│                                  │
│  ┌──────────────────────────┐  │
│  │  Agent Orchestrator      │  │
│  │  - Message Processing    │  │
│  │  - Tool Execution        │  │
│  │  - Approval Queue        │  │
│  └──────────────────────────┘  │
│                                  │
│  ┌──────────────────────────┐  │
│  │  Core Vibe Components    │  │
│  │  - Agent                 │  │
│  │  - Tool Manager          │  │
│  │  - LLM Backend           │  │
│  │  - Config Manager        │  │
│  └──────────────────────────┘  │
└─────────────────────────────────┘
         │
         │
┌────────▼────────┐
│  Mistral AI API │
└─────────────────┘
```

### Technology Stack Recommendations
- **Web Framework**: FastAPI (async, OpenAPI support, WebSocket)
- **Session Store**: Redis (fast, supports TTL, pub/sub for multi-instance)
- **Authentication**: JWT tokens
- **WebSocket**: For streaming responses and real-time updates
- **Storage**: File system or S3 for session logs and file operations

---

## Authentication

### JWT-Based Authentication

All API requests require a valid JWT token in the `Authorization` header:

```
Authorization: Bearer <jwt_token>
```

### Endpoints

#### `POST /api/v1/auth/login`
Authenticate user and obtain JWT token.

**Request:**
```json
{
  "username": "user@example.com",
  "password": "secure_password",
  "api_key": "mistral_api_key_optional"
}
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "token_type": "bearer",
  "expires_in": 3600,
  "user_id": "uuid-user-123"
}
```

#### `POST /api/v1/auth/refresh`
Refresh an expired JWT token.

**Request:**
```json
{
  "refresh_token": "refresh_token_here"
}
```

**Response:**
```json
{
  "access_token": "new_jwt_token",
  "expires_in": 3600
}
```

#### `POST /api/v1/auth/logout`
Invalidate current session token.

---

## Core API Endpoints

### Sessions

#### `POST /api/v1/sessions`
Create a new agent session.

**Request:**
```json
{
  "config": {
    "active_model": "devstral-2",
    "auto_approve": false,
    "api_timeout": 600,
    "project_context": {
      "working_directory": "/path/to/project",
      "max_chars": 40000,
      "max_files": 1000
    },
    "tools": {
      "bash": {
        "permission": "ask",
        "timeout": 120
      },
      "grep": {
        "permission": "always"
      },
      "read_file": {
        "permission": "always"
      },
      "write_file": {
        "permission": "ask"
      },
      "search_replace": {
        "permission": "ask"
      }
    }
  },
  "initial_prompt": "Hello, help me understand this codebase",
  "resume_session_id": null
}
```

**Response:**
```json
{
  "session_id": "session_abc123",
  "created_at": "2025-12-15T10:30:00Z",
  "status": "active",
  "config": { /* echoed config */ }
}
```

#### `GET /api/v1/sessions`
List all sessions for the authenticated user.

**Query Parameters:**
- `status`: Filter by status (`active`, `completed`, `error`)
- `limit`: Max results (default: 50)
- `offset`: Pagination offset

**Response:**
```json
{
  "sessions": [
    {
      "session_id": "session_abc123",
      "created_at": "2025-12-15T10:30:00Z",
      "updated_at": "2025-12-15T11:45:00Z",
      "status": "active",
      "message_count": 15,
      "model": "devstral-2"
    }
  ],
  "total": 42,
  "limit": 50,
  "offset": 0
}
```

#### `GET /api/v1/sessions/{session_id}`
Get details of a specific session.

**Response:**
```json
{
  "session_id": "session_abc123",
  "created_at": "2025-12-15T10:30:00Z",
  "updated_at": "2025-12-15T11:45:00Z",
  "status": "active",
  "config": { /* session config */ },
  "statistics": {
    "total_turns": 8,
    "total_input_tokens": 12500,
    "total_output_tokens": 8300,
    "estimated_cost": 0.025,
    "tool_calls": 12,
    "approved_tools": 10,
    "rejected_tools": 2
  },
  "messages": [
    {
      "role": "user",
      "content": "Hello, help me understand this codebase",
      "timestamp": "2025-12-15T10:30:00Z"
    },
    {
      "role": "assistant",
      "content": "I'd be happy to help you understand the codebase...",
      "timestamp": "2025-12-15T10:30:15Z",
      "tool_calls": []
    }
  ]
}
```

#### `DELETE /api/v1/sessions/{session_id}`
Delete a session and its associated data.

**Response:**
```json
{
  "success": true,
  "message": "Session session_abc123 deleted"
}
```

#### `PATCH /api/v1/sessions/{session_id}/config`
Update session configuration (e.g., change model, toggle auto-approve).

**Request:**
```json
{
  "active_model": "devstral-2501",
  "auto_approve": true
}
```

**Response:**
```json
{
  "session_id": "session_abc123",
  "config": { /* updated config */ }
}
```

### Messages

#### `POST /api/v1/sessions/{session_id}/messages`
Send a message to the agent (synchronous, non-streaming).

**Request:**
```json
{
  "content": "Create a new file called hello.py with a hello world function",
  "max_turns": 10,
  "max_price": 0.50
}
```

**Response:**
```json
{
  "message_id": "msg_xyz789",
  "status": "completed",
  "turns": [
    {
      "turn_number": 1,
      "assistant_message": {
        "role": "assistant",
        "content": "I'll create a hello.py file with a hello world function.",
        "tool_calls": [
          {
            "id": "call_001",
            "type": "function",
            "function": {
              "name": "write_file",
              "arguments": {
                "path": "hello.py",
                "content": "def hello_world():\n    print('Hello, World!')\n"
              }
            },
            "status": "pending_approval"
          }
        ]
      },
      "tool_results": []
    }
  ],
  "pending_approvals": [
    {
      "approval_id": "approval_001",
      "tool_call_id": "call_001",
      "tool_name": "write_file",
      "arguments": {
        "path": "hello.py",
        "content": "def hello_world():\n    print('Hello, World!')\n"
      },
      "expires_at": "2025-12-15T10:40:00Z"
    }
  ]
}
```

#### `GET /api/v1/sessions/{session_id}/messages`
Retrieve message history for a session.

**Query Parameters:**
- `limit`: Max messages (default: 50)
- `offset`: Pagination offset
- `role`: Filter by role (`user`, `assistant`, `system`)

**Response:**
```json
{
  "messages": [
    {
      "message_id": "msg_001",
      "role": "user",
      "content": "Create a new file...",
      "timestamp": "2025-12-15T10:30:00Z"
    },
    {
      "message_id": "msg_002",
      "role": "assistant",
      "content": "I'll create a hello.py file...",
      "tool_calls": [...],
      "timestamp": "2025-12-15T10:30:05Z"
    }
  ],
  "total": 25,
  "limit": 50,
  "offset": 0
}
```

### Tool Approvals

#### `POST /api/v1/sessions/{session_id}/approvals/{approval_id}`
Approve or reject a pending tool execution.

**Request:**
```json
{
  "action": "approve",  // or "reject"
  "reason": "Optional rejection reason"
}
```

**Response:**
```json
{
  "approval_id": "approval_001",
  "status": "approved",
  "tool_result": {
    "success": true,
    "output": "File hello.py created successfully"
  }
}
```

#### `GET /api/v1/sessions/{session_id}/approvals`
List pending tool approvals for a session.

**Response:**
```json
{
  "approvals": [
    {
      "approval_id": "approval_001",
      "tool_call_id": "call_001",
      "tool_name": "write_file",
      "arguments": {...},
      "created_at": "2025-12-15T10:30:05Z",
      "expires_at": "2025-12-15T10:40:05Z"
    }
  ]
}
```

### Tools

#### `GET /api/v1/tools`
List all available tools and their configurations.

**Response:**
```json
{
  "tools": [
    {
      "name": "bash",
      "description": "Execute shell commands",
      "permission": "ask",
      "parameters": {
        "type": "object",
        "properties": {
          "command": {
            "type": "string",
            "description": "The shell command to execute"
          }
        },
        "required": ["command"]
      }
    },
    {
      "name": "read_file",
      "description": "Read file contents",
      "permission": "always",
      "parameters": {
        "type": "object",
        "properties": {
          "path": {
            "type": "string",
            "description": "Path to the file"
          },
          "offset": {
            "type": "integer",
            "description": "Line offset to start reading from"
          },
          "limit": {
            "type": "integer",
            "description": "Maximum number of lines to read"
          }
        },
        "required": ["path"]
      }
    }
  ]
}
```

#### `POST /api/v1/sessions/{session_id}/tools/execute`
Manually execute a tool (for advanced use cases).

**Request:**
```json
{
  "tool_name": "grep",
  "arguments": {
    "pattern": "def.*:",
    "path": "./",
    "file_pattern": "*.py"
  }
}
```

**Response:**
```json
{
  "tool_execution_id": "exec_123",
  "status": "completed",
  "result": {
    "matches": [
      {
        "file": "hello.py",
        "line": 1,
        "content": "def hello_world():"
      }
    ]
  }
}
```

### Todo Management

#### `GET /api/v1/sessions/{session_id}/todos`
Get the current todo list for a session.

**Response:**
```json
{
  "todos": [
    {
      "id": "todo_001",
      "content": "Create hello.py file",
      "status": "completed",
      "priority": "high",
      "created_at": "2025-12-15T10:30:00Z",
      "completed_at": "2025-12-15T10:31:00Z"
    },
    {
      "id": "todo_002",
      "content": "Add unit tests",
      "status": "in_progress",
      "priority": "medium",
      "created_at": "2025-12-15T10:31:00Z"
    }
  ]
}
```

#### `POST /api/v1/sessions/{session_id}/todos`
Create a new todo item.

**Request:**
```json
{
  "content": "Refactor authentication module",
  "priority": "high"
}
```

#### `PATCH /api/v1/sessions/{session_id}/todos/{todo_id}`
Update todo status or priority.

**Request:**
```json
{
  "status": "completed"
}
```

### Configuration

#### `GET /api/v1/config`
Get global configuration and available models.

**Response:**
```json
{
  "models": [
    {
      "alias": "devstral-2",
      "name": "devstral-2501",
      "input_price": 0.05,
      "output_price": 0.15,
      "context_length": 32768
    }
  ],
  "providers": [
    {
      "name": "mistral",
      "api_base": "https://api.mistral.ai/v1",
      "backend": "mistral"
    }
  ],
  "default_config": {
    "active_model": "devstral-2",
    "auto_approve": false,
    "api_timeout": 600
  }
}
```

### Health & Status

#### `GET /api/v1/health`
Health check endpoint.

**Response:**
```json
{
  "status": "healthy",
  "version": "1.1.3",
  "uptime_seconds": 86400
}
```

#### `GET /api/v1/stats`
Server statistics and metrics.

**Response:**
```json
{
  "active_sessions": 42,
  "total_sessions": 1337,
  "total_messages": 50000,
  "total_tool_calls": 25000,
  "uptime_seconds": 86400
}
```

---

## WebSocket API

### Connection

```
WSS /api/v1/sessions/{session_id}/stream
```

**Authentication:** JWT token via query parameter or Sec-WebSocket-Protocol header
```
wss://api.example.com/api/v1/sessions/session_abc123/stream?token=<jwt>
```

### Message Format

All WebSocket messages use JSON with a `type` field to distinguish message types.

#### Client → Server Messages

##### Send User Message
```json
{
  "type": "user_message",
  "content": "Create a new Python file",
  "max_turns": 10,
  "max_price": 0.50
}
```

##### Approve/Reject Tool
```json
{
  "type": "tool_approval",
  "approval_id": "approval_001",
  "action": "approve"  // or "reject"
}
```

##### Update Config
```json
{
  "type": "update_config",
  "config": {
    "auto_approve": true
  }
}
```

##### Cancel Operation
```json
{
  "type": "cancel",
  "reason": "User requested cancellation"
}
```

##### Ping
```json
{
  "type": "ping"
}
```

#### Server → Client Messages

##### Connection Established
```json
{
  "type": "connected",
  "session_id": "session_abc123",
  "timestamp": "2025-12-15T10:30:00Z"
}
```

##### Streaming Assistant Response
```json
{
  "type": "assistant_message_delta",
  "delta": {
    "content": "I'll help you create ",
    "role": "assistant"
  },
  "message_id": "msg_123"
}
```

##### Complete Assistant Message
```json
{
  "type": "assistant_message_complete",
  "message_id": "msg_123",
  "message": {
    "role": "assistant",
    "content": "I'll help you create a new Python file.",
    "tool_calls": [...]
  }
}
```

##### Tool Call Pending Approval
```json
{
  "type": "tool_approval_required",
  "approval_id": "approval_001",
  "tool_call_id": "call_001",
  "tool_name": "write_file",
  "arguments": {
    "path": "example.py",
    "content": "print('Hello')"
  },
  "expires_at": "2025-12-15T10:40:00Z"
}
```

##### Tool Execution Started
```json
{
  "type": "tool_execution_started",
  "tool_call_id": "call_001",
  "tool_name": "write_file"
}
```

##### Tool Execution Result
```json
{
  "type": "tool_execution_result",
  "tool_call_id": "call_001",
  "result": {
    "success": true,
    "output": "File created successfully"
  }
}
```

##### Todo Update
```json
{
  "type": "todo_update",
  "todos": [
    {
      "id": "todo_001",
      "content": "Create Python file",
      "status": "completed"
    }
  ]
}
```

##### Statistics Update
```json
{
  "type": "statistics_update",
  "statistics": {
    "total_turns": 5,
    "total_input_tokens": 1200,
    "total_output_tokens": 800,
    "estimated_cost": 0.015
  }
}
```

##### Error
```json
{
  "type": "error",
  "error": {
    "code": "TOOL_EXECUTION_FAILED",
    "message": "Failed to write file: Permission denied",
    "details": {
      "tool_name": "write_file",
      "path": "/etc/protected.txt"
    }
  }
}
```

##### Pong
```json
{
  "type": "pong",
  "timestamp": "2025-12-15T10:30:00Z"
}
```

---

## Data Models

### Session
```typescript
interface Session {
  session_id: string;
  user_id: string;
  created_at: string;  // ISO 8601
  updated_at: string;
  status: "active" | "completed" | "error";
  config: SessionConfig;
  statistics: SessionStatistics;
}
```

### SessionConfig
```typescript
interface SessionConfig {
  active_model: string;
  auto_approve: boolean;
  api_timeout: number;
  auto_compact_threshold: number;
  context_warnings: boolean;
  project_context?: {
    working_directory: string;
    max_chars: number;
    max_files: number;
  };
  tools: {
    [tool_name: string]: ToolConfig;
  };
}
```

### ToolConfig
```typescript
interface ToolConfig {
  permission: "always" | "ask" | "never";
  timeout?: number;
  // Tool-specific config
  [key: string]: any;
}
```

### Message
```typescript
interface Message {
  message_id: string;
  role: "user" | "assistant" | "system";
  content: string;
  timestamp: string;
  tool_calls?: ToolCall[];
}
```

### ToolCall
```typescript
interface ToolCall {
  id: string;
  type: "function";
  function: {
    name: string;
    arguments: Record<string, any>;
  };
  status: "pending_approval" | "approved" | "rejected" | "executing" | "completed" | "failed";
  result?: ToolResult;
}
```

### ToolResult
```typescript
interface ToolResult {
  success: boolean;
  output?: string;
  error?: string;
  data?: any;
}
```

### ToolApproval
```typescript
interface ToolApproval {
  approval_id: string;
  session_id: string;
  tool_call_id: string;
  tool_name: string;
  arguments: Record<string, any>;
  created_at: string;
  expires_at: string;
  status: "pending" | "approved" | "rejected" | "expired";
}
```

### Todo
```typescript
interface Todo {
  id: string;
  content: string;
  status: "pending" | "in_progress" | "completed" | "cancelled";
  priority: "low" | "medium" | "high";
  created_at: string;
  updated_at?: string;
  completed_at?: string;
}
```

### SessionStatistics
```typescript
interface SessionStatistics {
  total_turns: number;
  total_input_tokens: number;
  total_output_tokens: number;
  estimated_cost: number;
  tool_calls: number;
  approved_tools: number;
  rejected_tools: number;
}
```

---

## Tool Execution Workflow

### Standard Flow (Tools Requiring Approval)

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ 1. Send message
     ▼
┌─────────────────┐
│  Agent Process  │
│  Message        │
└────┬────────────┘
     │
     │ 2. LLM responds with tool calls
     ▼
┌─────────────────┐
│  Check Tool     │
│  Permission     │
└────┬────────────┘
     │
     │ permission="ask"
     ▼
┌─────────────────┐
│  Create         │
│  Approval       │
│  Request        │
└────┬────────────┘
     │
     │ 3. Send approval request to client
     ▼
┌──────────┐
│  Client  │
│  Reviews │
└────┬─────┘
     │
     │ 4. Approve/Reject
     ▼
┌─────────────────┐
│  Execute Tool   │ (if approved)
│  or Skip        │ (if rejected)
└────┬────────────┘
     │
     │ 5. Return result to LLM
     ▼
┌─────────────────┐
│  LLM Continues  │
│  Conversation   │
└────┬────────────┘
     │
     │ 6. Send final response
     ▼
┌──────────┐
│  Client  │
└──────────┘
```

### Auto-Approve Flow

When `auto_approve=true` or tool has `permission="always"`:

1. Client sends message
2. Agent processes message
3. LLM responds with tool calls
4. Tools execute immediately (no approval needed)
5. Results returned to LLM
6. LLM continues conversation
7. Final response sent to client

### Batch Tool Execution

When LLM requests multiple tools in a single turn:

1. All tools requiring approval are batched into a single approval request
2. Client can approve/reject each tool individually
3. Approved tools execute in parallel (when possible)
4. Results are collected and sent back to LLM together

---

## Error Handling

### HTTP Error Codes

| Code | Meaning | Usage |
|------|---------|-------|
| 400 | Bad Request | Invalid request format or parameters |
| 401 | Unauthorized | Missing or invalid authentication token |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Session, message, or resource not found |
| 409 | Conflict | Session state conflict (e.g., already processing) |
| 422 | Unprocessable Entity | Valid format but semantic errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server-side error |
| 503 | Service Unavailable | Mistral API unavailable or server overloaded |

### Error Response Format

```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session session_abc123 does not exist",
    "details": {
      "session_id": "session_abc123",
      "user_id": "user_123"
    },
    "timestamp": "2025-12-15T10:30:00Z",
    "request_id": "req_xyz789"
  }
}
```

### Common Error Codes

#### Session Errors
- `SESSION_NOT_FOUND`: Session doesn't exist
- `SESSION_EXPIRED`: Session has expired
- `SESSION_LOCKED`: Session is being processed by another request
- `SESSION_LIMIT_EXCEEDED`: User has too many active sessions

#### Tool Errors
- `TOOL_NOT_FOUND`: Tool doesn't exist
- `TOOL_PERMISSION_DENIED`: Tool execution not permitted
- `TOOL_EXECUTION_FAILED`: Tool execution failed
- `TOOL_TIMEOUT`: Tool execution exceeded timeout
- `APPROVAL_REQUIRED`: Tool requires approval
- `APPROVAL_EXPIRED`: Approval request expired
- `APPROVAL_NOT_FOUND`: Approval ID doesn't exist

#### Configuration Errors
- `INVALID_MODEL`: Model doesn't exist or not available
- `INVALID_CONFIG`: Configuration validation failed
- `API_KEY_INVALID`: Mistral API key is invalid
- `API_KEY_MISSING`: Mistral API key not configured

#### Resource Errors
- `FILE_NOT_FOUND`: File doesn't exist
- `FILE_TOO_LARGE`: File exceeds size limit
- `PERMISSION_DENIED`: Insufficient file system permissions
- `DISK_FULL`: Insufficient disk space

#### Rate Limiting
- `RATE_LIMIT_EXCEEDED`: Too many requests
- `CONCURRENT_LIMIT_EXCEEDED`: Too many concurrent requests
- `COST_LIMIT_EXCEEDED`: Session exceeded cost limit
- `TURN_LIMIT_EXCEEDED`: Session exceeded turn limit

---

## Rate Limiting & Security

### Rate Limits

**Per User:**
- 100 requests per minute
- 10 concurrent sessions
- 1000 tool executions per hour

**Per Session:**
- 60 messages per hour
- 500 total messages per session
- Configurable `max_turns` and `max_price` per message

**Response Headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1702645800
```

### Security Considerations

#### File System Access
- **Sandboxing**: Restrict file operations to user's project directory
- **Path Validation**: Prevent directory traversal attacks (`../`, absolute paths)
- **File Size Limits**: Enforce max file sizes (e.g., 10MB for read/write)
- **Allowed Extensions**: Whitelist or blacklist file types
- **Symlink Protection**: Block operations on symlinks

#### Shell Execution
- **Command Allowlist**: Optionally restrict to safe commands
- **Timeout Enforcement**: Kill long-running processes
- **Resource Limits**: CPU, memory, disk I/O limits
- **Dangerous Command Detection**: Block `rm -rf /`, `:(){ :|:& };:`, etc.
- **Environment Isolation**: Use containerization (Docker) per session

#### API Key Management
- **User-Provided Keys**: Allow users to provide their own Mistral API keys
- **Server-Side Keys**: Securely store and rotate server API keys
- **Usage Tracking**: Monitor per-key usage and costs
- **Key Isolation**: Prevent key leakage in error messages or logs

#### Session Isolation
- **Multi-Tenancy**: Ensure complete isolation between user sessions
- **Session TTL**: Auto-expire inactive sessions (e.g., 1 hour)
- **Session Cleanup**: Delete session data after expiration
- **Audit Logging**: Log all tool executions and approvals

#### Input Validation
- **Prompt Injection**: Sanitize user inputs to prevent system prompt manipulation
- **JSON Schema Validation**: Validate all request bodies
- **Path Sanitization**: Clean file paths before operations
- **Length Limits**: Enforce max lengths on all text inputs

#### WebSocket Security
- **Authentication**: Require JWT on connection
- **Origin Checking**: Validate WebSocket origin
- **Message Size Limits**: Prevent large message attacks
- **Connection Limits**: Max connections per user
- **Heartbeat**: Detect and close stale connections

---

## Implementation Recommendations

### Backend Architecture

**Option 1: FastAPI + Redis + Docker**
```python
# app/main.py
from fastapi import FastAPI, WebSocket, Depends
from fastapi.security import HTTPBearer
from app.sessions import SessionManager
from app.auth import verify_token

app = FastAPI(title="Mistral Vibe API", version="1.0.0")
security = HTTPBearer()
session_manager = SessionManager()

@app.post("/api/v1/sessions")
async def create_session(token = Depends(verify_token)):
    # Implementation
    pass

@app.websocket("/api/v1/sessions/{session_id}/stream")
async def websocket_endpoint(websocket: WebSocket, session_id: str):
    # Implementation
    pass
```

**Option 2: Extend Existing ACP Server**
- Build REST API wrapper around `vibe.acp.acp_agent.ACPAgent`
- Reuse existing agent infrastructure
- Add HTTP/WebSocket transport alongside stdio

### Session Storage

**Redis Schema:**
```
# Session metadata
session:{session_id} → JSON(Session)

# User sessions index
user:{user_id}:sessions → Set[session_id]

# Session messages
session:{session_id}:messages → List[Message]

# Pending approvals
session:{session_id}:approvals → Hash[approval_id → Approval]

# Todo list
session:{session_id}:todos → List[Todo]

# Session lock (for concurrency control)
session:{session_id}:lock → String (request_id)
```

### File Operations

**Workspace Isolation:**
```
/workspaces/
  ├── user_123/
  │   ├── session_abc/
  │   │   ├── project_files/
  │   │   └── .vibe/
  │   └── session_def/
  └── user_456/
```

**Docker Container Per Session:**
```yaml
# docker-compose.yml
services:
  vibe-session:
    image: mistral-vibe-sandbox:latest
    volumes:
      - ./workspaces/user_123/session_abc:/workspace
    environment:
      - MISTRAL_API_KEY=${USER_API_KEY}
    mem_limit: 512m
    cpus: 0.5
```

### Deployment Considerations

1. **Horizontal Scaling**:
   - Stateless API servers
   - Redis for shared state
   - Load balancer for WebSocket connections (sticky sessions)

2. **Monitoring**:
   - Prometheus metrics (request latency, error rates, tool usage)
   - Sentry for error tracking
   - Session analytics (popular tools, cost tracking)

3. **Logging**:
   - Structured JSON logs
   - Session audit trail
   - Tool execution logs

4. **Cost Management**:
   - Track Mistral API usage per user
   - Enforce budget limits
   - Cache common LLM responses

---

## API Versioning

- Version in URL path: `/api/v1/...`, `/api/v2/...`
- Maintain backwards compatibility for at least 6 months
- Deprecation headers: `X-API-Deprecation: version=v1, sunset=2026-06-01`

---

## OpenAPI Specification

The complete OpenAPI 3.0 specification should be generated from code and available at:
- `/api/v1/openapi.json`
- `/api/v1/docs` (Swagger UI)
- `/api/v1/redoc` (ReDoc)

---

## Example Usage Flows

### Example 1: Simple Question & Answer (HTTP)

```bash
# 1. Login
curl -X POST https://api.example.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "user@example.com", "password": "pass123"}'
# Response: {"access_token": "eyJ...", ...}

# 2. Create session
curl -X POST https://api.example.com/api/v1/sessions \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"config": {"auto_approve": true}}'
# Response: {"session_id": "session_123", ...}

# 3. Send message
curl -X POST https://api.example.com/api/v1/sessions/session_123/messages \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{"content": "What files are in the current directory?"}'
# Response: {"message_id": "msg_456", "turns": [...], ...}
```

### Example 2: File Creation with Approval (WebSocket)

```javascript
// JavaScript client
const ws = new WebSocket('wss://api.example.com/api/v1/sessions/session_123/stream?token=eyJ...');

ws.onopen = () => {
  // Send request to create file
  ws.send(JSON.stringify({
    type: 'user_message',
    content: 'Create a file named test.txt with "Hello World"'
  }));
};

ws.onmessage = (event) => {
  const msg = JSON.parse(event.data);

  switch (msg.type) {
    case 'assistant_message_delta':
      // Stream assistant response
      console.log(msg.delta.content);
      break;

    case 'tool_approval_required':
      // Tool needs approval
      console.log(`Approve ${msg.tool_name}?`, msg.arguments);

      // User clicks "Approve"
      ws.send(JSON.stringify({
        type: 'tool_approval',
        approval_id: msg.approval_id,
        action: 'approve'
      }));
      break;

    case 'tool_execution_result':
      console.log('Tool result:', msg.result);
      break;
  }
};
```

### Example 3: Code Search and Refactoring (Mixed)

```python
import requests
import json

BASE_URL = "https://api.example.com/api/v1"
TOKEN = "eyJ..."
headers = {"Authorization": f"Bearer {TOKEN}"}

# Create session with project context
session = requests.post(f"{BASE_URL}/sessions", headers=headers, json={
    "config": {
        "project_context": {
            "working_directory": "/home/user/my-project"
        },
        "auto_approve": False  # Require approval for writes
    }
}).json()

session_id = session["session_id"]

# Search for TODO comments
response = requests.post(
    f"{BASE_URL}/sessions/{session_id}/tools/execute",
    headers=headers,
    json={
        "tool_name": "grep",
        "arguments": {
            "pattern": "TODO:",
            "path": "./",
            "file_pattern": "*.py"
        }
    }
).json()

print(f"Found {len(response['result']['matches'])} TODOs")

# Ask agent to refactor
msg_response = requests.post(
    f"{BASE_URL}/sessions/{session_id}/messages",
    headers=headers,
    json={
        "content": "Refactor the authentication module to use async/await"
    }
).json()

# Check for pending approvals
approvals = requests.get(
    f"{BASE_URL}/sessions/{session_id}/approvals",
    headers=headers
).json()

# Approve file changes
for approval in approvals["approvals"]:
    print(f"Approve {approval['tool_name']} on {approval['arguments'].get('path')}? (y/n)")
    if input().lower() == 'y':
        requests.post(
            f"{BASE_URL}/sessions/{session_id}/approvals/{approval['approval_id']}",
            headers=headers,
            json={"action": "approve"}
        )
```

---

## Next Steps

To implement this API specification:

1. **Choose Architecture**: FastAPI + Redis recommended for new implementation
2. **Session Management**: Implement SessionManager with Redis backend
3. **Agent Integration**: Wrap existing `vibe.core.agent.Agent` or extend ACP
4. **Authentication**: Set up JWT-based auth with user management
5. **WebSocket Handler**: Implement bidirectional streaming
6. **Tool Execution**: Add approval queue and execution sandbox
7. **File Operations**: Implement workspace isolation (Docker recommended)
8. **Testing**: Create integration tests for all endpoints
9. **Documentation**: Generate OpenAPI spec and interactive docs
10. **Security Audit**: Review sandboxing, input validation, rate limiting
11. **Deployment**: Containerize, set up CI/CD, monitoring

---

## Appendix: Comparison with ACP

The Agent Client Protocol (ACP) already provides some API functionality via stdio JSON-RPC. Key differences:

| Feature | ACP (stdio) | Web API (This Spec) |
|---------|-------------|---------------------|
| Transport | stdio | HTTP/WebSocket |
| Authentication | None | JWT-based |
| Multi-user | No | Yes |
| Session persistence | Client-managed | Server-managed |
| RESTful | No | Yes |
| Browser-compatible | No | Yes |
| Approval workflow | Yes | Yes (enhanced) |
| Streaming | Yes (stdio) | Yes (WebSocket) |

The Web API can be built on top of ACP concepts while adding web-specific requirements (auth, multi-tenancy, persistence, RESTful design).
