## Context

应用现状（见已归档设计与 proposal）：单体 Flask（应用工厂 + Blueprint + Jinja2 服务端渲染）+ Flask-SQLAlchemy + SQLite，材料落盘到 `UPLOAD_ROOT/<class_id>/<uuid>.<ext>`，班级过滤只存在于 `materials/repository.py`；允许类型由 `Config.ALLOWED_EXTENSIONS` 与 `ALLOWED_MIME_TYPES` 双白名单控制；建表仅靠 `db.create_all()`（不引入 Alembic）；容器入口固定执行 `flask init-db && flask seed`，gunicorn 2 workers，全部持久数据在命名卷 `/data`。约束延续：纯 Python 优先、Windows/课堂环境可复现、单容器单卷、鉴权与班级边界只信服务端会话。

本期新增面横跨材料模块、新检索模块、CLI、seed、配置与镜像构建，且引入嵌入模型与向量存储，故需要本设计。检索方案已与用户确认为**本地嵌入模型 + 向量库**。

## Goals / Non-Goals

**Goals:**

- txt/md 可上传（UTF-8 校验）、可在浏览器安全预览（txt 纯文本、md 渲染并消毒），师生双端体验一致。
- txt/md 上传后自动进入**按 class_id 强制隔离**的向量知识库；检索结果带来源文件、片段与定位锚点。
- 全离线：模型随镜像分发，运行时不依赖外网与第三方密钥。
- 索引与 SQLite 同库同卷持久；对已有卷中的旧库做幂等轻量升级；`flask reindex-kb` 可回填/重建且幂等。
- 测试不下载模型：嵌入后端可替换为确定性 stub。

**Non-Goals（本期不做）:**

- pdf/docx/pptx/xlsx/图片的内容抽取与预览（仅保留下载；知识库也不索引它们）。
- 问答机器人/LLM 生成答案、检索结果的 LLM 重排、多语言模型切换 UI。
- 材料的删除/编辑（因此索引的删除路径只服务于"重建"）；一师多班。
- 外部嵌入 API 后端（代码保留抽象，但不提供也不测试真实 API 通路）。
- 知识库的管理后台、索引失败的自动重试队列与异步任务框架。

## Decisions

### 1. 模块结构：新增 `app/kb/` 包，材料侧只加预览

```
app/kb/
  __init__.py        # kb_bp（检索页）
  chunking.py        # 文本切分、小节标题提取、行号/锚点计算
  embeddings.py      # Embedder 抽象 + FastEmbed 后端 + Stub 后端（单例）
  vector_store.py    # sqlite-vec 扩展加载、虚拟表 DDL、upsert/delete/KNN
  indexer.py         # 单材料（重）建索引：读文件→切分→嵌入→事务替换
  service.py         # 检索编排：校验→嵌入→KNN→本班过滤→溯源结果
  routes.py          # GET /kb（空态与带 q 的结果页）
  repository.py      # kb_chunks 的唯一数据访问层（所有读带 class_id）
templates/kb/search.html
templates/materials/preview.html
```

预览加入既有 `materials` Blueprint（`GET /materials/<id>/preview`），不新开模块；上传流程仍在 `materials/routes.py`，索引动作在材料提交成功后由 `kb.indexer` 完成。

### 2. 上传链路改造：文本先校验 UTF-8，再落盘→写库→建索引

- `Config.ALLOWED_EXTENSIONS` 默认值追加 `txt`、`md`；`ALLOWED_MIME_TYPES` 增加 `txt: {text/plain}` 与 `md: {text/markdown, text/x-markdown, text/plain}`（不同浏览器/系统对 .md 声明不一致，允许 text/plain，仍维持扩展名+MIME 双白名单）。新增 `PREVIEWABLE_EXTENSIONS = {"txt", "md"}`。
- 上传处理对 txt/md 改为：读入字节（20MB 上限内可接受）→ `bytes.decode("utf-8")` 严格解码，失败直接 400 且不触盘→用既有 `storage.write_bytes` 落盘→`repository.create_material`（保持"写库失败删文件"的补偿）。
- 材料记录提交成功后调用 `indexer.index_material(material)`：其内部对失败自洽（见 Decision 6），**上传接口不因索引失败而回滚材料**，仅记录日志并保持 `indexed_at` 为空。
- 列表/详情模板通过服务端注入的模板变量（`allowed_extensions`、`previewable_extensions`，由 `g`/context processor 从 config 派生）渲染格式清单与"预览"入口，杜绝页面写死。

