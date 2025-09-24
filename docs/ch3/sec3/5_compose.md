# Docker Compose 实践

## 1。从单容器到容器编排

在前面的课程中，我们学习了如何使用 Docker 容器来运行单个服务。
通过 `docker run` 命令，我们可以快速启动一个数据库、一个 Web 服务器或者一个缓存服务。
这种方式在开发简单应用时非常有效。然而，随着应用架构的演进，微服务的理念逐渐流行，一个应用可能由多个相互依赖的服务组成。
如果继续使用单容器管理方式，我们需要手动管理容器间的网络连接、存储卷映射、环境变量配置等，这不仅增加了运维的复杂度，还容易因手动操作而出错。

这就是为什么我们需要一个更高层次的工具来管理多容器应用。
Docker Compose 应运而生，它通过一个声明式的 YAML 配置文件，帮助我们定义和管理多容器应用。
通过 Docker Compose，我们可以用一个命令就完成整个应用的部署，而不需要手动管理每个容器。

## 2。Docker Compose 核心概念

Docker Compose 是一个用于定义和运行多容器 Docker 应用程序的工具。使用 Compose，你可以通过一个 YAML 文件来配置应用程序的所有服务，然后使用一个命令来创建和启动所有服务。

### 2.1 主要概念

- **服务 (Services)**：容器的定义，包括使用哪个镜像、端口映射、环境变量等
- **网络 (Networks)**：定义容器之间如何通信
- **卷 (Volumes)**：定义数据的持久化存储
- **依赖关系 (Dependencies)**：定义服务之间的启动顺序
- **环境变量 (Environment Variables)**：管理不同环境的配置

### 2.2 核心命令

- `docker compose up`：创建和启动所有服务
- `docker compose down`：停止和删除所有服务
- `docker compose ps`：查看服务状态
- `docker compose logs`：查看服务日志

## 3。实践项目：使用 docker compose 构建 Todo 应用

在本章节中，我们通过一个最小可用的 Todo 应用来实战 Docker Compose 编排。

### 3.1 目标组件

- **Nginx**：统一入口与反向代理 (对外 8080)
- **前端**：React 应用 (容器内 3000，仅内网)
- **后端**：Node.js Express API (容器内 3001，仅内网)
- **数据库**：MongoDB (容器内 27017，仅内网)

### 3.2 项目结构 (示意)

```text
5_compose/
├── docker-compose.yml    # Compose 配置文件
├── nginx/                # Nginx 反向代理
├── frontend/             # React 前端应用
└── backend/              # Node.js 后端服务
```

### 3.3 架构图

```text
                        ┌─────────────┐
                        │   Nginx     │
                        │   :8080     │
                        └─────┬───────┘
                             │
                    ┌────────┴────────┐
                    │                 │
            ┌───────▼─────┐   ┌──────▼──────┐
            │  Frontend   │   │   Backend    │
            │  (React)    │   │  (Node.js)   │
            │   :3000     │   │    :3001     │
            └─────────────┘   └──────┬───────┘
                                    │
                            ┌───────▼───────┐
                            │   MongoDB     │
                            │  Database     │
                            │    :27017     │
                            └───────────────┘
```

### 3.4 Docker Compose 配置

将下列 `docker-compose.yml` 内容复制到你的工程中使用 (该 compose 通过容器内命令动态生成 `nginx.conf`、前端 `index.html` 与后端 `server.js`)：

