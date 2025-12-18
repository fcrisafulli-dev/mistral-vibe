# Mistral Vibe Web UI: Solo Developer Guide

**Version:** 1.0
**Target:** Single full-stack developer
**Timeline:** 6-8 weeks (MVP)
**Complexity:** Intermediate

---

## Executive Summary

This guide simplifies the web UI implementation for a **single developer** working part-time or full-time. We cut out enterprise complexity (Kubernetes, microservices, etc.) and focus on a **working prototype** that can evolve over time.

**What You'll Build:**
- Chat interface to interact with Mistral Vibe
- On-demand Docker containers for isolated sessions
- Simple file browser and code viewer
- Terminal output display

**Tech Stack (Simplified):**
- **Frontend:** React + Vite (no Next.js complexity)
- **Backend:** Single Node.js server (handles WebSocket + Docker)
- **Database:** SQLite (file-based, no PostgreSQL setup)
- **Deployment:** Docker Compose (single command deploy)
- **Auth:** Simple token-based (no OAuth initially)

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Architecture Overview](#architecture-overview)
3. [Week-by-Week Plan](#week-by-week-plan)
4. [Implementation Guide](#implementation-guide)
5. [Deployment](#deployment)
6. [Next Steps](#next-steps)

---

## 1. Prerequisites

### Skills Needed
- ✅ Comfortable with JavaScript/TypeScript
- ✅ Basic React knowledge
- ✅ Familiar with Node.js and Express
- ✅ Basic Docker understanding
- ✅ Can read Python code (to understand vibe-acp)

### Tools to Install
```bash
# Node.js 20+
node --version  # Should be v20+

# Docker Desktop
docker --version

# pnpm (faster than npm)
npm install -g pnpm

# Optional: VS Code with extensions
code --install-extension dbaeumer.vscode-eslint
```

### API Keys
- Mistral API key (get from https://console.mistral.ai/)

---

## 2. Architecture Overview

### Simplified Architecture

```
┌─────────────────────────────────────────────┐
│  Browser                                     │
│  ┌────────────────────────────────────────┐ │
│  │  React App (Vite)                      │ │
│  │  - Chat UI                             │ │
│  │  - Code viewer (Monaco)                │ │
│  │  - Terminal (xterm.js)                 │ │
│  └─────────────────┬──────────────────────┘ │
└────────────────────┼────────────────────────┘
                     │ WebSocket
┌────────────────────┼────────────────────────┐
│  Node.js Server (single process)            │
│  ┌──────────────────────────────────────┐  │
│  │  Express Server                       │  │
│  │  - WebSocket handler                  │  │
│  │  - stdio ↔ WebSocket bridge          │  │
│  │  - Docker container manager           │  │
│  │  - SQLite for sessions                │  │
│  └─────────────────┬────────────────────┘  │
└────────────────────┼────────────────────────┘
                     │ Docker SDK
┌────────────────────┼────────────────────────┐
│  Docker Container (per session)             │
│  ┌──────────────────────────────────────┐  │
│  │  vibe-acp running                     │  │
│  │  User's project files                 │  │
│  └──────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

### What We're NOT Building (Yet)
- ❌ Kubernetes/complex orchestration
- ❌ OAuth/SSO authentication
- ❌ User accounts and billing
- ❌ Multi-region deployment
- ❌ Advanced monitoring/alerting
- ❌ File editing (read-only for now)

### What We ARE Building (MVP)
- ✅ Chat interface with streaming responses
- ✅ Docker container per session
- ✅ Code viewing (read-only Monaco editor)
- ✅ Terminal output display
- ✅ Tool execution with approval
- ✅ Basic session persistence (SQLite)

---

## 3. Week-by-Week Plan

### Week 1-2: Backend Foundation
**Goal:** Get vibe-acp running in Docker and bridge stdio to WebSocket

**Tasks:**
- [ ] Create Dockerfile for vibe-acp container
- [ ] Build Node.js server skeleton
- [ ] Implement Docker container lifecycle (create/destroy)
- [ ] Bridge stdio ↔ WebSocket communication
- [ ] Test ACP protocol end-to-end with curl/wscat

**Deliverable:** Working WebSocket endpoint that talks to vibe-acp

### Week 3-4: Frontend Basics
**Goal:** Build chat interface that connects to backend

**Tasks:**
- [ ] Set up Vite + React + TypeScript project
- [ ] Build chat UI with message list
- [ ] Implement WebSocket client
- [ ] Display streaming agent messages
- [ ] Add input field and send button

**Deliverable:** Chat interface that can send prompts and receive responses

### Week 5-6: Tool Integration
**Goal:** Handle tool calls and display results

**Tasks:**
- [ ] Implement tool approval UI (modal/dialog)
- [ ] Integrate xterm.js for bash output
- [ ] Add Monaco editor for code viewing
- [ ] Handle all ACP session updates
- [ ] Build simple file browser (list files)

**Deliverable:** Full tool execution flow works

### Week 7-8: Polish & Deploy
**Goal:** Make it production-ready and deployable

**Tasks:**
- [ ] Add session persistence (SQLite)
- [ ] Error handling and loading states
- [ ] Basic authentication (API key)
- [ ] Create Docker Compose file
- [ ] Write deployment docs
- [ ] Bug fixes and UX improvements

**Deliverable:** Deployable application with docker-compose

---

## 4. Implementation Guide

### 4.1 Docker Container Setup

**File:** `docker/Dockerfile.vibe-acp`

```dockerfile
FROM python:3.12-slim

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    ripgrep \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Install uv (fast Python package installer)
RUN curl -LsSf https://astral.sh/uv/install.sh | sh
ENV PATH="/root/.local/bin:${PATH}"

# Install Mistral Vibe
RUN uv tool install mistral-vibe

# Set working directory
WORKDIR /workspace

# Create vibe config directory
RUN mkdir -p /root/.vibe

# Entry point
CMD ["vibe-acp"]
```

**Build:**
```bash
docker build -f docker/Dockerfile.vibe-acp -t vibe-acp:latest .
```

### 4.2 Backend Server (Simplified)

**File:** `backend/src/index.ts`

```typescript
import express from 'express';
import { WebSocketServer } from 'ws';
import Docker from 'dockerode';
import { randomUUID } from 'crypto';

const app = express();
const docker = new Docker();
const sessions = new Map<string, Session>();

interface Session {
  id: string;
  containerId: string;
  ws: WebSocket;
  process: any;
}

// WebSocket endpoint
const wss = new WebSocketServer({ port: 4000 });

wss.on('connection', async (ws) => {
  const sessionId = randomUUID();
  console.log(`New session: ${sessionId}`);

  try {
    // Create container
    const container = await docker.createContainer({
      Image: 'vibe-acp:latest',
      Env: ['MISTRAL_API_KEY=your_api_key_here'],
      Tty: false,
      OpenStdin: true,
      StdinOnce: false,
      AttachStdin: true,
      AttachStdout: true,
      AttachStderr: true,
    });

    await container.start();

    // Attach to container streams
    const stream = await container.attach({
      stream: true,
      stdin: true,
      stdout: true,
      stderr: true,
    });

    // Store session
    sessions.set(sessionId, {
      id: sessionId,
      containerId: container.id,
      ws: ws,
      process: { stdin: stream, stdout: stream },
    });

    // Pipe container stdout → WebSocket
    stream.on('data', (chunk) => {
      // Docker multiplexes streams, parse it
      const data = chunk.toString('utf8');
      if (ws.readyState === ws.OPEN) {
        ws.send(data);
      }
    });

    // Pipe WebSocket → container stdin
    ws.on('message', (data) => {
      stream.write(data.toString() + '\n');
    });

    // Cleanup on disconnect
    ws.on('close', async () => {
      console.log(`Session closed: ${sessionId}`);
      try {
        await container.stop({ t: 5 });
        await container.remove();
      } catch (e) {
        console.error('Cleanup error:', e);
      }
      sessions.delete(sessionId);
    });

  } catch (error) {
    console.error('Container error:', error);
    ws.close();
  }
});

app.listen(3001, () => {
  console.log('Server running on http://localhost:3001');
});

console.log('WebSocket server running on ws://localhost:4000');
```

**Dependencies:**
```json
{
  "dependencies": {
    "express": "^4.21.0",
    "ws": "^8.18.0",
    "dockerode": "^4.0.2"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/ws": "^8.5.10",
    "tsx": "^4.7.0",
    "typescript": "^5.5.0"
  }
}
```

**Run:**
```bash
cd backend
pnpm install
pnpm tsx src/index.ts
```

### 4.3 Frontend (Simplified)

**File:** `frontend/src/App.tsx`

```tsx
import { useState, useEffect, useRef } from 'react';
import './App.css';

interface Message {
  id: string;
  type: 'user' | 'agent' | 'tool_call' | 'tool_result';
  content: string;
}

function App() {
  const [messages, setMessages] = useState<Message[]>([]);
  const [input, setInput] = useState('');
  const [ws, setWs] = useState<WebSocket | null>(null);
  const [connected, setConnected] = useState(false);

  // Connect to WebSocket
  useEffect(() => {
    const socket = new WebSocket('ws://localhost:4000');

    socket.onopen = () => {
      console.log('Connected to server');
      setConnected(true);

      // Send initialize request
      socket.send(JSON.stringify({
        jsonrpc: '2.0',
        id: 1,
        method: 'initialize',
        params: {
          clientCapabilities: {
            terminal: true,
            fs: { readTextFile: true, writeTextFile: true }
          }
        }
      }));
    };

    socket.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data);
        console.log('Received:', data);

        // Handle different message types
        if (data.sessionUpdate === 'agent_message_chunk') {
          setMessages(prev => [...prev, {
            id: Date.now().toString(),
            type: 'agent',
            content: data.content.text
          }]);
        }
      } catch (e) {
        console.error('Parse error:', e);
      }
    };

    socket.onclose = () => {
      console.log('Disconnected from server');
      setConnected(false);
    };

    setWs(socket);

    return () => {
      socket.close();
    };
  }, []);

  const sendPrompt = () => {
    if (!ws || !input.trim()) return;

    // Add user message to UI
    setMessages(prev => [...prev, {
      id: Date.now().toString(),
      type: 'user',
      content: input
    }]);

    // Send newSession request (simplified - should track session ID)
    ws.send(JSON.stringify({
      jsonrpc: '2.0',
      id: 2,
      method: 'newSession',
      params: {
        cwd: '/workspace'
      }
    }));

    // Send prompt (after session is created - should wait for response)
    setTimeout(() => {
      ws.send(JSON.stringify({
        jsonrpc: '2.0',
        id: 3,
        method: 'prompt',
        params: {
          sessionId: 'temp-session-id', // Should use real session ID
          prompt: [{ type: 'text', text: input }]
        }
      }));
    }, 1000);

    setInput('');
  };

  return (
    <div className="app">
      <header>
        <h1>Mistral Vibe Web UI</h1>
        <div className="status">
          {connected ? '🟢 Connected' : '🔴 Disconnected'}
        </div>
      </header>

      <div className="chat-container">
        <div className="messages">
          {messages.map(msg => (
            <div key={msg.id} className={`message ${msg.type}`}>
              <div className="message-content">{msg.content}</div>
            </div>
          ))}
        </div>

        <div className="input-area">
          <input
            type="text"
            value={input}
            onChange={(e) => setInput(e.target.value)}
            onKeyPress={(e) => e.key === 'Enter' && sendPrompt()}
            placeholder="Ask me anything about your code..."
            disabled={!connected}
          />
          <button onClick={sendPrompt} disabled={!connected}>
            Send
          </button>
        </div>
      </div>
    </div>
  );
}

export default App;
```

**File:** `frontend/src/App.css`

```css
.app {
  height: 100vh;
  display: flex;
  flex-direction: column;
  background: #1e1e1e;
  color: #d4d4d4;
}

header {
  padding: 1rem;
  background: #2d2d2d;
  border-bottom: 1px solid #3e3e3e;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

h1 {
  margin: 0;
  font-size: 1.5rem;
}

.status {
  font-size: 0.875rem;
}

.chat-container {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.message {
  padding: 0.75rem 1rem;
  border-radius: 8px;
  max-width: 80%;
}

.message.user {
  background: #0e639c;
  margin-left: auto;
}

.message.agent {
  background: #2d2d2d;
}

.input-area {
  padding: 1rem;
  background: #2d2d2d;
  border-top: 1px solid #3e3e3e;
  display: flex;
  gap: 0.5rem;
}

.input-area input {
  flex: 1;
  padding: 0.75rem;
  background: #3c3c3c;
  border: 1px solid #555;
  border-radius: 4px;
  color: #d4d4d4;
  font-size: 1rem;
}

.input-area input:focus {
  outline: none;
  border-color: #0e639c;
}

.input-area button {
  padding: 0.75rem 1.5rem;
  background: #0e639c;
  border: none;
  border-radius: 4px;
  color: white;
  cursor: pointer;
  font-size: 1rem;
}

.input-area button:hover {
  background: #1177bb;
}

.input-area button:disabled {
  background: #555;
  cursor: not-allowed;
}
```

**Setup:**
```bash
# Create Vite project
pnpm create vite frontend -- --template react-ts
cd frontend
pnpm install
pnpm dev
```

### 4.4 Complete Docker Compose Setup

**File:** `docker-compose.yml`

```yaml
version: '3.9'

services:
  backend:
    build: ./backend
    ports:
      - "3001:3001"
      - "4000:4000"
    volumes:
      # Mount Docker socket to create containers
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - MISTRAL_API_KEY=${MISTRAL_API_KEY}
    depends_on:
      - build-vibe-image

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - VITE_WS_URL=ws://localhost:4000

  # Build the vibe-acp image that will be used for sessions
  build-vibe-image:
    build:
      context: ./docker
      dockerfile: Dockerfile.vibe-acp
    image: vibe-acp:latest
    command: ["echo", "Image built successfully"]
```

**Usage:**
```bash
# Set API key
export MISTRAL_API_KEY="your_api_key_here"

# Start everything
docker-compose up

# Access at http://localhost:3000
```

---

## 5. Deployment

### 5.1 Local Development

```bash
# Clone your repo
git clone <repo-url>
cd mistral-vibe-web

# Set up environment
cp .env.example .env
# Edit .env and add your MISTRAL_API_KEY

# Start all services
docker-compose up

# Access UI at http://localhost:3000
```

### 5.2 Deploy to Single VPS (DigitalOcean/Hetzner)

**Requirements:**
- VPS with 8GB RAM, 4 CPU cores
- Docker and Docker Compose installed
- Domain name (optional)

**Steps:**
```bash
# SSH into VPS
ssh root@your-server-ip

# Install Docker
curl -fsSL https://get.docker.com | sh

# Clone repo
git clone <repo-url>
cd mistral-vibe-web

# Set environment
echo "MISTRAL_API_KEY=your_key" > .env

# Start with auto-restart
docker-compose up -d

# Optional: Set up Nginx reverse proxy
apt install nginx
# Configure nginx to proxy port 80 → 3000
```

**Nginx config:**
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /ws {
        proxy_pass http://localhost:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'Upgrade';
    }
}
```

### 5.3 Cost Estimate (Monthly)

| Service | Provider | Cost |
|---------|----------|------|
| **VPS (8GB RAM)** | Hetzner CPX21 | $7/month |
| **Domain** | Namecheap | $12/year (~$1/month) |
| **SSL Cert** | Let's Encrypt | Free |
| **Total** | | **~$8/month** |

Note: Mistral API costs are separate (user provides their own key).

---

## 6. Next Steps

### After MVP is Working

**Phase 2 Features (Priority Order):**

1. **Session Persistence** (1 week)
   - Save session history to SQLite
   - Resume previous sessions
   - Session list UI

2. **File Editing** (1 week)
   - Make Monaco editor editable
   - Auto-save to container
   - File tree navigation

3. **Better Auth** (1 week)
   - User accounts (simple username/password)
   - Multiple users
   - Per-user API keys

4. **Improved UX** (1 week)
   - Better loading states
   - Error messages
   - Keyboard shortcuts
   - Dark/light theme toggle

5. **Tool Improvements** (1 week)
   - Better terminal emulation
   - Syntax highlighting in tool results
   - File diff viewer

### When to Scale Up

Move to the full architecture (from the main report) when you have:
- 100+ active users
- Need for horizontal scaling
- Team support requirements
- Enterprise features (SSO, RBAC)

---

## 7. Troubleshooting

### Common Issues

**Issue 1: Docker container won't start**
```bash
# Check Docker is running
docker ps

# View logs
docker-compose logs backend

# Rebuild image
docker-compose build --no-cache
```

**Issue 2: WebSocket connection fails**
```bash
# Test WebSocket with wscat
npm install -g wscat
wscat -c ws://localhost:4000

# Check firewall
sudo ufw allow 4000
```

**Issue 3: vibe-acp not responding**
```bash
# Test vibe-acp directly
docker run -it vibe-acp:latest

# Check API key is set
docker exec <container-id> env | grep MISTRAL
```

**Issue 4: Frontend can't connect to backend**
- Check CORS settings in backend
- Verify WebSocket URL in frontend (.env file)
- Check browser console for errors

---

## 8. Quick Start Checklist

### Day 1: Setup
- [ ] Install Node.js 20+
- [ ] Install Docker Desktop
- [ ] Get Mistral API key
- [ ] Clone template repo (if available)

### Week 1: Backend
- [ ] Build vibe-acp Docker image
- [ ] Create Node.js WebSocket server
- [ ] Test container creation/destruction
- [ ] Verify stdio ↔ WebSocket bridge works

### Week 2: Frontend
- [ ] Create Vite React app
- [ ] Build chat UI
- [ ] Connect to WebSocket
- [ ] Display messages

### Week 3-4: Integration
- [ ] Implement ACP protocol fully
- [ ] Add tool approval UI
- [ ] Integrate Monaco + xterm.js
- [ ] Test all tools

### Week 5-6: Polish
- [ ] Add error handling
- [ ] Session persistence
- [ ] Basic auth
- [ ] Deploy to VPS

---

## 9. Code Repository Structure

```
mistral-vibe-web/
├── backend/
│   ├── src/
│   │   ├── index.ts           # Main server
│   │   ├── docker-manager.ts  # Container lifecycle
│   │   ├── acp-bridge.ts      # stdio ↔ WebSocket
│   │   └── database.ts        # SQLite helpers
│   ├── package.json
│   └── tsconfig.json
├── frontend/
│   ├── src/
│   │   ├── App.tsx            # Main UI
│   │   ├── components/
│   │   │   ├── Chat.tsx
│   │   │   ├── CodeViewer.tsx
│   │   │   └── Terminal.tsx
│   │   └── hooks/
│   │       └── useWebSocket.ts
│   ├── package.json
│   └── vite.config.ts
├── docker/
│   └── Dockerfile.vibe-acp
├── docker-compose.yml
├── .env.example
└── README.md
```

---

## 10. Resources

### Documentation
- **Mistral Vibe Docs**: https://github.com/mistralai/mistral-vibe
- **ACP Spec**: https://github.com/anthropics/agent-client-protocol
- **Dockerode**: https://github.com/apocas/dockerode
- **Monaco Editor**: https://microsoft.github.io/monaco-editor/
- **xterm.js**: https://xtermjs.org/

### Tutorials
- Building WebSocket apps with Node.js
- Docker SDK for Node.js
- React + TypeScript best practices
- Deploying with Docker Compose

### Community
- Mistral Discord: https://discord.gg/mistralai
- GitHub Discussions: mistralai/mistral-vibe

---

## Conclusion

This simplified plan focuses on **getting something working quickly** rather than building a perfect production system. You can iterate and improve over time as you get user feedback.

**Key Principles:**
1. ✅ Start simple, add complexity as needed
2. ✅ Use managed services where possible
3. ✅ Focus on core functionality first
4. ✅ Deploy early, iterate fast
5. ✅ Don't over-engineer

**Timeline Summary:**
- **Weeks 1-2:** Backend foundation
- **Weeks 3-4:** Frontend basics
- **Weeks 5-6:** Tool integration
- **Weeks 7-8:** Polish & deploy

**Total Time:** 6-8 weeks part-time, 4-5 weeks full-time

Good luck! 🚀

---

**Questions?** Open an issue or reach out on Discord.