### 3. 预览：服务端渲染 + bleach 白名单消毒

- 路由复用 `repository.get_scoped_or_404(user, id)`：跨班天然 404；扩展名不在 `PREVIEWABLE_EXTENSIONS` 时 `abort(415)`；物理文件缺失与下载一致返回 500。
- txt：模板中 `<pre>{{ content }}</pre>`，依赖 Jinja 自动转义，换行空白原样保留。
- md：`markdown.markdown(text, extensions=["fenced_code", "tables", "toc"])` 后用 `bleach.clean(html, tags=..., attributes=..., protocols={"http","https"})` 消毒，才允许在模板 `|safe`。白名单只含排版标签（标题、p、ul/ol/li、pre/code、blockquote、em/strong、a、table 系列等），**不含 script/iframe/img 事件属性**；链接强制 noopener。
- 片段定位：`chunking` 同时记录每个块的首行行号与最近标题。txt 预览按行输出 `<span id="L{n}">`，溯源链接用 `#L{line}`；md 借助 toc 扩展的标题 slug（索引与预览共用 `markdown.extensions.toc.slugify` 同一函数），溯源链接用 `#{slug}`。锚点只在本页内定位，不引入 JS。

### 4. 数据模型：常规表 + sqlite-vec 虚拟表，同库存储

- `materials` 增加可空列 `indexed_at DATETIME`（非文本材料与未索引材料为 NULL，同时充当回填判据）。
- 新表 `kb_chunks`：`id`、`material_id FK→materials.id`、`class_id`（冗余落列并建索引，是班级过滤的物理保证）、`chunk_index`、`char_start`、`line_no`、`heading NULL`、`content TEXT`、`created_at`；`UNIQUE(material_id, chunk_index)`。
- 向量表为 sqlite-vec 的 `vec0` 虚表：`kb_vectors(chunk_id INTEGER PRIMARY KEY, embedding FLOAT[512])`，通过裸 DDL `CREATE VIRTUAL TABLE IF NOT EXISTS` 建立，与主库同一文件，因此天然随 `/data/campusclaw.db` 持久，无需第二个卷。
- 既有卷升级（`db.create_all()` 不会给旧表加列）：新增幂等 `ensure_runtime_schema()`——用 `PRAGMA table_info(materials)` 检查缺列则 `ALTER TABLE materials ADD COLUMN indexed_at DATETIME`，并确保虚表存在；在 `create_app` 的既有 sqlite 初始化段与 `flask init-db` 中各调用一次。不引入 Alembic。
- sqlite-vec 扩展在现有 sqlite 连接事件里加载（与 `PRAGMA foreign_keys/WAL` 同处）：`import sqlite_vec; sqlite_vec.load(dbapi_connection)`，之后 SQLAlchemy 会话即可在同一连接上对虚表做 DML/KNN。

### 5. 嵌入：fastembed（ONNX）+ 后端抽象，镜像预热

- `embeddings.py` 定义 `Embedder`（`dim`、`embed(list[str]) -> list[list[float]]`），两个实现：
  - **FastEmbedEmbedder**（默认）：模型 `BAAI/bge-small-zh-v1.5`（512 维，中文句向量），经 `fastembed.TextEmbedding(model_name=..., cache_dir=Config.KB_MODEL_DIR)` 加载；相对 sentence-transformers 免 torch（onnxruntime + tokenizers，镜像增量约 300MB，模型约 100MB）。运行时以 `local_files_only` 方式使用缓存目录，缺失即报错并提示重建镜像，不发起外网兜底。
  - **StubEmbedder**（`KB_EMBEDDING_BACKEND=stub`，测试专用）：对文本做确定性分词哈希，投影到低维向量并归一化；相同词条共现的查询与文本得分更高，使隔离/命中/排序断言无需模型即可运行。