```yaml
version: "3.9"
name: todo-app

services:
  # 统一入口网关：反向代理到 frontend 与 backend
  nginx:
    image: nginx:1.25-alpine
    container_name: todo_nginx
    ports:
      - "8080:80"
    depends_on:
      - frontend
      - backend
    command: >-
      sh -c '
      cat > /etc/nginx/nginx.conf <<"EOF"
      user  nginx;
      worker_processes  auto;
      events { worker_connections  1024; }
      http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;
        sendfile      on;
        keepalive_timeout  65;
        upstream frontend { server frontend:80; }
        upstream backend  { server backend:3001; }
        server {
          listen 80;
          location / { proxy_pass http://frontend; proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; }
          location /api/ { rewrite ^/api/?(.*)$ /$1 break; proxy_pass http://backend; proxy_set_header Host $host; proxy_set_header X-Real-IP $remote_addr; }
        }
      }
      EOF
      && nginx -g "daemon off;"'

  # 前端：无构建版 React（CDN 加载），由 nginx 直接静态托管
  frontend:
    image: nginx:1.25-alpine
    container_name: todo_frontend
    command: >-
      sh -c '
      cat > /usr/share/nginx/html/index.html <<"EOF"
      <!doctype html>
      <html>
        <head>
          <meta charset="UTF-8" />
          <meta name="viewport" content="width=device-width, initial-scale=1.0" />
          <title>Todo App</title>
          <style>
            body { font-family: ui-sans-serif, system-ui; max-width: 680px; margin: 24px auto; }
            li { display: flex; gap: 8px; align-items: center; padding: 6px 0; }
            button { cursor: pointer; }
          </style>
          <script crossorigin src="https://unpkg.com/react@18/umd/react.production.min.js"></script>
          <script crossorigin src="https://unpkg.com/react-dom@18/umd/react-dom.production.min.js"></script>
        </head>
        <body>
          <h1>Todo App</h1>
          <div id="root"></div>
          <script>
            const e = React.createElement;
            const API_BASE = '/api';
            function App(){
              const [todos, setTodos] = React.useState([]);
              const [text, setText] = React.useState('');
              async function load(){
                const res = await fetch(`${API_BASE}/todos`);
                setTodos(await res.json());
              }
              async function add(){
                if(!text.trim()) return;
                await fetch(`${API_BASE}/todos`, { method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ title: text })});
                setText('');
                load();
              }
              async function toggle(id, completed){
                await fetch(`${API_BASE}/todos/${id}`, { method:'PATCH', headers:{'Content-Type':'application/json'}, body: JSON.stringify({ completed: !completed })});
                load();
              }
              async function remove(id){ await fetch(`${API_BASE}/todos/${id}`, { method:'DELETE' }); load(); }
              React.useEffect(()=>{ load(); },[]);
              return e('div', null,
                e('div', { style:{ display:'flex', gap:8 } },
                  e('input', { value:text, onChange:ev=>setText(ev.target.value), placeholder:'What to do?', style:{ flex:1, padding:8 } }),
                  e('button', { onClick:add }, 'Add')
                ),
                e('ul', null, todos.map(t => e('li', { key:t._id },
                  e('input', { type:'checkbox', checked:!!t.completed, onChange:()=>toggle(t._id, !!t.completed) }),
                  e('span', { style:{ textDecoration: t.completed ? 'line-through' : 'none' } }, t.title),
                  e('button', { style:{ marginLeft:'auto' }, onClick:()=>remove(t._id) }, 'Delete')
                )))
              );
            }
            ReactDOM.createRoot(document.getElementById('root')).render(React.createElement(App));
          </script>
        </body>
      </html>
      EOF
      && nginx -g "daemon off;"'

  # 后端：Node.js（运行时安装依赖并生成 server.js）
  backend:
    image: node:18-alpine
    container_name: todo_backend
    environment:
      - MONGODB_URI=mongodb://mongodb:27017/todos
      - PORT=3001
    depends_on:
      - mongodb
    command: >-
      sh -c '
      cat > server.js <<"EOF"
      import express from "express";
      import cors from "cors";
      import { MongoClient, ObjectId } from "mongodb";
      const app = express();
      const port = process.env.PORT || 3001;
      const mongoUri = process.env.MONGODB_URI || "mongodb://localhost:27017/todos";
      app.use(cors());
      app.use(express.json());
      const client = new MongoClient(mongoUri);
      let collection;
      async function init(){
        await client.connect();
        const db = client.db();
        collection = db.collection("todos");
      }
      app.get("/health", (_req, res) => res.json({ ok: true }));
      app.get("/todos", async (_req, res) => {
        const items = await collection.find({}).sort({ _id: -1 }).toArray();
        res.json(items);
      });
      app.post("/todos", async (req, res) => {
        const doc = { title: String(req.body?.title ?? ""), completed: false };
        const r = await collection.insertOne(doc);
        res.status(201).json({ _id: r.insertedId, ...doc });
      });
      app.patch("/todos/:id", async (req, res) => {
        const id = req.params.id; const body = req.body || {};
        await collection.updateOne({ _id: new ObjectId(id) }, { $set: body });
        const updated = await collection.findOne({ _id: new ObjectId(id) });
        if(!updated) return res.status(404).json({ message:"Not Found" });
        res.json(updated);
      });
      app.delete("/todos/:id", async (req, res) => {
        const id = req.params.id;
        await collection.deleteOne({ _id: new ObjectId(id) });
        res.status(204).end();
      });
      init().then(()=> app.listen(port, () => console.log(`API listening on ${port}`)))
        .catch(err => { console.error("Mongo connect error", err); process.exit(1); });
      EOF
      && npm init -y >/dev/null 2>&1 \
      && npm i express@4 cors@2 mongodb@6 --silent \
      && node server.js'

  # 数据库：官方 MongoDB
  mongodb:
    image: mongo:7
    container_name: todo_mongodb
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

### 3.5 配置与代码说明

- **统一入口 `nginx`**：容器启动时写入 `nginx.conf` 并前台运行。
- **前端 `frontend`**：容器启动时生成 `index.html`，通过 CDN 加载 React，无需构建工具。
- **后端 `backend`**：容器启动时写入 `server.js`，随后初始化 npm 并安装 `express`、`cors`、`mongodb`，最后运行服务。
- **数据库 `mongodb`**：官方镜像，使用命名卷 `mongodb_data` 持久化数据。

### 3.6 服务解析

1. **nginx 服务**
   - 使用官方 `nginx:alpine` 镜像
   - 对外暴露 `8080:80`
   - 运行前生成 `nginx.conf`，将 `/` 转发到 `frontend`，`/api` 转发到 `backend`

2. **frontend 服务**
   - 使用官方 `nginx:alpine` 镜像
   - 启动时写入带 React CDN 的 `index.html`
   - 通过网关的 `/` 访问

3. **backend 服务**
   - 使用官方 `node:18-alpine` 镜像
   - 启动时生成 `server.js`，安装依赖并运行
   - 通过 `MONGODB_URI` 连接 `mongodb`

4. **mongodb 服务**
   - 使用官方 `mongo:7` 镜像
   - 使用命名卷 `mongodb_data` 持久化

`backend/src/index.js`：

```javascript
import express from "express";
import mongoose from "mongoose";
import cors from "cors";

