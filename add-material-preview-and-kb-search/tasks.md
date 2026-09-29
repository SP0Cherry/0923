# Tasks

## 1. 依赖与配置

- [ ] 1.1 在 requirements.txt 增加 `Markdown>=3.6,<4`、`bleach>=6,<7`、`fastembed`（固定近期稳定版本）、`sqlite-vec>=0.1,<1`，并在 venv 执行 `pip install -r requirements.txt` 验证安装成功（numpy/onnxruntime 由 fastembed 带入）
- [ ] 1.2 更新 `app/config.py`：默认允许扩展名追加 txt/md，MIME 双白名单增加 `txt:{text/plain}`、`md:{text/markdown,text/x-markdown,text/plain}`，新增 `PREVIEWABLE_EXTENSIONS={"txt","md"}` 与 KB 配置（`KB_EMBEDDING_BACKEND/KB_EMBEDDING_MODEL/KB_MODEL_DIR/KB_CHUNK_CHARS=400/KB_CHUNK_OVERLAP=80/KB_TOP_K=8/KB_QUERY_MAX_LEN=200`）；用加载配置的单元测试断言 txt/md 在校验集合内、默认值齐全
- [ ] 1.3 更新 `.env.example`，补充 KB 相关环境变量说明（均为非密钥项）与 `KB_MODEL_DIR` 容器路径示例；确认无真实密钥写入

## 2. 数据模型与 SQLite schema 升级

- [ ] 2.1 在 `app/models.py` 的 `Material` 增加可空列 `indexed_at`，并新增 `KbChunk` 模型（id、material_id、class_id 带索引、chunk_index、char_start、line_no、heading、content、created_at，`UNIQUE(material_id, chunk_index)` 及 material/class 关系）；用 `db.create_all()` 建表测试验证表结构
- [ ] 2.2 在应用工厂的 sqlite 连接事件中加载 sqlite-vec 扩展（与既有 PRAGMA 设置同处，`sqlite_vec.load(dbapi_connection)`）；编写连接加载测试，在测试 SQLite 上执行 `SELECT vec_version()`（或等价虚表建表语句）验证扩展可用
- [ ] 2.3 新增幂等 `ensure_runtime_schema()`（放在 `app/kb/vector_store.py` 或独立 schema 辅助模块）：`PRAGMA table_info(materials)` 缺列时 `ALTER TABLE ... ADD COLUMN indexed_at`，并 `CREATE VIRTUAL TABLE IF NOT EXISTS kb_vectors USING vec0(chunk_id INTEGER PRIMARY KEY, embedding FLOAT[512])`；在 `create_app` 初始化段与 `flask init-db` 中调用；测试在全新库与模拟旧库（仅有旧 materials 表）上各跑一次，断言列/虚表存在且重复执行不报错

## 3. txt/md 上传校验改造

- [ ] 3.1 改造上传视图：txt/md 先读入字节并按 UTF-8 严格解码（解码失败返回 400 编码错误，不触盘不写库），随后复用 `storage.write_bytes` 落盘与 `repository.create_material` 写库；其他类型保持原流式路径不变；测试覆盖：合法 txt/md 成功、非 UTF-8 的 txt/md 返回 400 且无文件无记录、md 声明 text/plain 被接受
- [ ] 3.2 为模板注入服务端配置派生的格式清单（context processor 或视图变量：全部允许扩展名 + 可预览扩展名集合），更新 `templates/materials/list.html` 上传表单：文件选择 accept 与说明文字展示完整清单并标注 txt/md 支持在线预览；测试断言上传页包含全部扩展名与预览说明

## 4. 文本材料在线预览

- [ ] 4.1 在 `app/storage.py` 或 materials 模块新增受班级边界约束的文本读取辅助（读取字节、UTF-8 解码、物理文件缺失判定）；新增 `GET /materials/<id>/preview` 路由：`get_scoped_or_404` 鉴权 → 非 txt/md `abort(415)` → 文件缺失 500；测试覆盖跨班 404、非文本 415、文件缺失 500
- [ ] 4.2 实现 txt 预览模板（`<pre>` 自动转义，按行包裹 `<span id="L{n}">` 锚点）；测试验证尖括号/脚本片段以纯文本展示、换行保留、未登录访问 302
- [ ] 4.3 实现 md 安全渲染管道：`markdown.markdown(extensions=["fenced_code","tables","toc"])` 后 `bleach.clean` 白名单消毒（仅排版标签、协议限制 http/https），新建 `templates/materials/preview.html`；测试验证标题/列表渲染为 HTML，且 `<script>`、`onerror`、`javascript:` 等输入在响应中不构成可执行内容
- [ ] 4.4 更新材料列表与详情页：仅对 txt/md 材料显示"预览"链接（学生与教师视图均显示）；测试验证学生可见并可打开预览、非文本材料行无预览入口

## 5. 知识库核心：切分、嵌入、向量存储、索引