- 应用内 embedder 为惰性单例（首次入库/检索时加载），避免仅健康检查就占用模型内存。
- Dockerfile：`apt-get install -y --no-install-recommends libgomp1`（onnxruntime 依赖）；`pip install -r requirements.txt`；`ARG HF_ENDPOINT=https://hf-mirror.com`（构建期可换源）后用一次性 Python 调用把模型预取到 `/app/models`（`ENV KB_MODEL_DIR=/app/models`），chown 给 appuser。运行时不依赖该构建参数。
- 配置（`config.py` + `.env.example`，均非密钥）：`KB_EMBEDDING_BACKEND`、`KB_EMBEDDING_MODEL`、`KB_MODEL_DIR`、`KB_CHUNK_CHARS=400`、`KB_CHUNK_OVERLAP=80`、`KB_TOP_K=8`、`KB_QUERY_MAX_LEN=200`。

### 6. 切分、索引事务与生命周期

- 切分（`chunking.py`，纯函数易测）：按段落累积到约 400 字符（中文按字符计）、相邻块 80 字符重叠；逐段跟踪最近的 Markdown/文本标题行作为 `heading`；记录 `char_start` 与 `line_no`。空文件/纯空白文件不产块（上传仍成功，表现为"该材料无检索结果"）。
- `indexer.index_material(material)`：读盘 UTF-8 文本 → 切分 → 批量嵌入 → **单事务替换**：删该 material 的旧 `kb_vectors`、`kb_chunks`，插入新块与向量，置 `indexed_at=now`，提交；中途异常回滚，不留半套块，并向上抛出由调用方记日志。重建语义保证 reindex/seed 重复执行不产生重复数据。
- 上传调用点捕获异常仅写日志（材料已合法存在，索引可由 CLI 修复，满足 spec 的失败场景）；`flask reindex-kb [--all] [--material-id ID]` 默认处理"扩展名为 txt/md 且 `indexed_at IS NULL`"的材料，`--all` 全量替换重建；输出处理条数。
- seed：A/B 班各写一份内容显著不同的中文 md/txt（如 A 班"光合作用"讲义、B 班"牛顿第二定律"讲义），随后对这些材料走同一 `index_material`（替换语义，配合 seed 幂等）。

### 7. 检索编排与班级隔离的落点

- `GET /kb`（HTML 页，表单 method=get，字段 `q`）：`login_required()` 未登录 302 到登录页；空/纯空白/超过 200 字符不执行检索，页面给出内联提示。
- `service.search(user, q)`：stub/真实后端嵌入查询 → `vector_store` 对**全库**做 KNN（课堂规模每年级几千到几万块，单次向量比对毫秒级，不在 vec 侧加 LIMIT）→ 与 `kb_chunks` JOIN 后 **WHERE class_id = g.user.class_id**（过滤只存在于 `kb/repository.py`，与材料侧同样纪律：业务代码无法构造不带班级的片段查询）→ ORDER BY distance LIMIT `KB_TOP_K`。
- 之所以先全库 KNN 再过滤：`vec0` 虚表不支持任意元数据过滤；内容字段只在班级过滤后的 JOIN 中读取，跨班只参与距离计算、绝不返回任何文本。班级数量与数据量增长后可改为"每班虚表/候选池截断"，已在风险中标注。
- 结果对象：`material_name`、`snippet`（块原文，模板自动转义）、`heading`、`line_no`、`distance/score`、溯源 URL（md→`/materials/<id>/preview#<slug>`，txt→`.../preview#L<line>`，均为经鉴权的既有路由）。

### 8. 依赖、镜像与运行形态

- requirements 新增：`Markdown>=3.6,<4`、`bleach>=6,<7`、`fastembed`（固定近期稳定版本并在构建期验证模型可取）、`sqlite-vec>=0.1,<1`；numpy/onnxruntime/tokenizers 由 fastembed 带入。
- docker-entrypoint 维持 `init-db → seed → gunicorn`，无需改顺序（init-db 完成虚表建表，seed 完成文本材料与索引）。gunicorn 保持 2 workers：模型按 worker 惰性各载一份（额外约 200–400MB RSS），课堂机足够；worker 数通过入口脚本中既有常量保持，不新增编排服务。
- 本地开发（venv）首次使用真实后端时模型下载到 `KB_MODEL_DIR`（默认数据目录下 `models/`）；测试一律 stub，CI/课堂断网无影响。
- 宿主机磁盘：Docker 虚拟磁盘（镜像、模型层、命名卷）需位于空间充足分区；部署说明提示在构建前于 Docker Desktop 设置中将 Disk image location 迁到目标分区（如 `D:\1-file\DockerContent`）。

