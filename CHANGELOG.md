# Changelog

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

4. **附带问题：** `app/core/response_middleware.py` 的路径跳过列表中遗漏了 `"/video"`，导致 `/video` 页面请求会经过不必要的日志中间件处理。

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

#### 4. 修复 middleware 遗漏的路径

**文件：** `app/core/response_middleware.py`

在 `ResponseLoggerMiddleware.dispatch` 方法的路径跳过列表中添加 `"/video"`：

```python
# Before
if path.startswith("/static/") or path in (
    "/", "/login", "/imagine", "/voice",
    "/admin", "/admin/login", "/admin/config", "/admin/cache", "/admin/token",
):

# After
if path.startswith("/static/") or path in (
    "/", "/login", "/imagine", "/voice", "/video",
    "/admin", "/admin/login", "/admin/config", "/admin/cache", "/admin/token",
):
```

### 涉及文件清单

| 文件路径 | 变更类型 | 说明 |
|---------|---------|------|
| `app/static/public/` → `app/static/pub/` | 重命名 | 整个目录通过 git mv 重命名 |
| `app/api/pages/public.py` | 修改 | 4 处 FileResponse 路径更新 |
| `app/static/pub/pages/imagine.html` | 修改 | 2 处静态资源 URL 更新 |
| `app/static/pub/pages/login.html` | 修改 | 1 处静态资源 URL 更新 |
| `app/static/pub/pages/video.html` | 修改 | 2 处静态资源 URL 更新 |
| `app/static/pub/pages/voice.html` | 修改 | 2 处静态资源 URL 更新 |
| `app/core/response_middleware.py` | 修改 | 路径跳过列表添加 "/video" |

### 验证方式

1. 本地 `uv run main.py`，验证所有页面路由正常加载
2. 部署到 Vercel，测试开启"功能玩法"后访问 `/login`、`/imagine`、`/voice`、`/video`
3. 验证页面中 CSS/JS 资源正常加载（浏览器 DevTools Network 面板无 404）

### 经验总结

> **Vercel 部署时避免使用 `public` 作为目录名。** Vercel 沿用前端框架（Next.js、Vite 等）的约定，会对名为 `public` 的目录做特殊处理（如提升到根路径或从构建产物中排除），即使是 Python 后端项目也会受影响。如果静态资源目录必须放在项目内，使用 `pub`、`assets`、`www` 等替代名称。