- [ ] 5.1 编写 `app/kb/chunking.py` 纯函数：UTF-8 文本按约 400 字符切块、80 字符重叠，输出每块的文本、char_start、line_no 与最近标题 heading；md 标题提取与预览共用 `markdown.extensions.toc.slugify` 生成锚点 slug；单元测试覆盖短文本单块、长文本多块不丢字、重叠正确、标题归属与空文本产零块
- [ ] 5.2 编写 `app/kb/embeddings.py`：`Embedder` 抽象 + `StubEmbedder`（确定性分词哈希、归一化、共享词条的 query/文本相似度更高）+ `FastEmbedEmbedder`（BAAI/bge-small-zh-v1.5，cache_dir 取 `KB_MODEL_DIR`，运行时 local_files_only）；提供按 config 取惰性单例的工厂；stub 单测验证同维输出、归一化与可重复结果，且 stub 路径全程不访问网络
- [ ] 5.3 完善 `app/kb/vector_store.py`：kb_vectors 的插入（块 id + 向量序列化）、按 material_id 删除、查询向量 KNN（返回 chunk_id/distance，全库扫描不在 vec 侧截断）；用测试库验证写入→KNN 命中自身→删除后不命中的往返
- [ ] 5.4 编写 `app/kb/repository.py`：`replace_chunks(class_id, material_id, chunks, vectors)` 单事务替换（先删旧块与向量再插入）、`set_indexed_at`、`search_scoped(user_class_id, candidate_rows, limit)` JOIN kb_chunks 强制带 class_id 过滤并按 distance 排序；测试断言所有查询路径都带班级条件、替换幂等（同材料替换两次行数不变）
- [ ] 5.5 编写 `app/kb/indexer.py`：`index_material(material)` 读盘→切块→批量嵌入→事务替换→置 indexed_at；异常时回滚不留半套块并抛出；上传视图在材料提交成功后调用并捕获异常仅记日志（材料保留、indexed_at 留空）；测试验证：上传 txt/md 后块与 indexed_at 立即可用、非文本无块、嵌入失败时上传仍 201 且无半成品块

## 6. 检索服务、路由与页面

- [ ] 6.1 编写 `app/kb/service.py`：`search(user, q)` 输入校验（非空、非纯空白、长度 ≤200，非法返回明确错误）→ 嵌入查询 → KNN → 本班 repository 过滤 → 组装结果（来源文件名、snippet、heading、line_no/锚点 URL、score），数量受 KB_TOP_K 限制；用 stub 后端测试语义命中、相关度排序与 top_k 截断
- [ ] 6.2 新增 `app/kb` Blueprint 与 `GET /kb` 检索页（空态展示搜索框，带 q 时渲染结果），在应用工厂注册；`login_required()` 保证未登录 302、程序化 401；测试覆盖未登录重定向/401、空查询与超长查询不执行检索且有提示
- [ ] 6.3 编写 `templates/kb/search.html`：搜索表单（GET）、结果列表（文件名、片段、小节/行号、相关度、溯源链接指向材料预览锚点）；在 `base.html` 导航增加"知识库检索"入口（登录后可见）；测试验证结果页含来源名与片段、溯源链接 URL 合法且跨班访问该链接为 404
- [ ] 6.4 跨班隔离集成测试：A/B 班分别上传含独特内容的文本材料，A 班用户检索 B 班独特内容零命中且响应中不出现 B 班任何文本/文件名，同班教师与学生结果集合一致

## 7. CLI 与种子数据

- [ ] 7.1 在 `app/cli.py` 新增幂等 `flask reindex-kb [--all] [--material-id ID]`：默认处理 txt/md 且 indexed_at 为空的材料，--all 全量替换重建；输出处理统计；测试验证回填缺失索引、--all 连跑两次块数不变、--material-id 只重建指定材料
- [ ] 7.2 更新 `app/seed.py`：A/B 班各增加一份内容显著不同的中文文本示例材料（一份 md 一份 txt），走 `index_material` 入库；保持 get-or-create 幂等；测试验证首次 seed 后两班材料可分别检索到本班内容，连跑两次材料数与块数不变，A 班账号检索 B 班独特内容零命中

## 8. 持久化与回归测试

- [ ] 8.1 持久化测试：对同一 SQLite 文件与上传目录销毁并重建 app（模拟容器重启），断言已索引材料仍可检索、indexed_at 保留、虚表数据存在
- [ ] 8.2 在 stub 后端下运行全量 pytest（含既有 auth/access-control/materials/foundations/security 用例），确认全部通过且无测试触发模型下载或外网请求

## 9. 容器化与端到端验收

- [ ] 9.1 更新 Dockerfile：安装 libgomp1，`ARG HF_ENDPOINT=https://hf-mirror.com`，构建期一次性调用将 BGE 模型预取到 `/app/models`（`ENV KB_MODEL_DIR=/app/models`），确认 appuser 对模型目录有读权限；执行 `docker compose build` 验证镜像构建成功且模型位于镜像内
- [ ] 9.2 确认 docker-entrypoint 的 `init-db → seed → gunicorn` 顺序下虚表自动创建、种子文本自动索引；`docker compose up` 启动后在无额外 API 密钥环境下验证 `/health` 200
- [ ] 9.3 浏览器端到端验收：教师上传 md（含标题/列表与一段脚本字符串）→ 列表出现并可预览（脚本不执行）→ 学生用自然语言在 `/kb` 检索命中该材料 → 点击溯源链接定位到预览锚点 → 换 B 班账号检索相同内容零命中；记录验收结果
- [ ] 9.4 运行 `openspec validate add-material-preview-and-kb-search --strict` 通过，并更新部署说明：构建前须将 Docker Desktop 的 Disk image location 迁至空间充足分区（如 D:\1-file\DockerContent）