const app = express();
const port = process.env.PORT || 3001;
const mongoUri = process.env.MONGODB_URI || "mongodb://localhost:27017/todos";

app.use(cors());
app.use(express.json());

const TodoSchema = new mongoose.Schema({
  title: { type: String, required: true },
  completed: { type: Boolean, default: false }
}, { timestamps: true });

const Todo = mongoose.model("Todo", TodoSchema);

app.get("/health", (_req, res) => res.json({ ok: true }));

app.get("/todos", async (_req, res) => {
  const items = await Todo.find().sort({ createdAt: -1 });
  res.json(items);
});

app.post("/todos", async (req, res) => {
  const todo = await Todo.create({ title: req.body.title ?? "" });
  res.status(201).json(todo);
});

app.patch("/todos/:id", async (req, res) => {
  const { id } = req.params;
  const todo = await Todo.findByIdAndUpdate(id, req.body, { new: true });
  if (!todo) return res.status(404).json({ message: "Not Found" });
  res.json(todo);
});

app.delete("/todos/:id", async (req, res) => {
  const { id } = req.params;
  await Todo.findByIdAndDelete(id);
  res.status(204).end();
});

mongoose.connect(mongoUri).then(() => {
  console.log("Connected to MongoDB");
  app.listen(port, () => console.log(`API listening on ${port}`));
}).catch((err) => {
  console.error("Mongo connection error", err);
  process.exit(1);
});
```

### 3.7 使用说明

`frontend/index.html`：

```html
<!doctype html>
<html>
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Todo App</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
  </html>
```

`frontend/src/main.jsx`：

```jsx
import React from "react";
import { createRoot } from "react-dom/client";
import App from "./App.jsx";

