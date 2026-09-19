<!-- CODEGRAPH_START -->
## CodeGraph

In repositories indexed by CodeGraph (a `.codegraph/` directory exists at the repo root), reach for it BEFORE grep/find or reading files when you need to understand or locate code:

- **MCP tool** (when available): `codegraph_explore` answers most code questions in one call — the relevant symbols' verbatim source plus the call paths between them, including dynamic-dispatch hops grep can't follow. Name a file or symbol in the query to read its current line-numbered source. If it's listed but deferred, load it by name via tool search.
- **Shell** (always works): `codegraph explore "<symbol names or question>"` prints the same output.

If there is no `.codegraph/` directory, skip CodeGraph entirely — indexing is the user's decision.
<!-- CODEGRAPH_END -->

## Windows约束 
当前环境是 Windows 11 / pwsh7 
- 默认禁止使用 Bash 语法，除非确定此shell处在 Linux 环境 
- 不要使用 Bash 引号/转义习惯，在 PowerShell 命令里，复杂正则优先用单引号包裹。 
- 如果正则本身同时包含单引号和双引号，优先拆成多个简单 rg 命令。 
- 执行多行 Python 禁止使用 Bash heredoc；改用 PowerShell here-string | python - 
- pwsh 中，语句块表达式（如 `foreach`、`if`）不能直接作为管道输入。 需要先使用 `$()` / `@()` 包裹，或先赋值给变量。 普通命令输出可直接进入管道，无需额外包裹。 
- PowerShell 使用 `rg` 时，通配目录必须先用 `Get-ChildItem -Filter` 展开为真实路径，禁止直接把含 `*` 的搜索路径传给 `rg`。

超星学习通自动刷课脚本。无测试、无 lint、无类型检查——改完靠实际运行验证，不要找测试套件。

## 运行

- 安装：`pip install -r requirements.txt`（Python 要求 `>=3.13`，见 `pyproject.toml`；Docker 用 `python:3.13-slim`）。
- 推荐方式：复制 `config_template.ini` 为 `config.ini`，填好 `[common]` 账号密码，执行 `python main.py -c config.ini`。
- 命令行方式：`python main.py -u <手机号> -p <密码> -l <课程ID1,课程ID2> [-s 1.0] [-j 4] [--use-cookies] [--auto-sign] [-v]`。
- Docker：`ENTRYPOINT ["python3", "main.py", "-c", "/config/config.ini"]`，挂载 `/config` 卷；默认配置从 `config_template.ini` 复制而来。
- 关键参数：`-s/--speed` 在 `main()` 中被钳制到 `1.0–2.0`；`-j/--jobs` 为每门课程的并发章节数（默认 4）；未传 `-l` 或 `course_list` 为空时会交互式提问，直接回车代表刷全部课程。

## 结构

- `main.py` = 入口与编排：`init_config` → `init_chaoxing` → `login` → `filter_courses` → `process_course`。任务分发：`process_course`（按 `JobProcessor` 线程池并发处理章节）→ `process_chapter`（每章节内任务点用 `ThreadPoolExecutor(max_workers=5)` 并发）→ `process_job`（`video`/`document`/`workid`/`read`/`live` 五类）。
- `api/` 为全部库代码，模块分工见 `api/README.md`。核心在 `api/base.py`（`Chaoxing`、`SessionManager` 单例、`RateLimiter`、`StudyResult`）；题库在 `api/answer.py`；通知在 `api/notification.py`；直播在 `api/live.py` + `api/live_process.py`。
- 配置：INI 分 `[common]` / `[tiku]` / `[notification]` 三节，各键含义见 `config_template.ini` 行内注释。题库 `provider` 支持逗号分隔的回退链（如 `TikuGo,TikuYanxi,TikuLike`）；用 `AI`/`SiliconFlow` 时启动会做大模型连通性检查。

## 注意事项

- 密钥：真实账号密码绝不提交。`cookies.txt`（仓库根目录，即 `GlobalConst.COOKIES_PATH`）和填好内容的 `config.ini` 都是本地运行时文件，不要进 git。注意 `.github/workflows/main.yml` 目前在 `run:` 里硬编码了 `-u/-p/-l` 明文（虽然同时定义了 `CHAOXING_*` secrets），不要模仿这种写法。
- `--use-cookies` 从 `cookies.txt`（`k=v;` 格式，见 `api/cookies.py`）读取登录态，此时忽略 `-u/-p`。
- 日志：经 `api/logger.py` 用 `loguru` + `tqdm` 输出；`-v` 控制控制台级别（默认 `INFO`），完整 `TRACE` 始终写入 `chaoxing.log`（10 MB 轮转）。不要加 `print` 式日志。
- 并发：`requests.Session` 是共享的 `SessionManager` 单例（`HTTPAdapter(max_retries=10)`、5 秒超时）；靠 `RateLimiter` + 小随机休眠防突发请求。新增超星 HTTP 调用必须走该单例 session。
- 答题：`[tiku] submit=false` 表示只保存搜到的答案、不提交；`cover_rate` 控制自动提交的最低题库覆盖率；被章节测验锁定的后续章节必须提交才能解锁，改动前先看模板注释。
