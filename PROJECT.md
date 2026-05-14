# Collaborative Code Editor — Project Overview

A **real-time collaborative code editor** where multiple users can join a session, write code together in the same editor, and see each other's changes live — similar to Google Docs but for code.

**Live URL:** http://docker-aws-yt-alb-790815787.ap-northeast-1.elb.amazonaws.com/

---

## What Does This App Do?

1. User visits the app and enters a **username** to join
2. A **Monaco code editor** (same editor used in VS Code) loads in the browser
3. Multiple users can **type in the same editor simultaneously** — changes appear in real-time for everyone
4. A **sidebar shows all connected users** currently in the session
5. When a user leaves/closes the tab, they are **automatically removed** from the users list

---

## Tech Stack

| Layer     | Technology              | Why                                                        |
| --------- | ----------------------- | ---------------------------------------------------------- |
| Frontend  | **React 19**            | Component-based UI, hooks for state management             |
| Editor    | **Monaco Editor**       | VS Code's editor in the browser — syntax highlighting, etc |
| Styling   | **Tailwind CSS 4**      | Utility-first CSS, rapid styling without writing CSS files |
| Bundler   | **Vite 7**              | Lightning-fast dev server and optimized production builds  |
| Backend   | **Express 5** (Node.js) | Lightweight HTTP server to serve frontend + handle sockets |
| Realtime  | **Socket.IO**           | WebSocket-based real-time bidirectional communication      |
| CRDT Sync | **Yjs + y-socket.io**   | Conflict-free merging of simultaneous edits (CRDT)         |
| Binding   | **y-monaco**            | Bridges Yjs document ↔ Monaco Editor model                 |

---

## Frontend — How It Works

### File Structure

```
Frontend/
├── index.html              # Entry HTML
├── vite.config.js          # Vite config with React + Tailwind plugins
├── package.json            # Dependencies
└── src/
    ├── main.jsx            # React root render
    └── app/
        ├── App.jsx         # Main component (editor + users sidebar)
        └── App.css         # Tailwind import
```

### Key Concepts in `App.jsx`

#### 1. Username / Login Screen

```jsx
const [username, setUsername] = useState(() => {
  return new URLSearchParams(window.location.search).get("username") || "";
});
```

- On load, checks the URL for `?username=xyz`
- If no username → shows a simple **join form**
- On submit → sets username in state and updates the URL (`?username=vinay`)

#### 2. Yjs Document (CRDT)

```jsx
const ydoc = useMemo(() => new Y.Doc(), []);
const yText = useMemo(() => ydoc.getText("monaco"), [ydoc]);
```

- **Yjs** is a CRDT (Conflict-free Replicated Data Type) library
- Creates a shared document (`Y.Doc`) and a shared text type (`Y.Text`)
- Any edit by any user is **automatically merged without conflicts** — even if two people type at the same position simultaneously

#### 3. Socket.IO Provider (Real-time Sync)

```jsx
const provider = new SocketIOProvider(window.location.origin, "monaco", ydoc, {
  autoConnect: true,
});
```

- Connects to the backend via **WebSockets** (Socket.IO)
- Uses `window.location.origin` so it works in both:
  - **Local dev:** connects to `http://localhost:3000`
  - **Production (AWS):** connects to the ALB URL automatically
- The provider syncs the Yjs document across all connected clients

#### 4. Awareness (Who's Online)

```jsx
provider.awareness.setLocalStateField("user", { username });

provider.awareness.on("change", () => {
  const states = Array.from(provider.awareness.getStates().values());
  setUsers(
    states
      .filter((state) => state.user && state.user.username)
      .map((state) => state.user),
  );
});
```

