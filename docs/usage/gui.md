## 使用图形界面

图形界面由 FastAPI 后端和 React 前端组成。

### 一键启动

如果你已安装图形终端模拟器（gnome-terminal、konsole、alacritty 等），或已安装 `tmux` 作为回退：

```bash
./start.sh
```

该脚本会自动激活 `naturalcc` 环境，并打开两个终端窗口（或 tmux 分屏）：
- 一个运行 FastAPI 后端
- 一个运行 Vite 前端开发服务器

使用 tmux 时，session 名称为 `ncc-agent`。按 `Ctrl+B` 再按 `D` 可分离会话；重新 attach 用 `tmux attach -t ncc-agent`。

### 开发模式

适合修改前端代码时使用，支持 Vite 热更新。

终端 1：

```bash
python agent_web_api.py --host 127.0.0.1 --port 7860
```

终端 2：

```bash
cd webui
npm run dev
```

打开：

```text
http://127.0.0.1:5173/
```

开发模式下，Vite 在 `5173` 提供前端页面，并把 `/api/*` 请求代理到 `7860` 的 FastAPI 后端。

### 本地打包模式

适合只启动一个服务来同时提供 UI 和 API。

```bash
cd webui
npm run build
cd ..
python agent_web_api.py --host 127.0.0.1 --port 7860
```

打开：

```text
http://127.0.0.1:7860/
```

打包模式下，FastAPI 会服务 `webui/dist`，并在同一端口提供所有后端 API。

### UI 使用流程

1. 设置项目根目录。
2. 选择一个或多个目标文件。
3. 确认第一个目标文件是主解析文件。
4. 选择或输入模型名。
5. 可选填写 API Key、symbol、completion type 或 prefix。
6. 输入开发指令。
7. 点击 preview 预览最终 prompt，或点击 execute 调用 Aider 执行修改。