createRoot(document.getElementById("root")).render(<App />);
```

`frontend/src/App.jsx`：

```jsx
import React, { useEffect, useState } from "react";

const API_BASE = import.meta.env.VITE_API_BASE || "http://localhost:8080/api";

export default function App() {
  const [todos, setTodos] = useState([]);
  const [text, setText] = useState("");

  async function load() {
    const res = await fetch(`${API_BASE}/todos`);
    setTodos(await res.json());
  }

  async function add() {
    if (!text.trim()) return;
    await fetch(`${API_BASE}/todos`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title: text })
    });
    setText("");
    load();
  }

  async function toggle(id, completed) {
    await fetch(`${API_BASE}/todos/${id}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ completed: !completed })
    });
    load();
  }

  async function remove(id) {
    await fetch(`${API_BASE}/todos/${id}`, { method: "DELETE" });
    load();
  }

  useEffect(() => { load(); }, []);

  return (
    <div style={{ maxWidth: 560, margin: "24px auto", fontFamily: "ui-sans-serif, system-ui" }}>
      <h1>Todo App</h1>
      <div style={{ display: "flex", gap: 8 }}>
        <input value={text} onChange={e => setText(e.target.value)} placeholder="What to do?" style={{ flex: 1, padding: 8 }} />
        <button onClick={add}>Add</button>
      </div>
      <ul>
        {todos.map(t => (
          <li key={t._id} style={{ display: "flex", alignItems: "center", gap: 8, padding: "8px 0" }}>
            <input type="checkbox" checked={t.completed} onChange={() => toggle(t._id, t.completed)} />
            <span style={{ textDecoration: t.completed ? "line-through" : "none" }}>{t.title}</span>
            <button style={{ marginLeft: "auto" }} onClick={() => remove(t._id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### 3.8 配置解析与说明

#### 服务定义

1. **nginx 服务**
   - 使用官方 `nginx:alpine` 镜像
   - 绑定宿主机 `8080:80`，作为统一入口
   - 通过 `nginx.conf` 将 `/` 转发到 `frontend:3000`，`/api/` 转发到 `backend:3001`

2. **frontend 服务**
   - 基于 `node:18-alpine` 构建，使用 Vite 开发服务器 (端口 3000，仅容器网络)
   - `VITE_API_BASE` 指向 `http://nginx/api`
   - 使用绑定挂载支持热重载

3. **backend 服务**
   - 基于 `node:18-alpine` 构建，Express API 监听 3001 (仅容器网络)
   - 通过 `MONGODB_URI` 连接 `mongodb` 服务
   - 使用绑定挂载以便开发迭代

4. **mongodb 服务**
   - 使用官方 `mongo:7` 镜像
   - 使用命名卷 `mongodb_data` 持久化数据

#### 网络配置

- 使用默认 bridge 网络，服务间通过服务名互相访问 (`frontend`、`backend`、`mongodb`、`nginx`)

#### 数据持久化

- 命名卷 `mongodb_data` 用于持久化 MongoDB 数据
- 前后端使用绑定挂载以支持热更新 (生产环境建议改为只读镜像 + 构建产物)

### 3.9 使用说明

1. 在后台启动服务：

```bash
docker compose up -d
```

1. 查看服务状态：

```bash
docker compose ps
```

1. 查看服务日志：

```bash
docker compose logs nginx
docker compose logs frontend
docker compose logs backend
docker compose logs mongodb
```

1. 停止所有服务：

```bash
docker compose down
```

1. 重新构建服务 (镜像更新时)：

```bash
docker compose pull && docker compose up -d --force-recreate
```

1. 重启单个服务：

```bash
docker compose restart frontend
```

### 3.10 访问应用

本地开发 (如 VS Code) 直接在浏览器打开 `http://localhost:8080` 即可访问 Todo 应用。

- 使用 VS Code Dev Containers/Remote - Containers 时，`8080` 端口通常会自动转发；也可在 Ports 面板手动添加端口转发。
- 若端口被占用，可在 `docker-compose.yml` 中将 `8080:80` 改为其他可用端口 (如 `30080:80`)，然后重新启动：`docker compose up -d`。