### 9. 测试策略（pytest + test client，沿用 tmp_path SQLite/上传目录）

- conftest 注入 `KB_EMBEDDING_BACKEND=stub`；应用工厂的 `ensure_runtime_schema` 在测试库上自动建虚表与新列。
- 新增/扩展用例：
  1. 上传：txt/md 合法成功；伪造扩展名的非 UTF-8 文本 400 且无文件无记录；md 声明 text/plain 也被接受；
  2. 预览：txt 转义显示 `<script>` 文本；md 标题列表被渲染、原文 `<script>`/`onerror`/`javascript:` 不出现在输出的可执行形态；学生可预览本班；跨班 404；非文本 415；物理文件缺失 500；
  3. 上传页清单含全部允许扩展名且标注 txt/md 可预览；
  4. 知识库：文本上传后立即语义命中；非文本无块；查询 B 班独有内容 A 班零命中（含文件名不泄露）；未登录 302/401；空查询与超长查询被拒；结果含来源名、片段、锚点链接且按相关度排序、数量受 top_k 限制；
  5. 溯源链接 200 且跨班访问该链接 404；
  6. 持久化：同一数据库文件重建 app 后索引仍在；
  7. `reindex-kb` 连跑两次块数不变；旧/失败材料（monkeypatch 令嵌入抛错）上传仍 201、`indexed_at` 为空，CLI 后恢复可检索；
  8. seed 幂等：连跑两次材料数与块数不变。

## Risks / Trade-offs

- [构建期模型下载受网络影响] → 构建参数 `HF_ENDPOINT` 默认指向国内可访问镜像；失败可重试或手工放置模型缓存；运行时 local_files_only，故障模式明确。
- [全库 KNN 后再按班过滤在超大规模下有开销，且依赖"先过滤后取内容"纪律] → 课堂单年级规模足够；过滤 SQL 集中在 kb/repository 并有跨班测试锁死；日后量级增长再改为按班分区虚表。
- [Markdown XSS 变种（事件属性、伪协议、编码绕过）] → bleach 白名单 + 协议限制 + 安全测试用例；原文只经转义/消毒管道输出，禁止直接 `|safe` 原始内容。
- [旧数据卷 schema 升级] → 启动期与 init-db 的幂等 `ALTER TABLE`/建虚表；SQLite ADD COLUMN 对可空列是在线元数据操作；回滚时旧代码忽略新表新列，无破坏。
- [gunicorn 多 worker 各载一份模型，内存上升] → 2 workers 可接受；模型惰性加载；如课堂机内存紧张可把 worker 降为 1（纯文档化操作，不改架构）。
- [.md 声明 MIME 五花八门导致合法文件被拒] → md 白名单兼容 text/plain；扩展名仍为第一判据，测试覆盖三种声明。
- [20MB 文本切分/嵌入耗时拖长上传请求] → 单材料块数有限，fastembed 批量嵌入为本地计算，秒级可接受；失败与材料解耦并可重建；不为此引入任务队列。

## Migration Plan

1. 合并代码并在 venv `pip install -r requirements.txt`，stub 后端下跑全量 pytest。
2. 宿主机先在 Docker Desktop 将 Disk image location 迁到空间充足分区（用户侧一次性操作）。
3. 重新构建镜像（构建期完成模型预热）→ `docker compose up`：entrypoint 自动完成新列/虚表升级、种子文本材料入库。
4. 验收：未登录 `/kb` 跳登录；A 班账号检索 A 班预置内容命中、检索 B 班独有内容零命中；上传 md 后即时可预览、可检索；溯源链接可定位；容器重建后索引仍在。
5. 回滚：代码回退到旧版本镜像即可（新表/新列被旧代码忽略）；如需彻底回退，备份后删除命名卷重建种子数据。

## Open Questions

无。以下细节按合理假设固化：切块约 400 字符/重叠 80 字符；top_k=8；查询上限 200 字符；预览白名单不含 img/iframe；模型固定 `BAAI/bge-small-zh-v1.5`（512 维）；gunicorn 维持 2 workers；非文本与空文本材料不进索引但上传合法。
