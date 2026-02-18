# Changelog

---

## Fork 与上游仓库的差异清单

> 上游仓库：[chenyme/grok2api](https://github.com/chenyme/grok2api) `upstream/main`
> 以下为本 fork（`test` 分支）相对于上游的**全部定制改动**，每次同步上游时需保留。

### 1. 静态资源目录 `static/public` → `static/pub`（Vercel 兼容）

**原因：** Vercel 构建/部署对名为 `public` 的目录有特殊处理（前端框架约定），导致 `app/static/public/` 下的文件在 Lambda 运行时不存在，`FileResponse` 内部 `os.stat()` 失败返回 500。

**涉及文件及改动：**

| 文件 | 改动说明 |
|------|---------|
| `app/static/pub/` (整个目录) | 由 `app/static/public/` 重命名而来 |
| `app/api/pages/public.py` | 4 处 `FileResponse` 路径 `"public/pages/..."` → `"pub/pages/..."` |
| `app/static/pub/pages/imagine.html` | CSS/JS 引用路径 `/static/public/` → `/static/pub/` |
| `app/static/pub/pages/login.html` | JS 引用路径 `/static/public/` → `/static/pub/` |
| `app/static/pub/pages/video.html` | CSS/JS 引用路径 `/static/public/` → `/static/pub/` |
| `app/static/pub/pages/voice.html` | CSS/JS 引用路径 `/static/public/` → `/static/pub/` |

### 2. Serverless 配置跨实例同步（`SERVER_TYPE=serverless`）

**原因：** Vercel 等 serverless 环境下，每个请求可能由不同的 Lambda 实例处理，TOML 文件配置不共享。通过设置 `SERVER_TYPE=serverless`，在每次请求时定期从数据库重新加载配置，实现跨实例同步。

**涉及文件及改动：**

| 文件 | 改动说明 |
|------|---------|
| `vercel.json` | 保留 `"SERVER_TYPE": "serverless"` 环境变量（上游已移除） |
| `app/core/config.py` | 新增 `_last_load_time` 字段和 `reload_if_stale(interval=30)` 异步方法，间隔 30 秒从数据库重新加载配置 |
| `app/core/response_middleware.py` | 导入 `os` 和 `config`，读取 `SERVER_TYPE` 环境变量，在 `dispatch` 中调用 `_app_config.reload_if_stale()` |

### 3. MySQL SSL 连接支持（`?ssl=true` URL 参数）

**原因：** `aiomysql` 不支持通过 URL 查询参数启用 SSL（如 `mysql://host/db?ssl=true`），需要手动解析 URL 中的 `?ssl=true` 参数，将其转换为 `connect_args={"ssl": ssl.create_default_context()}`，然后从 URL 中移除 `ssl` 参数以避免 `aiomysql` 报错。

**涉及文件及改动：**

| 文件 | 改动说明 |
|------|---------|
| `app/core/storage.py:509-529` | 在 `SQLStorage.__init__` 中新增 SSL 参数解析逻辑：解析 URL 中的 `?ssl=true`，通过 `connect_args` 传入 SSL 上下文，从 URL 移除 `ssl` 参数 |

---

## 2026-02-18 — merge: sync upstream/main updates into test branch

**Branch:** `test`
**上游基准：** `upstream/main` @ `7b02255`

### 合并的上游变更

从上游合并了 11 个新提交（`fcc957c..7b02255`），包含以下功能和修复：

#### 1. 新模型：Grok 4.20 Beta

- **文件：** `app/services/grok/services/model.py`
- 新增 `grok-4.20-beta` 模型定义（`grok_model="grok-420"`，`model_mode="MODEL_MODE_GROK_420"`，`tier=BASIC`，`cost=LOW`）

#### 2. ResettableSession — 可自动重建的 HTTP 会话

- **新增文件：** `app/services/reverse/utils/session.py`
- `ResettableSession` 封装 `curl_cffi.requests.AsyncSession`，在遇到指定 HTTP 状态码（默认 403）时自动重建连接，用于轮换代理场景
- 支持通过配置 `retry.reset_session_status_codes` 自定义触发状态码
- **全量替换：** 以下文件中的 `AsyncSession` 均替换为 `ResettableSession`：
  - `app/services/grok/services/chat.py`
  - `app/services/grok/batch_services/assets.py`
  - `app/services/grok/batch_services/nsfw.py`
  - `app/services/grok/batch_services/usage.py`
  - `app/services/grok/utils/download.py`
  - `app/services/grok/utils/upload.py`
- **配置：** `config.defaults.toml` 新增 `reset_session_status_codes = [403]`

#### 3. 增强的工具文本提取（rollout_id 支持）

- **文件：** `app/services/grok/services/chat.py`
- 新增 `extract_tool_text(raw, rollout_id)` 函数，解析 `<xai:tool_usage_card>` 标签内容
- 支持 `web_search`、`search_images`、`chatroom_send` 等工具类型的结构化解析
- `StreamProcessor` 新增 `rollout_id` 字段、`_filter_tool_card` 方法，实现流式工具卡片过滤
- `CollectProcessor` 新增非流式工具卡片正则替换逻辑

#### 4. 文件上传流式传输

- **文件：** `app/services/grok/utils/upload.py`
- 文件获取请求添加 `stream=True` 参数，改善大文件下载时的内存占用

#### 5. 重试机制改进

- **文件：** `app/services/reverse/utils/retry.py`
- `on_retry` 回调支持异步函数（通过 `inspect.isawaitable` 检测并 `await`）
- `on_retry` 类型签名从 `Callable[..., None]` 改为 `Callable[..., Any]`

#### 6. Usage 服务 `remainingQueries` 回退

- **文件：** `app/services/grok/batch_services/usage.py`、`app/services/token/manager.py`
- 当 API 返回中缺少 `remainingTokens` 时，回退读取 `remainingQueries` 字段
- `TokenManager` 中对应的 quota 同步和恢复逻辑也增加了 `remainingQueries` 回退

#### 7. 视频下载 URL 尾部换行

- **文件：** `app/services/grok/utils/download.py`
- `fmt="url"` 返回值末尾添加 `\n` 换行符

#### 8. UI 改进

- **文件：** `app/static/pub/css/imagine.css`、`app/static/pub/js/imagine.js`
- 图片瀑布流新增状态标签（生成中 / 完成 / 失败），对应 CSS 样式类 `.image-status`
- 错误时标记对应图片为失败状态
- **新增文件：** `app/static/common/js/toast.js` — 通用 toast 组件

#### 9. 版本号升级

- `pyproject.toml`：`0.3.0` → `0.3.1`
- 全部 HTML 页面中的静态资源版本号：`v=0.3.0` → `v=0.3.1`
- `app/static/common/js/footer.js`、`header.js`、`public-header.js`：版本号更新

#### 10. 其他

- `.gitignore`：`data/` 和 `app/data/` 目录整体排除（原先仅排除子路径）
- `data/config.toml`：从版本控制中移除（由 `.gitignore` 排除）
- `vercel.json`：上游移除了 `SERVER_TYPE=serverless`（本 fork 保留，见差异清单第 2 项）
- `scripts/test_usage_response.py`：新增 Usage 响应测试脚本
- `readme.md` / `docs/README.en.md`：功能列表新增 grok-4.20-beta

### 冲突解决

4 个 HTML 文件存在冲突（上游将 `static/pub` 改回 `static/public`，本 fork 保持 `pub`）：

| 文件 | 解决方式 |
|------|---------|
| `app/static/pub/pages/imagine.html` | 保持 `/static/pub/` 路径，采纳版本号 `v=0.3.1` |
| `app/static/pub/pages/login.html` | 保持 `/static/pub/` 路径，采纳版本号 `v=0.3.1` |
| `app/static/pub/pages/video.html` | 保持 `/static/pub/` 路径，采纳版本号 `v=0.3.1` |
| `app/static/pub/pages/voice.html` | 保持 `/static/pub/` 路径，采纳版本号 `v=0.3.1` |

### 变更文件总计

33 个文件变更（1 个新增，1 个删除，31 个修改）。

---

## 2026-02-17 — fix: rename static/public to static/pub to avoid Vercel 500 error

**Commit:** `007219b`
**Branch:** `test`

### 问题现象

Vercel 部署环境中，admin 页面（`/admin/login`、`/admin/config` 等）完全正常，但开启"功能玩法"后访问以下 public 页面返回 **500 Internal Server Error**：

- `/login`
- `/imagine`
- `/voice`
- `/video`

其他部署方式（本地、Docker 等）无此问题，仅 Vercel 环境复现。

### 根因分析

1. 错误响应 `{"error":{"message":"Internal server error","type":"server_error","param":null,"code":"internal_error"}}` 来自 `generic_exception_handler`（`app/core/exceptions.py:198`），说明路由函数抛出了非 `HTTPException` 的未处理异常。

2. Admin 和 Public 页面路由使用**完全相同**的 `STATIC_DIR` 解析 + `FileResponse` 模式。`is_public_enabled()` 已验证不会抛异常。唯一区别是文件路径子目录：`admin/pages/` 正常 vs `public/pages/` 报错。

3. **根本原因：** Vercel 构建/部署对名为 `public` 的目录有特殊处理（前端框架约定），导致 `app/static/public/` 下的文件在 Lambda 运行时**不存在**。`FileResponse` 内部调用 `os.stat()` 失败，抛出 `RuntimeError`，被 `generic_exception_handler` 捕获后返回 500。

### 修改内容

#### 1. 重命名静态资源目录

```
app/static/public/  →  app/static/pub/
```

通过 `git mv` 重命名，避免 Vercel 对 `public` 目录名的特殊处理。涉及以下文件的整体迁移：

- `css/imagine.css`, `css/video.css`, `css/voice.css`
- `js/imagine.js`, `js/login.js`, `js/video.js`, `js/voice.js`
- `pages/imagine.html`, `pages/login.html`, `pages/video.html`, `pages/voice.html`

#### 2. 更新 Python 路由文件中的路径引用

**文件：** `app/api/pages/public.py`

4 处 `FileResponse` 路径更新：

```python
# Before
FileResponse(STATIC_DIR / "public/pages/login.html")
FileResponse(STATIC_DIR / "public/pages/imagine.html")
FileResponse(STATIC_DIR / "public/pages/voice.html")
FileResponse(STATIC_DIR / "public/pages/video.html")

# After
FileResponse(STATIC_DIR / "pub/pages/login.html")
FileResponse(STATIC_DIR / "pub/pages/imagine.html")
FileResponse(STATIC_DIR / "pub/pages/voice.html")
FileResponse(STATIC_DIR / "pub/pages/video.html")
```

#### 3. 更新 HTML 文件中的静态资源引用

所有 `/static/public/` 引用替换为 `/static/pub/`：

| 文件 | 修改处数 | 具体内容 |
|------|---------|---------|
| `app/static/pub/pages/imagine.html` | 2 处 | CSS (`imagine.css`) + JS (`imagine.js`) |
| `app/static/pub/pages/login.html` | 1 处 | JS (`login.js`) |
| `app/static/pub/pages/video.html` | 2 处 | CSS (`video.css`) + JS (`video.js`) |
| `app/static/pub/pages/voice.html` | 2 处 | CSS (`voice.css`) + JS (`voice.js`) |

### 涉及文件清单

| 文件路径 | 变更类型 | 说明 |
|---------|---------|------|
| `app/static/public/` → `app/static/pub/` | 重命名 | 整个目录通过 git mv 重命名 |
| `app/api/pages/public.py` | 修改 | 4 处 FileResponse 路径更新 |
| `app/static/pub/pages/imagine.html` | 修改 | 2 处静态资源 URL 更新 |
| `app/static/pub/pages/login.html` | 修改 | 1 处静态资源 URL 更新 |
| `app/static/pub/pages/video.html` | 修改 | 2 处静态资源 URL 更新 |
| `app/static/pub/pages/voice.html` | 修改 | 2 处静态资源 URL 更新 |

### 验证方式

1. 本地 `uv run main.py`，验证所有页面路由正常加载
2. 部署到 Vercel，测试开启"功能玩法"后访问 `/login`、`/imagine`、`/voice`、`/video`
3. 验证页面中 CSS/JS 资源正常加载（浏览器 DevTools Network 面板无 404）

### 经验总结

> **Vercel 部署时避免使用 `public` 作为目录名。** Vercel 沿用前端框架（Next.js、Vite 等）的约定，会对名为 `public` 的目录做特殊处理（如提升到根路径或从构建产物中排除），即使是 Python 后端项目也会受影响。如果静态资源目录必须放在项目内，使用 `pub`、`assets`、`www` 等替代名称。