- **Awareness** is a Yjs feature that tracks ephemeral state (like who's connected)
- Each client sets their `username` in awareness
- On any change (join/leave), the users list updates for everyone
- On tab close (`beforeunload`), the user's awareness is cleared

#### 5. Monaco Editor + Binding

```jsx
<Editor
  height="100%"
  defaultLanguage="javascript"
  theme="vs-dark"
  onMount={handleMount}
/>
```

```jsx
new MonacoBinding(
  yText,
  editorRef.current.getModel(),
  new Set([editorRef.current]),
);
```

- Uses `@monaco-editor/react` to render the VS Code editor
- **MonacoBinding** connects the Yjs shared text → Monaco editor model
- Any keystroke in the editor updates the Yjs doc → syncs to all other clients → updates their editors

### Frontend Dependencies

| Package                | Purpose                                      |
| ---------------------- | -------------------------------------------- |
| `react`, `react-dom`   | UI framework                                 |
| `@monaco-editor/react` | Monaco Editor as a React component           |
| `yjs`                  | CRDT library for conflict-free collaboration |
| `y-socket.io`          | Yjs ↔ Socket.IO transport provider           |
| `y-monaco`             | Yjs ↔ Monaco Editor binding                  |
| `tailwindcss`          | Utility CSS framework                        |
| `vite`                 | Build tool and dev server                    |

---

## Backend — How It Works

### File Structure

```
Backend/
├── package.json
├── server.js           # The entire server — simple and minimal
└── public/             # (generated at build time — contains compiled frontend)
```

### `server.js` Breakdown

```js
import express from "express";
import { createServer } from "http";
import { Server } from "socket.io";
import { YSocketIO } from "y-socket.io/dist/server";
```

1. **Express** → serves the static frontend files from `public/`
2. **HTTP Server** → required so Socket.IO can attach to it
3. **Socket.IO** → WebSocket server with CORS enabled for all origins
4. **YSocketIO** → Yjs server-side provider that:
   - Manages shared Yjs documents
   - Syncs document state between all connected clients
   - Handles new connections (sends current doc state)
   - Merges incoming changes from clients

```js
app.use(express.static("public"));
```

- Serves the compiled React frontend (Vite build output)
- This is why a single server handles both frontend and backend

```js
app.get("/health", (req, res) => {
  res.status(200).json({ message: "ok", success: true });
});
```

- Health check endpoint used by **AWS ALB** to verify the container is running

```js
httpServer.listen(3000, () => { ... });
```

- Server runs on port **3000** inside the container

### Backend Dependencies

| Package       | Purpose                             |
| ------------- | ----------------------------------- |
| `express`     | HTTP server and static file serving |
| `socket.io`   | WebSocket communication             |
| `y-socket.io` | Yjs document sync over Socket.IO    |

---

## Features Summary

| Feature                            | How It Works                                   |
| ---------------------------------- | ---------------------------------------------- |
| Real-time collaborative editing    | Yjs CRDT + Socket.IO sync across all clients   |
| VS Code-quality editor             | Monaco Editor with syntax highlighting         |
| Live user presence                 | Yjs Awareness protocol tracks connected users  |
| No conflicts on simultaneous edits | CRDT merges changes automatically              |
| Username-based sessions            | URL param + join form                          |
| Auto-cleanup on disconnect         | `beforeunload` clears user from awareness      |
| Single server deployment           | Express serves both frontend build + WebSocket |
| Health check for AWS               | `/health` endpoint for ALB target group        |

---

## How to Run Locally

```bash
# Terminal 1 — Backend
cd Backend
npm install
npm run dev        # starts on http://localhost:3000

# Terminal 2 — Frontend
cd Frontend
npm install
npm run dev        # starts on http://localhost:5173
```

> **Note:** In local dev, the frontend dev server (Vite) runs on port 5173, and the backend on 3000. The SocketIOProvider connects to `window.location.origin`, so for local development you may want to temporarily hardcode `http://localhost:3000` or set up a Vite proxy.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────┐
│                  Browser                     │
│                                              │
│  ┌──────────┐     ┌───────────────────────┐ │
│  │  Join     │     │   Monaco Editor       │ │
│  │  Form     │────▶│   (y-monaco binding)  │ │
│  └──────────┘     └───────────┬───────────┘ │
│                               │              │
│  ┌──────────┐     ┌───────────▼───────────┐ │
│  │  Users    │◀────│   Yjs + SocketIO      │ │
│  │  Sidebar  │     │   Provider            │ │
│  └──────────┘     └───────────┬───────────┘ │
│                               │              │
└───────────────────────────────┼──────────────┘
                                │ WebSocket
                                ▼
┌───────────────────────────────────────────────┐
│              Node.js Server (port 3000)        │
│                                                │
│  Express ──── Static Files (React build)       │
│  Socket.IO ── YSocketIO (Yjs doc sync)         │
│  /health ──── ALB health check                 │
└────────────────────────────────────────────────┘
```
