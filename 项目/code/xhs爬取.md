我先打开 `xhs_search` 目录下的核心脚本（搜索、详情、评论、用户详情、并发控制、API与签名工具等）以梳理完整的爬取流程与反爬策略，然后给出结构化说明。
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool
Ran tool

我已通读 `xhs_search` 目录的核心代码，下面给你一份“从输入到返回”的完整爬取流程说明。

### 总体架构
- **技术栈**：`requests + PyExecJS`（本地 Node 运行时）进行 HTTP 调用与签名头生成；`loguru` 记录日志。
- **分层**：
  - `apis/xhs_pc_apis.py`：封装所有小红书 PC Web API 的 HTTP 请求与分页逻辑。
  - `xhs_utils/xhs_util.py`：用本地 JS 脚本生成反爬所需签名头（X-s、X-t、x-s-common、x-b3-traceid、x-xray-traceid）。
  - `tool/*.py`：面向 Agent 的工具函数包装（带并发控制、参数校验、返回 JSON 字符串，便于 LLM 工具调用）。
  - `concurrency_control.py`：每账号与全局并发限流。

### 鉴权与反爬绕过
- 必须提供有效的浏览器登录 Cookie，关键是 `a1` 字段（作为加密输入）。
- 通过本地 JS 文件计算反爬头：
  - `static/xhs_xs_xsc_56.js` 计算 `X-s`、`X-t`、`x-s-common`
  - `static/xhs_xray.js` 计算 `x-xray-traceid`
  - Python 侧还会生成 `x-b3-traceid`（随机十六进制串）
- 代码位置（签名与头构造）：
```17:35:agents/graph/tool/xhs_search/xhs_utils/xhs_util.py
def generate_xs_xs_common(a1, api, data=''):
    ret = js.call('get_request_headers_params', api, data, a1)
    xs, xt, xs_common = ret['xs'], ret['xt'], ret['xs_common']
    return xs, xt, xs_common
```
```78:94:agents/graph/tool/xhs_search/xhs_utils/xhs_util.py
def generate_headers(a1, api, data=''):
    xs, xt, xs_common = generate_xs_xs_common(a1, api, data)
    x_b3_traceid = generate_x_b3_traceid()
    headers = get_request_headers_template()
    headers['x-s'] = xs
    headers['x-t'] = str(xt)
    headers['x-s-common'] = xs_common
    headers['x-b3-traceid'] = x_b3_traceid
    ...
    return headers, data
```
- 请求前统一打包参数（含 Cookie 转换、签名、头）：
```90:95:agents/graph/tool/xhs_search/xhs_utils/xhs_util.py
def generate_request_params(cookies_str, api, data=''):
    cookies = trans_cookies(cookies_str)
    a1 = cookies['a1']
    headers, data = generate_headers(a1, api, data)
    return headers, cookies, data
```

### 关键能力与调用流
- 搜索笔记：`/api/sns/web/v1/search/notes`
  - 工具函数入口：
```19:76:agents/graph/tool/xhs_search/search_notes.py
@tool
@with_concurrency_control
def search_notes(...):
    ensure_project_path()
    from apis.xhs_pc_apis import XHS_Apis
    api = XHS_Apis()
    success, msg, res_json = api.search_note(...)
    ...
```
  - 实际 HTTP 调用：
```415:521:agents/graph/tool/xhs_search/apis/xhs_pc_apis.py
def search_note(self, query: str, cookies_str: str, page=1, ...):
    api = "/api/sns/web/v1/search/notes"
    data = {..., "filters":[...], "image_formats":["jpg","webp","avif"]}
    headers, cookies, data = generate_request_params(cookies_str, api, data)
    response = requests.post(self.base_url + api, headers=headers, data=data.encode('utf-8'), cookies=cookies, proxies=proxies)
    res_json = response.json()
```
  - 支持组合筛选（排序、类型、时间、范围、距离）、分页循环（`search_some_note`）。

- 获取笔记详情：`/api/sns/web/v1/feed`
  - URL 中必须携带 `xsec_token` 与 `xsec_source`（从页面链接上带出）。
```354:389:agents/graph/tool/xhs_search/apis/xhs_pc_apis.py
def get_note_info(self, url: str, cookies_str: str, proxies: dict = None):
    urlParse = urllib.parse.urlparse(url)
    note_id = urlParse.path.split("/")[-1]
    kvDist = {...}  # 提取 xsec_token/xsec_source
    api = f"/api/sns/web/v1/feed"
    data = {"source_note_id": note_id, "xsec_source": ..., "xsec_token": kvDist['xsec_token'], ...}
    headers, cookies, data = generate_request_params(cookies_str, api, data)
    response = requests.post(self.base_url + api, headers=headers, data=data, cookies=cookies, proxies=proxies)
```
  - 工具封装：`tool/note_detail.py` 的 `get_note_detail(...)`。

- 获取评论
  - 一级评论：`/api/sns/web/v2/comment/page`
  - 二级评论：`/api/sns/web/v2/comment/sub/page`
  - 聚合全部评论：先拉全量一级，再按需分页拉取二级并拼接。
```616:669:agents/graph/tool/xhs_search/apis/xhs_pc_apis.py
def get_note_out_comment(...):  # 一级评论
def get_note_all_out_comment(...):  # 一级评论全量分页
def get_note_inner_comment(...):  # 二级评论
def get_note_all_inner_comment(...):  # 二级评论全量分页并合并到父评论
```
  - 工具封装：`tool/note_comments.py` 的 `get_note_comments(...)`（从 URL 中提取 `note_id` 与 `xsec_token`）。

- 用户信息与内容列表
  - 用户详情：`/api/sns/web/v1/user/otherinfo`
  - 用户发布/点赞/收藏列表：对应的 `user_posted`、`note/like/page`、`note/collect/page`，均支持分页聚合（All_* 方法）。

### 并发与稳定性
- 并发控制：按 Cookie（用户）限流，默认每个 Cookie 最多 2 个并发，全局最多 10。
```10:24:agents/graph/tool/xhs_search/concurrency_control.py
class ConcurrencyController:
    def __init__(self, max_per_cookie: int = 2, max_global: int = 10):
```
- 包装方式：`@with_concurrency_control` 装饰器自动基于传入的 `cookie` 参数排队执行，避免同账号频繁并发触发风控。
- 代理支持：所有 API 均接受 `proxies` 透传到 `requests`。

### 目录技巧与执行依赖
- `ensure_project_path()`：在工具函数入口处切换当前工作目录并注入 `sys.path`，保证本目录的 `apis/`、`xhs_utils/` 与 `static/` JS 能被正确加载（ExecJS 编译时要读 JS 文件）。
- 局部依赖：`agents/graph/tool/xhs_search/requirements.txt` 额外声明了 `PyExecJS/requests/loguru`，确保签名与 HTTP 可用。
- Node 运行时：`PyExecJS` 需要系统已安装 Node（或其他 JS 引擎）。

### 常见注意事项
- 必须提供有效登录 Cookie，特别是 `a1`；`xsec_token/xsec_source` 需从站内链接带出。
- 反爬 JS 版本变化会导致签名失效，需要更新 `static/*.js`。
- UA、Referer、Origin 等头不可随意更改（模板已内置）。
- 出现 403/401/签名校验失败时，先检查 Cookie 是否过期、是否缺少 `a1`、`xsec_token` 是否来自最新页面链接。

如果你需要，我可以提供一个最小可运行示例（给出真实 `cookie` 与页面 URL）来演示“搜索→详情→评论”的完整链路调用。
