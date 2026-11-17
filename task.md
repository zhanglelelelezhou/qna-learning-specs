# 问答式学习 Tasks

- 版本：v0.1
- 日期：2026-11-16
- 状态：2026-11-17 用户授权进入开发；T1 完成，T2–T22 未开始
- 需求基线：已批准 Spec v0.2
- 设计基线：已批准 Plan v0.1
- 本阶段交付：22 个有依赖、有具体文件、有操作步骤和验证方式的任务。
- 接续依据：四份文档已存在；用户于 2026-11-17 明确指示读取并从 T1 开始开发。按现有范围逐项实施和验证。

## 1. 执行与验证约定

所有路径相对于项目根目录 qna-learning/。服务端命令在 backend/ 执行；小程序命令在 miniprogram/ 执行。开发时在项目目录内创建独立 Python 虚拟环境，并将依赖锁定；下面的 python 指该虚拟环境的解释器。

### 1.1 固定验证入口

| 范围 | 实际命令或操作 | 判定规则 |
|---|---|---|
| 后端全部自动化 | python -m unittest discover -s tests -v | 记录实际运行、失败、错误、跳过数量；跳过的真实服务检查不能算通过。 |
| 后端单模块 | python -m unittest tests.test_auth -v 等各任务列出的完整模块名 | 对应行为断言满足；不能仅凭成功退出认定真实外部服务已接通。 |
| 后端语法检查 | python -m compileall -q app tests | 无语法错误；此检查不能代替行为测试。 |
| 后端启动 | python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --workers 1 | GET http://127.0.0.1:8000/api/v1/health 返回实际就绪状态；T1 验证进程可启动，T2 后同时核对存储就绪。 |
| 小程序类型检查 | npm run typecheck | 应用与测试类型检查通过。 |
| 小程序逻辑测试 | npm test | 真实运行 TypeScript 测试，核对录音、播放、页面和任务行为；不能冒充微信环境。 |
| 小程序构建 | npm run build | TypeScript 编译并复制 WXML/WXSS/JSON，生成 .build/ 可导入目录；不包含测试文件、密钥或用户数据。 |
| 真实服务检查 | RUN_LIVE_TESTS=1 python -m unittest tests.test_providers.LiveProviderTests -v | 只有凭据、服务和网络具备且实际运行成功才记录通过；没有条件则记录未验证。 |
| 单次清理 | python -m app.tasks.cleanup --once | 只处理已确认删除或已到期的对象，返回分类计数与时间，不输出正文和凭据。 |
| 微信环境 | 用开发者工具打开小程序项目并在指定 Android/iPhone 上体验 | 记录真实设备、版本及操作证据；没有实际观察就记未验证。 |

上述 npm 脚本、Python 测试和清理入口将在对应任务中建立，本文件不声称它们已经存在。不存在独立的额外 lint 门槛；后续若配置静态检查工具，必须写明真实命令再执行，不声称没有配置的 lint 已通过。

### 1.2 工作规则

- 每个任务先完成步骤，再运行本任务验证；保存命令、输出摘要和实际结果后才标记完成。
- 自动化可以使用测试进程内的受控微信／模型适配器验证分支；生产不提供测试登录和演示回答。
- 对密钥、真实微信账号、设备等缺失条件不作猜测。先完成不受影响的工作，在验证记录中写出缺少条件及补验步骤；需要用户参与时给出具体配置或操作。
- 不增加公众发布、付费、连续通话、OCR、Word 导入或 Web 端。
- 不把“源码已写好”“构建通过”“模拟接口通过”写成真实语音或真机验收通过。
- 原文档编制阶段不编写实现代码；现已进入开发，状态只在实际验证后更新。

## 2. 文件清单

表内同一个文件可能由多个相关任务依次补充。Python 包目录按需要添加 __init__.py，保证测试命令的包导入一致。

| 操作 | 文件 | 归属任务／职责 |
|---|---|---|
| 新建 | README.md、.gitignore、.env.example | T1、T19、T22：启动、限制、配置和交付；忽略真实环境、数据、缓存和生成产物。 |
| 新建 | Dockerfile、compose.yaml | T19：单服务进程、持久卷、128 MiB 以上 tmpfs、ffmpeg 和健康检查。 |
| 更新 | docs/v1/plan.md | 本阶段仅记录用户批准状态，不改变已批准设计。 |
| 新建／后续维护 | docs/v1/task.md、docs/v1/checklist.md | 本文件在当前阶段交付；Checklist 在下阶段交付；开发后仅据事实更新状态及证据。 |
| 新建 | docs/setup-wechat.md、docs/privacy-notice.md、docs/verification.md | T3、T19–T22：配置、处理用途、按环境区分的真实验证记录。 |
| 新建 | backend/pyproject.toml、backend/requirements.lock | T1：声明并锁定运行／测试依赖。 |
| 新建 | backend/app/__init__.py、backend/tests/__init__.py | T1：确定模块入口。 |
| 新建 | backend/app/main.py、backend/app/config.py、backend/app/contracts.py、backend/app/errors.py | T1–T13：入口、共享限制、Plan 契约、错误码和装配。 |
| 新建 | backend/app/storage/database.py、backend/app/storage/repositories.py、backend/app/storage/files.py | T2：事务、用户范围查询和私有文件；后续任务补充领域查询。 |
| 新建 | backend/app/storage/ephemeral_audio.py | T11：受限临时录音池、tmpfs 和清理。 |
| 新建 | backend/app/migrations/001_initial.sql | T2：Plan 核心表、外键、唯一约束和迁移版本。 |
| 新建 | backend/app/api/auth.py | T3：登录、退出、隐私及确认。 |
| 新建 | backend/app/api/materials.py | T5、T7、T13：资料导入、预览、目录、进度和删除入口。 |
| 新建 | backend/app/api/sessions.py | T8–T11：初学、复习、历史、恢复、文字和语音提交。 |
| 新建 | backend/app/api/jobs.py | T4：任务状态、取消和重试。 |
| 新建 | backend/app/api/speech.py | T12：合成、片段状态、鉴权下载和删除状态查询。 |
| 新建 | backend/app/services/identity.py、backend/app/providers/wechat.py | T3：微信身份交换、摘要令牌和隐私授权。 |
| 新建 | backend/app/services/materials.py | T5、T7：导入、解析预览、目录与索引协调。 |
| 新建 | backend/app/services/learning.py | T8–T10：教学动作、公共视图、恢复和事务提交。 |
| 新建 | backend/app/services/voice.py | T11–T12：转写后教学链及独立合成。 |
| 新建 | backend/app/services/deletion.py | T13：确认删除、即时封锁和关联清理。 |
| 新建 | backend/app/learning/engine.py、backend/app/learning/intents.py | T8–T9：纯状态转换和当前阶段允许的意图。 |
| 新建 | backend/app/learning/context.py、backend/app/learning/prompts.py、backend/app/learning/citations.py | T6–T9：来源上下文、结构化任务提示及引用验证。 |
| 新建 | backend/app/documents/parser.py、backend/app/documents/parser_worker.py、backend/app/documents/chunking.py | T5：逐页／段落提取、子进程超时和块映射。 |
| 新建 | backend/app/documents/retrieval.py | T7：当前资料范围内的向量精确排序。 |
| 新建 | backend/app/providers/base.py、backend/app/providers/qwen_text.py、backend/app/providers/qwen_embedding.py | T6：外部文本／向量契约、真实请求及统一失败处理。 |
| 新建 | backend/app/providers/qwen_asr.py、backend/app/providers/qwen_tts.py | T11–T12：Base64 转写和分段合成适配。 |
| 新建 | backend/app/tasks/scheduler.py、backend/app/tasks/lifecycle.py、backend/app/tasks/cleanup.py | T4、T11–T13：持久任务、截止时间、重启恢复、周期与手动清理。 |
| 新建 | backend/tests/test_api.py、backend/tests/test_storage.py、backend/tests/support.py | T1、T2、T10：健康检查、存储行为、隔离测试应用和可控时钟。 |
| 新建 | backend/tests/test_auth.py、backend/tests/test_jobs.py、backend/tests/test_documents.py | T3–T5：授权、任务边界、资料正常与错误输入。 |
| 新建 | backend/tests/test_providers.py、backend/tests/test_retrieval.py、backend/tests/test_learning.py | T6–T9、T20：供应商契约、来源检索、教学行为和显式启用的真实服务检查。 |
| 新建 | backend/tests/test_voice.py、backend/tests/test_deletion.py | T11–T13：实际转码、转写状态、分段资产和删除生命周期。 |
| 新建 | backend/tests/fixtures/source.txt、backend/tests/fixtures/source.md、backend/tests/fixtures/known-facts.json、backend/tests/fixtures/injection.txt、backend/tests/fixtures/make_fixtures.py | T5：三主题中文样例、明确事实、资料注入内容及可重复 PDF／音频样例。 |
| 新建 | miniprogram/app.ts、miniprogram/app.json、miniprogram/app.wxss、miniprogram/project.config.json | T1、T14–T18：入口、页面注册、统一阅读样式、微信导入配置。 |
| 新建 | miniprogram/package.json、miniprogram/package-lock.json、miniprogram/tsconfig.json、miniprogram/tsconfig.test.json、miniprogram/scripts/build.mjs | T1：类型检查、TypeScript 测试和可重复构建。新增辅助文件仅细化 Plan 的构建验证入口。 |
| 新建 | miniprogram/types/contracts.ts | T1、T14：和服务端一致的公共类型、配置及 snake_case 映射。 |
| 新建 | miniprogram/services/api.ts、miniprogram/services/auth.ts、miniprogram/services/materials.ts、miniprogram/services/sessions.ts、miniprogram/services/jobs.ts | T14：请求、上传、鉴权、轮询、重试和业务 API。 |
| 新建 | miniprogram/services/recorder.ts、miniprogram/services/playback.ts | T17–T18：录音和分段播放封装。 |
| 新建 | miniprogram/state/learning.ts、miniprogram/state/voice.ts | T14、T16–T18：会话状态、录音状态和迟到结果隔离。 |
| 新建 | miniprogram/pages/library/index.ts、miniprogram/pages/library/index.wxml、miniprogram/pages/library/index.wxss、miniprogram/pages/library/index.json | T15、T19：资料列表、状态、继续和删除。 |
| 新建 | miniprogram/pages/import/index.ts、miniprogram/pages/import/index.wxml、miniprogram/pages/import/index.wxss、miniprogram/pages/import/index.json | T15：导入与隐私确认。 |
| 新建 | miniprogram/pages/preview/index.ts、miniprogram/pages/preview/index.wxml、miniprogram/pages/preview/index.wxss、miniprogram/pages/preview/index.json | T15：原文预览、警告和生成目录。 |
| 新建 | miniprogram/pages/outline/index.ts、miniprogram/pages/outline/index.wxml、miniprogram/pages/outline/index.wxss、miniprogram/pages/outline/index.json | T15：章节、学习起点、复习角色和范围。 |
| 新建 | miniprogram/pages/study/index.ts、miniprogram/pages/study/index.wxml、miniprogram/pages/study/index.wxss、miniprogram/pages/study/index.json | T16–T18：两种模式、文字、来源、录音和播报。 |
| 新建 | miniprogram/pages/source/index.ts、miniprogram/pages/source/index.wxml、miniprogram/pages/source/index.wxss、miniprogram/pages/source/index.json | T16：原文定位。 |
| 新建 | miniprogram/pages/history/index.ts、miniprogram/pages/history/index.wxml、miniprogram/pages/history/index.wxss、miniprogram/pages/history/index.json | T16：历史、总结和恢复。 |
| 新建 | miniprogram/tests/jobs.test.ts、miniprogram/tests/learning.test.ts、miniprogram/tests/playback.test.ts、miniprogram/tests/recorder.test.ts、miniprogram/tests/support.ts | T14–T18：模拟微信能力与假时钟验证；仅在测试编译和测试运行中出现。 |
| 生成、不入源码 | miniprogram/.build/、backend/tests/fixtures/generated/ | 编译与生成样例，不含真实用户资料；可重复生成。 |
| 交付时生成 | 项目目录外的 deliverables/qna-learning-source.zip | T22：完整源码、四份审批文档和运行说明；排除密钥、用户资料、依赖目录与内部辅助文件。 |

## 3. 有序任务

### T1：建立能启动和构建的项目基础

**交付结果：** 后端健康入口可运行，小程序 TypeScript 有明确构建入口，公共契约和限制有唯一来源。

**文件：** README.md、.gitignore、.env.example；backend/pyproject.toml、backend/requirements.lock；backend/app/__init__.py、main.py、config.py、contracts.py、errors.py；backend/tests/__init__.py、test_api.py、support.py；miniprogram/app.ts、app.json、app.wxss、project.config.json、package.json、package-lock.json、tsconfig.json、tsconfig.test.json、scripts/build.mjs、types/contracts.ts。

**依赖：** 无。

**步骤：**
1. 声明 Python 3.12 运行依赖并创建虚拟环境，锁定实际安装版本；建立 FastAPI 应用工厂和 /api/v1/health。
2. 把 20 MiB、200,000／100 字符、4,000 字符、120 秒录音、120／90／60 秒等待和 5 题／1 次追问定义为服务端共享规则。
3. 按 Plan 第 2–4 节建立 Pydantic 数据契约、公共视图、错误码和外部适配器边界；客户端只定义可公开字段，不复制私有答案结构。
   按 Plan 第 8 节，在会话公开响应中附带可读的限制配置，供前端显示长度、录音与等待上限；服务端仍独立强制校验，客户端不能放宽。
4. 建立 Node 构建脚本：编译应用 TypeScript，复制所需 JSON、WXML、WXSS 到 .build/；测试和服务端文件不进入产物。测试运行使用独立配置。
5. 添加健康检查和缺少配置行为验证；配置模板只写变量名和空值，不含真实秘密。此时不伪造后续业务能力。

**验证：** 在 backend/ 运行 python -m unittest tests.test_api.SmokeTests -v；启动单进程服务并访问 /api/v1/health，确认健康响应可观察。在 miniprogram/ 运行 npm run typecheck 和 npm run build，确认构建产物没有后端配置、测试或真实凭据。

**T1 实际记录（2026-11-17）：** 已完成。后端 7 项 SmokeTests、8 项 ContractTests 全部通过；实际单 worker 服务健康 HTTP 200，storage 明确为 not_initialized。小程序类型检查、构建及 1 项启动测试通过。证据见 [验证记录](../verification.md)。

**T1 文件细化：** 为生成可注册页面，建立 pages/library/index 的 TS/JSON/WXML/WXSS 基础壳，业务仍属 T15；适配器 Protocol 位于 app/providers/base.py，真实实现保持原任务归属；tests/contracts.test.ts 验证公开类型和不依赖外部能力的启动。没有新增产品功能。

### T2：建立按用户隔离的持久存储

**交付结果：** 领域对象可事务保存、重开连接后可读，不能越过所有权访问或泄露私有路径。

**文件：** backend/app/storage/database.py、repositories.py、files.py；backend/app/migrations/001_initial.sql；backend/app/config.py；backend/tests/test_storage.py、support.py。

**依赖：** T1。

**步骤：**
1. 创建 Plan 列出的用户、令牌、资料、原文块、向量、目录、小节、讲解计划、会话、进度、消息、题目、回答、任务、在途名额、音频、用量及删除表。
2. 配置外键、唯一键、迁移版本、busy_timeout、secure_delete 和默认回滚日志；用 BEGIN IMMEDIATE 实现短写事务，不把网络调用放在事务中。
3. 建立接收 UserContext 的资源查询和写入入口；客户端不能指定有效 owner_id。对不存在和其他用户对象统一报 NOT_FOUND。
4. 私有文件路径只用服务端 ID 生成；防止原始文件名带路径影响存储。持久资料目录与临时音频目录分开。
5. 用甲乙两用户和一次事务中断验证隔离、重开连接及原子回滚。

**验证：** 运行 python -m unittest tests.test_storage -v，观察甲可读取自己的资料、乙读取同 ID 被拒绝；中途异常无半条进度或消息；重开连接仍保留已提交对象；路径穿越样例不能写出私有目录。

### T3：接通微信身份契约和隐私前置确认

**交付结果：** 没有合法身份或隐私确认时不能处理资料；登录退出、令牌过期和账号隔离有明确行为。

**文件：** backend/app/providers/wechat.py；backend/app/services/identity.py；backend/app/api/auth.py；backend/app/main.py、errors.py；backend/tests/test_auth.py；docs/privacy-notice.md。

**依赖：** T2。

**步骤：**
1. 实现 IdentityService.login，以 wx.login 的 code 经服务器交换身份；只信供应商返回身份，不信客户端 openid。
2. 随机令牌保存摘要、7 天期限及撤销状态；提供登录、退出和身份依赖。
3. 实现 /privacy 和 /consents，记录版本与同意时间；未同意时禁止上传、粘贴、转写和模型调用。
4. 隐私说明列明资料／语音发送对象、用途、录音临时处理、删除范围及供应商保留边界，不写无法兑现的保证。
5. 测试通过 httpx 受控微信响应及隔离应用注入身份；生产装配不开放测试登录或回退身份。

**验证：** 运行 python -m unittest tests.test_auth -v。未登录、过期、退出后、拒绝隐私四类请求被拒绝；未同意时外部内容调用次数为零；用户乙不能获取用户甲的对象；输出无微信 session_key。真实微信登录在 T21 真机中另验。

### T4：建立可取消、可恢复且不重复执行的任务入口

**交付结果：** 长任务可以观察状态、取消、超时及显式重试，重复点击不会重复提交结果。

**文件：** backend/app/tasks/scheduler.py、lifecycle.py；backend/app/api/jobs.py；backend/app/storage/repositories.py；backend/app/main.py、contracts.py；backend/tests/test_jobs.py、support.py。

**依赖：** T3。

**步骤：**
1. 实现 JobService.create、cancel_chain、commit_result；保存 Idempotency-Key 与请求摘要，维护 material／learning／asr 在途名额。
2. 生命周期调度器按任务种类执行已注册处理器；提供待处理、运行、成功、失败、取消、过期公开视图，未实现的种类不能返回假成功。
3. 实现共享截止时间、可控时钟和每秒过期扫描；排队、外部调用和一次结构修复共用总预算。
4. 结果提交前重新核对任务状态、资料删除标记、revision、截止时间和 lease，原子保存回复与进度。
5. 重启后标记结果不确定的调用为 INTERRUPTED，不自动重复收费；明确失败后才建立重试任务。语音链的临时引用恢复由 T11 补齐。
6. 测试重复 key、冲突 key、取消后迟到、90 秒超时、旧 revision、重启和在途限制。

**验证：** 运行 python -m unittest tests.test_jobs -v；通过实际 /jobs 接口看到同 key 对应同一任务，冲突返回 409；取消／过期后的处理结果不能改变消息与进度；重启不重复执行不确定的外部请求。

### T5：完成资料导入、解析和可定位预览

**交付结果：** 四种已批准输入都能进入文字预览，错误资料有明确原因，PDF 来源页码和文本段落可追溯。

**文件：** backend/app/documents/parser.py、parser_worker.py、chunking.py；backend/app/services/materials.py；backend/app/api/materials.py；backend/tests/test_documents.py；backend/tests/fixtures/source.txt、source.md、known-facts.json、injection.txt、make_fixtures.py；backend/app/storage/files.py。

**依赖：** T4。

**步骤：**
1. 实现文件导入和粘贴；服务端流式限制 20 MiB，严格校验 UTF-8、文件类型、正文 100 个非空白字符及 200,000 字符上限。
2. PDF 放到可终止子进程逐页提取，120 秒内结束；保存页码、段落和稳定原文块 ID。超时或损坏时保留资料错误状态，不生成可学习假正文。
3. 检测扫描／混合 PDF 并给出失败或图片未纳入警告；不调用 OCR。按标题和段落提取 TXT／Markdown／粘贴正文。
4. 返回分页原文预览，不在预览前调用模型。支持放弃、重新导入和删除入口。
5. 建立约 1,000 字符的检索块及 100 字符重叠，并保留完整原文 span。
6. 生成可重复的小型文本／扫描／混合／加密／损坏 PDF 和后续音频样例；用三个明确主题的中文资料及已知事实作验收基准。

**验证：** 运行 python -m unittest tests.test_documents -v。覆盖 20 MiB 边界、200,000／200,001 字符、99／100 非空白字符、UTF-8 解码错误；打开 PDF 来源返回正确页码，扫描件明确失败，混合件有警告；超时子进程结束且没有半份成功预览。

### T6：实现文本和向量服务适配及结构化失败处理

**交付结果：** 真实服务请求有可替换边界，缺配置或不合法输出明确失败，不出现演示回答。

**文件：** backend/app/providers/base.py、qwen_text.py、qwen_embedding.py；backend/app/learning/prompts.py；backend/app/config.py、contracts.py、errors.py；backend/tests/test_providers.py。

**依赖：** T4。

**步骤：**
1. 实现 ModelProvider.generate 和 EmbeddingProvider.embed；读取业务空间地址与服务器凭据，默认 qwen-plus 非思考及 text-embedding-v4。
2. 分别定义目录、讲解计划、意图、教学、复述反馈、复习计划、反馈和总结的 JSON 输出契约；未知字段或类型错误不得直接进入教学状态。
3. 在同一总截止时间内最多允许一次结构修复；429、额度不足、鉴权、网络错误、超时和无效响应使用统一错误码，不能泄露完整供应商响应。
4. 向量每批最多 10 个输入、1024 维，验证返回数量、有限数值及模型名；记录 token 等用量元数据，不记录正文、完整提示词或凭据。
5. 在测试中用 httpx.MockTransport 核对实际请求格式、非思考开关、修复次数和错误路径；LiveProviderTests 默认不发外网请求，留到 T20 显式启用。

**验证：** 运行 python -m unittest tests.test_providers -v，核对受控服务请求及失败行为。真实服务测试若因未启用而跳过，单独列出，不计为接通；生产缺配置请求返回 SERVICE_UNCONFIGURED，不能返回固定教学文本。

### T7：把原文索引、目录、检索和引用接入资料流程

**交付结果：** 确认预览后可生成有来源的目录，问答依据只来自当前用户当前资料，引用能打开真实原文。

**文件：** backend/app/documents/retrieval.py；backend/app/learning/context.py、citations.py、prompts.py；backend/app/services/materials.py；backend/app/api/materials.py；backend/tests/test_retrieval.py、test_documents.py。

**依赖：** T5、T6。

**步骤：**
1. 实现 prepare_outline：确认预览后再建向量和目录；整条任务 90 秒内完成，显式重试可复用当前资料已成功向量。
2. 发送全篇有效原文及块 ID 生成有序小节，检查来源范围覆盖开头、各主要部分和尾部；不能只对开头摘要生成全篇目录。
3. 原子发布 active_outline_id。替换必须确认并匹配旧目录 ID，旧会话仍指向旧目录，失败不覆盖原目录。
4. 实现 Retriever.retrieve，先按用户、资料、小节范围筛选候选，再做精确余弦排序；上下文包含当前目标、最多 8 个检索块和最近 6 轮完整对话。
5. 实现 CitationValidator.validate，核对块 ID、逐字 quote、偏移及所属资料，页码与段落从存储补充；来源外内容用 AI 补充和资料未涉及标记。
6. 加入恶意资料指令、虚假引用、同内容不同账号、超出范围问题和长资料尾部覆盖用例。

**验证：** 运行 python -m unittest tests.test_retrieval tests.test_documents -v；通过目录和原文 HTTP 入口观察三个主题均被覆盖；错误页码／quote／跨账号来源被拒绝；替换失败后旧会话可恢复；资料指令不改变账号权限或暴露内部内容。

### T8：完成初学模式的完整文字学习流程

**交付结果：** 按节讲解，完成一小节才邀请复述，复述反馈和跳过行为真实保存。

**文件：** backend/app/learning/engine.py、intents.py、context.py、prompts.py；backend/app/services/learning.py；backend/app/api/sessions.py；backend/tests/test_learning.py。

**依赖：** T7。

**步骤：**
1. 实现 TeachingEngine.decide 的初学部分；为小节保存 key_points、steps 和来源范围。
2. 接通开始、继续、提问、简化、举例、再讲、复述、跳过复述、下一节。提问／举例不推进讲解步骤，最后一步完成后才由后端追加整体复述邀请。
3. 自由文字按 UTTERANCE 进入受 phase 限制的意图识别；不确定时作为提问处理，不擅自改变进度。
4. 对复述输出正确点、遗漏、理解偏差及解释；分别保存 taught、recall_completed、needs_review，不把跳过或失败当作完成。
5. 用户明确进入下一节时允许跳过复述；重复讲解保留历史。使用任务原子提交，取消或不合法模型输出不推进。
6. 验证一小节三个知识点、中途插问、遗漏一点误解一点的复述、跳过、重讲、再次复述和下一节。

**验证：** 运行 python -m unittest tests.test_learning.LearnModeTests -v；从开始会话和 turns 入口走完流程：讲解中无强制逐段测验，完整讲解后出现复述邀请，跳过显示已讲解未复述，错误复述显示具体反馈和需要回看。

### T9：完成老师和面试官复习流程

**交付结果：** 一轮最多发布 5 道主问题，每次只问一道，每题最多追问一次，答案和总结不伪造独立作答。

**文件：** backend/app/learning/engine.py、intents.py、prompts.py；backend/app/services/learning.py；backend/app/api/sessions.py；backend/tests/test_learning.py。

**依赖：** T8。

**步骤：**
1. 为选择的小节和 teacher／interviewer 角色生成恰好 5 题及私有参考要点；允许未初学就进入复习。
2. 只序列化当前已问问题，禁止会话、任务结果或历史接口泄露未来题目和私有参考答案。
3. 回答后生成反馈，并由状态规则限制追问；追问不增加主问题索引，回答追问后不再追问第二次。
4. 实现跳过、查看答案、提前结束；被跳过或已看答案的题不记录为独立答对。
5. 第五题或提前结束后只根据已发生回答总结薄弱点和回看小节；总结失败保留记录并可重试，不重出 5 题。
6. 用正常五题、全题追问、跳过、揭示答案、提前结束和模型尝试追加第六题等用例验证。

**验证：** 运行 python -m unittest tests.test_learning.ReviewModeTests -v；正常完成恰有 5 道主问题、每题至多一次追问，公开响应无未展示答案；老师／面试官请求携带对应角色，跳过／看答案的结果和提前结束总结与真实记录一致。角色教学质量留到 T20 人工核对。

### T10：完成文字 HTTP 集成、历史恢复和公开视图核对

**交付结果：** 资料、两种会话、消息及进度通过真实应用入口连通；重启和切换模式后能继续学习。

**文件：** backend/app/main.py；backend/app/api/auth.py、materials.py、sessions.py、jobs.py；backend/app/services/learning.py；backend/tests/test_api.py、support.py。

**依赖：** T9。

**步骤：**
1. 按 Plan 路由契约装配资料、目录、会话、文字、历史、进度和任务入口；统一分页、snake_case、错误和身份检查。
2. 创建新会话返回长任务；resume 返回原状态，不再次出题或重新开始讲解。
3. 保存初学与复习各自历史；旧目录会话可继续读取；取消／失败和旧 revision 不能污染当前会话。
4. 为直接 API 越权、4,000 字符边界、重复点击、目录替换、退出恢复和模式切换建立跨模块测试。
5. 核对全部公共响应只包含 Plan 的公开字段，所有公开接口有实际客户端或集成调用入口。

**验证：** 运行 python -m unittest tests.test_api -v；在临时数据库中通过 HTTP 完成导入→目录→初学→复述→复习→关闭并重开应用→恢复，检查会话消息、题序和进度不丢失、不混淆；4,000 字符可发送、4,001 被拒绝。此任务使用隔离模型，不代表真实 AI 已验收。

### T11：完成两分钟录音的服务端转写与教学接续

**交付结果：** 合规录音只生成有效转写消息，随后自动进入已有教学流程；取消和失败后不留持久录音。

**文件：** backend/app/providers/qwen_asr.py；backend/app/storage/ephemeral_audio.py；backend/app/services/voice.py；backend/app/api/sessions.py；backend/app/tasks/lifecycle.py；backend/app/config.py；backend/tests/test_voice.py、test_jobs.py。

**依赖：** T10。

**步骤：**
1. 实现 ASRProvider.transcribe；服务器核验格式、实际时长及 4 MiB 上限，拒绝依赖客户端时长作判断。
2. 使用 ffprobe 和 ffmpeg 真正执行格式检查和单声道 16 kHz PCM 转换；有效时长最多 120 秒，全部处理共用 60 秒预算。
3. multipart 溢写和转码目录均在 tmpfs；实现最多 4 个对象的临时录音池、成功／异常／取消后的 finally 清理和启动清扫。生产目录不合规则拒绝语音。
4. 静音、空转写、损坏音频、超时和额度失败不创建虚假用户消息；有效结果才保存文字。
5. 在同一事务中从转写任务把学习名额交给后续教学任务，result.next_job_id 允许前端接续轮询；取消整条链后不能创建新回复。
6. 用实际生成的 120 秒、超长、静音、损坏及正常短音频执行转码和状态测试；重启丢失临时引用返回 AUDIO_REUPLOAD_REQUIRED。

**验证：** 运行 python -m unittest tests.test_voice.TranscriptionTests tests.test_jobs -v；实际 120 秒测试音频通过处理且可自动进入文字任务，超限和静音不产生用户消息，所有退出路径后临时目录／池为空、持久目录无原录音；已取消转写的迟到结果不建立教学子任务。真实转写在 T20／T21 补验。

### T12：完成独立语音合成、分段资产和私有下载

**交付结果：** 一条 AI 回复可完整分段合成、缓存和重试，语音错误不会重新生成教学答案。

**文件：** backend/app/providers/qwen_tts.py；backend/app/services/voice.py；backend/app/api/speech.py；backend/app/storage/files.py；backend/tests/test_voice.py、test_api.py。

**依赖：** T11。

**步骤：**
1. 实现 TTSProvider.synthesize，固定默认音色 Cherry；按句分成最多 500 字符片段，不截断实质正文。
2. 最多并发 2 个片段，总准备 60 秒；合法音频全部准备后才发布完整清单，部分失败清理不完整资产。
3. 供应商下载地址仅服务端使用，校验允许域名、重定向、音频类型和大小；不接受模型或用户指定任意下载地址。
4. 实现 speech 任务、状态及 /audio/{id} 鉴权下载；缓存键关联用户、资料、消息、模型和音色，默认 24 小时过期。
5. 验证重播缓存、已过期重建、部分片段失败、取消、跨用户下载和资料删除标记；重试不得调用文本模型。

**验证：** 运行 python -m unittest tests.test_voice.SynthesisTests tests.test_api -v；把含多段且超过 600 字符的回复合成为有序完整清单，每段不超过 500 字符；缓存重播不新增合成调用，失败保留原文字且无文本模型重调；用户乙无法下载甲音频。

### T13：完成删除、到期清理与数据生命周期

**交付结果：** 用户确认删除后立即失去相关访问，实际运行清理后全部项目内容对象消失且有时间证据。

**文件：** backend/app/services/deletion.py；backend/app/tasks/cleanup.py、lifecycle.py；backend/app/api/materials.py、speech.py；backend/app/storage/repositories.py；backend/tests/test_deletion.py、test_voice.py；docs/verification.md。

**依赖：** T12。

**步骤：**
1. DELETE 确认后标记删除、封锁资料及全部关联读取、取消相关任务链；取消确认不改变内容。
2. 实现每分钟清理、1／5／15／60 分钟重试及接近 24 小时的优先处理；清理源文件、正文、块、向量、目录、题目、对话、进度和音频。
3. 提供 python -m app.tasks.cleanup --once，输出分类计数和检查时间；仅保留最小删除记录，不保留正文副本。
4. 到期 TTS 缓存独立清理；删除时旧合成或教学结果不能重新写回。
5. 自动化验证重试和重入；另用真实临时存储执行一次删除，记录删除请求、实际完成时间、对象计数及文件存在性，证明在 24 小时内完成，不以假时钟代替实际清理证据。

**验证：** 运行 python -m unittest tests.test_deletion tests.test_voice -v，并对隔离验证资料实际运行清理入口；观察删除立即封锁、运行后内容与文件为零、分类时间记录齐全。清理失败时不能标记删除完成。

### T14：建立小程序身份、API 和任务状态基础

**交付结果：** 客户端可以依契约登录、请求、上传和跟踪任务，取消或切换后旧结果不覆盖新状态。

**文件：** miniprogram/services/api.ts、auth.ts、materials.ts、sessions.ts、jobs.ts；miniprogram/types/contracts.ts；miniprogram/state/learning.ts、voice.ts；miniprogram/tests/jobs.test.ts、learning.test.ts、support.ts；miniprogram/app.ts。

**依赖：** T1、T10。

**步骤：**
1. 封装 wx.login、会话令牌存取和退出；过期时清理旧身份，不复用旧用户资料界面。
2. 实现带 Authorization 的普通请求与上传；统一 snake_case 映射、Unicode 字符计数、15 秒请求失败提示和分页。
3. 每次用户提交生成幂等 key，网络重试复用，查询原任务优先于新建收费请求。
4. 实现每秒轮询、next_job_id 链接、取消、后台暂停及前台恢复；切换使用 generation_id 丢弃迟到结果。
5. 依赖服务器 allowed_actions、phase 和 revision 决定按钮；客户端不能修改服务端进度或私有题目。

**验证：** 在 miniprogram/ 运行 npm test 和 npm run typecheck。断言同一提交重用 key、过期令牌清界面、取消／切换后旧结果不覆盖新会话且语音链自动接续。

### T15：完成资料导入、预览和模式选择页面

**交付结果：** 用户在小程序中可从资料列表完成导入、确认正文、生成目录，并选择初学或复习。

**文件：** miniprogram/pages/library/index.ts、index.wxml、index.wxss、index.json；pages/import/、pages/preview/、pages/outline/ 下各自的 index.ts、index.wxml、index.wxss、index.json；miniprogram/app.json、app.wxss；miniprogram/tests/learning.test.ts。

**依赖：** T7、T14。

**步骤：**
1. 注册资料、导入、预览和目录页面，使用可滚动的手机竖屏布局，显示真实加载、空态、失败和重试。
2. 首次上传／处理前展示隐私说明并确认；拒绝时不触发内容上传或 AI 请求。
3. 提供 PDF／TXT／Markdown 文件选择和粘贴文字，客户端做友好预检，仍以服务端规则为准。
4. 显示文字预览、页码／段落、图片未纳入警告和确认按钮；目录生成失败可重试，替换已有目录须明确确认。
5. 目录显示小节目标和进度；初学选起点，复习选范围与老师／面试官。选择后创建对应会话进入学习页。

**验证：** 运行 npm run typecheck、npm test、npm run build。通过页面逻辑测试确认拒绝隐私调用次数为零，选择范围和角色正确进入请求，错误不显示成功。用开发者工具检查四页导航和竖屏布局；没有工具时这部分记录未验证，保留 T21 的实际补验。

### T16：完成两种学习模式、来源和历史界面

**交付结果：** 用户可完成文字初学／复习，区分资料与补充，打开来源并离开后恢复。

**文件：** miniprogram/pages/study/、pages/source/、pages/history/ 下各自的 index.ts、index.wxml、index.wxss、index.json；miniprogram/state/learning.ts；miniprogram/services/sessions.ts；miniprogram/tests/learning.test.ts；miniprogram/app.json。

**依赖：** T9、T14、T15。

**步骤：**
1. 学习页显示当前资料、小节、模式、角色或题序，按公共 ContentBlock 呈现资料依据、AI 补充及资料未涉及。
2. 提供文字发送和当前允许动作；初学讲完小节才显示复述提示，复习一题一答及至多一次追问，不在客户端保留私有参考答案。
3. 引用点击打开真实原文块和 PDF 页码／段落；对不存在来源显示明确错误，不拼凑来源内容。
4. 历史页显示真实对话、复习总结、需要回看和继续入口；切换模式创建对应会话，恢复不重新生成题目。
5. 录音及播放按钮位置先预留交互区域，功能在 T17–T18 接入后才启用；不显示假声波、假转写或假播放成功。

**验证：** 运行 npm run typecheck、npm test、npm run build；页面逻辑用例走完小节讲解、跳过复述、五题含追问、看答案、提前结束和恢复，观察动作与服务器状态一致。开发者工具核对来源及历史导航；真实教学内容在 T20／T21 核对。

### T17：接通小程序分段播报、停止和重播

**交付结果：** AI 文字可以自动播报和重播，停止、录音请求、切后台或迟到下载不会继续发声。

**文件：** miniprogram/services/playback.ts；miniprogram/state/voice.ts；miniprogram/pages/study/index.ts、index.wxml、index.wxss；miniprogram/tests/playback.test.ts、support.ts。

**依赖：** T12、T16。

**步骤：**
1. 完整文字消息收到后按自动播报开关请求 speech，关闭时不发合成请求。
2. 通过带令牌的 wx.downloadFile 下载私有音频，用一个 InnerAudioContext 按片段顺序播放；首次下载和播放器就绪也受原 60 秒准备截止时间约束。
3. 支持停止、重播当前回复和开关；停止增加播放 generation_id，并终止后续下载／播放。
4. 接通切后台、播放器异常和给录音模块使用的 stop 方法；旧片段下载完成后不能重新播放。
5. 服务失败显示文字已保留，可只重试语音；不重新提交教学问题。

**验证：** 运行 npm test、npm run typecheck、npm run build；受控下载器和播放器验证分段顺序、关闭自动播报零合成、停止后迟到片段不播放、60 秒等待边界和重试仅调用语音。实际扬声器播放在 T21 真机观察。

### T18：接通两分钟手动录音与自动提交

**交付结果：** 点击开始和结束即可语音学习，120 秒自动提交，取消／拒绝权限／退出不会误发。

**文件：** miniprogram/services/recorder.ts；miniprogram/state/voice.ts；miniprogram/pages/study/index.ts、index.wxml、index.wxss；miniprogram/tests/recorder.test.ts、support.ts；miniprogram/app.json。

**依赖：** T11、T16、T17。

**步骤：**
1. 使用 RecorderManager：duration=120000、16 kHz、单声道 AAC、48 kbps；从 onStart 计时，110 秒提示即将结束。
2. 点击开始先停止 AI 播报，再申请录音；拒绝权限时保留文字输入和可理解的权限提示。
3. 手动结束和达到上限共用 onStop 提交路径，用一次性标识防止重复发送；结束后自动上传、转写、显示识别文字并接续教学，无二次发送按钮。
4. 取消、录音错误、切后台时停止并丢弃录音、清理临时文件；下一次进入不自动录音。
5. 显示录音、上传、转写、等回复、准备播报、播放等不同状态；一个提交未结束时阻止再次输入，可取消等待。
6. 重录／文字更正生成新有效发言，不修改已保存历史；服务要求重录时不反复重试已经不存在的音频。

**验证：** 运行 npm test、npm run typecheck、npm run build；假时钟到 110／120 秒检查提示及单次提交，手动结束与 onStop 同时发生仍只发送一次，取消／后台／拒绝权限不发送，开始录音确实调用播放停止。实际录满两分钟在 T21 单独验证。

### T19：完善删除交互、运行配置和交付说明

**交付结果：** 用户可明确确认或取消删除，项目方能按文档配置真实环境并启动单实例服务。

**文件：** miniprogram/pages/library/index.ts、index.wxml；miniprogram/services/materials.ts；Dockerfile、compose.yaml、README.md、.env.example；docs/setup-wechat.md、privacy-notice.md、verification.md；backend/app/config.py、main.py。

**依赖：** T13、T18。

**步骤：**
1. 资料删除弹窗列明将删原资料、目录、对话、进度和音频；取消不发删除，确认后清理相关客户端缓存和进行中音频。
2. 整理虚拟环境、依赖安装、ffmpeg、服务启动、微信 AppID 和 HTTPS 合法域名配置；说明单进程和同机持久盘限制。
3. Docker 使用单服务进程、私有数据持久卷与至少 128 MiB 的 tmpfs；multipart 临时目录也指向 tmpfs，不把原录音写入持久目录。
4. 保持测试身份、受控模型、真实秘密和用户数据不进入生产构建；缺服务配置时明确报错，不提供假教学回退。
5. 文档分别给出本地可运行、真实服务、开发者工具、Android 和 iPhone 的验证条件；Windows 源码使用说明不假装已经在用户 E 盘安装。
6. 不实际购买服务器、开通账号或公开发布；提供可审阅的源码和配置。

**验证：** 运行后端自动化及小程序三项检查，实际启动服务访问 /health；观察删除取消／确认请求差异。若 Docker 可用，运行 docker compose config 和 docker compose build；不可用时仅记录容器构建未验证，不凭文件存在宣称成功。核查构建产物和交付目录不含秘密与真实数据。

### T20：真实模型、转写和合成服务联调

**交付结果：** 对预设事实资料用真实服务核对教学、角色差异和语音；不以受控响应代替通过证据。

**文件：** backend/tests/test_providers.py；docs/verification.md；必要时修复已有 providers、learning、voice 模块；不改 Spec 的行为边界。

**依赖：** T19。

**步骤：**
1. 确认项目方已经在其服务环境配置可用百炼凭据、业务空间与额度，不在聊天或日志打印秘密；沿用已批准服务选型和小规模样例。
2. 显式运行 LiveProviderTests，实际检查文本模型、1024 维向量、中文转写和语音合成，记录模型名、时间、用量和请求编号。
3. 用三主题资料核对目录、小节讲解、遗漏／误解反馈、老师／面试官侧重、资料外补充标签及正确来源；自动验证结构，人工核对教学语义。
4. 核对音频有效、完整长回复分段有序、两分钟输入边界，并确认原录音已清理；记录失败与修复，不只保存成功样例。
5. 条件缺失时逐项标明“未验证”、缺什么和补验命令，不能用默认跳过的测试套件给出服务已接通结论。

**验证：** 在 backend/ 显式运行 RUN_LIVE_TESTS=1 python -m unittest tests.test_providers.LiveProviderTests -v，记录实际输出与费用元数据；人工按已知事实逐例核对。只有真正发出请求、收到合法内容并完成核对的项目记通过。网络／凭据不可用时停止相关真实调用，不自动更换为未批准供应商。

### T21：微信开发者工具与 Android／iPhone 端到端检查

**交付结果：** 实际观察两种学习模式和两分钟语音在目标微信环境的行为。

**文件：** docs/verification.md；必要时修复对应小程序与服务端文件；实际配置中的秘密不进入交付包。

**依赖：** T20 的真实服务已可用；若 T20 未验证，本任务不能宣称端到端通过。

**步骤：**
1. 在项目方真实微信账号下配置 AppID、体验成员、隐私能力和 HTTPS 域名，导入 miniprogram/ 项目并运行 npm run build 的最新产物。
2. 开发者工具完成登录、隐私拒绝／同意、文件与粘贴导入、预览、目录、初学复述、老师及面试官复习、来源打开、历史恢复和删除确认。
3. 记录一台 Android 和一台 iPhone 的系统、微信版本；每台各执行两种模式语音收发，录满 120 秒并检查自动结束只提交一次。
4. 验证拒绝麦克风、停止／重播、录音停止播报、切后台丢弃录音、网络失败文字保留、键盘及底部安全区域不遮挡关键按钮。
5. 用两个真实用户检查个人数据隔离。把没有实际观察到的设备、操作和服务场景逐项留为未验证。
6. 如需用户参与，仅给出具体操作及要观察的结果；不得假装通过当前 Linux 工作区直接操作用户 Windows 上的微信开发者工具。

**验证：** 按批准后的 Checklist 执行真实操作，逐项记录观察结果和版本；必要证据以不含个人信息的文字记录为主。开发者工具、Android、iPhone 分开判断，单一平台成功不能替代其他平台。

### T22：回归、验收与完整源码交付

**交付结果：** 可下载的完整源码包、明确的运行方式，以及如实区分通过／未通过／未验证的验收记录。

**文件：** docs/v1/task.md、checklist.md；docs/verification.md；README.md；项目目录外的 deliverables/qna-learning-source.zip。

**依赖：** T19；T20、T21 按已具备条件完成并提供实际状态，条件缺失时保留未验证，不阻止交付已完成源码。

**步骤：**
1. 对最后一轮修复运行后端完整自动化、语法检查、小程序类型检查、逻辑测试和构建；只在新改动或失败需要时重复检查。
2. 按获批 checklist.md 执行验收，逐项保存证据；同一清单项只计一次，三种状态之和等于总项数。
3. 更新 Tasks 中各任务实际状态；代码完成但真实服务／设备未验的任务明确区分“实现完成”和“验证未完成”，不能统一勾成完成。
4. 整理 README 的启动、配置、限制、未验证项和补验方式；若全部端到端尚未完成，说明还差哪项条件。
5. 生成 qna-learning-source.zip，包含服务端、小程序、锁定依赖、四份流程文档和说明；排除 .env、真实密钥、用户资料、数据库、录音、node_modules、虚拟环境、缓存、内部辅助工具和重复构建目录。
6. 检查压缩包清单和秘密排除，将最终源码包与验收文档保存为可下载交付物。未授权的提交、公开发布或实际购买不在此任务中执行。

**验证：** 实际运行第 1.1 节所列全量命令；核对 Checklist 计数、失败和跳过；用压缩包清单确认两个源码目录、四份文档及运行说明齐全且无秘密和用户数据。仅在全部对应证据满足时报告完整目标通过。

## 4. 执行顺序与依赖

可按下表顺序串行执行全部任务，无需并行代理。表中“可独立推进”仅表示技术依赖允许，不代表会自动委派任务。

| 批次 | 任务 | 前置条件／交付里程碑 |
|---|---|---|
| 1 | T1 → T2 → T3 → T4 | 应用基础、隔离存储、身份与隐私、任务状态。 |
| 2 | T5、T6 | 二者都依赖 T4；资料预览与文本／向量适配可分别验证。 |
| 3 | T7 → T8 → T9 → T10 | 检索目录 → 初学 → 复习 → 后端文字闭环。 |
| 4 | T11 → T12 → T13 | 服务端转写 → 分段合成 → 删除与清理。 |
| 5 | T14 → T15 → T16 | 客户端请求 → 资料页面 → 两种模式、来源及历史。T14 在 T10 后即可开始。 |
| 6 | T17 → T18 → T19 | 播放 → 两分钟录音 → 完整运行与删除交互。 |
| 7 | T20 → T21 | 真实服务及真实微信环境；条件缺失分别记未验证。 |
| 8 | T22 | 最终回归、Checklist 验收及源码交付，列清仍未验证条件。 |

一个合法的完整执行顺序为 T1、T2、T3、T4、T5、T6、T7、T8、T9、T10、T11、T12、T13、T14、T15、T16、T17、T18、T19、T20、T21、T22。

## 5. 需求与任务覆盖

| 需求 | 任务 | 验收标准 |
|---|---|---|
| F1 个人身份与资料空间 | T2、T3、T10、T14、T21 | AC1 |
| F2 资料导入 | T5、T15 | AC2 |
| F3 解析预览与失败 | T5、T15 | AC3 |
| F4 目录与范围 | T7、T15 | AC4 |
| F5 初学教学 | T8、T16、T20、T21 | AC5、AC6 |
| F6 复习教学 | T9、T16、T20、T21 | AC7、AC8 |
| F7 轮流语音与文字 | T10、T11、T18、T20、T21 | AC9、AC18 |
| F8 AI 播报 | T12、T17、T20、T21 | AC10 |
| F9 依据与补充 | T7、T8、T9、T16、T20 | AC11 |
| F10 记录与恢复 | T8、T9、T10、T16、T21 | AC6、AC8、AC12 |
| F11 删除 | T13、T19、T21 | AC13 |
| N1 隐私、隔离及凭据 | T2、T3、T11、T12、T13、T14、T19 | AC1、AC9、AC13、AC14 |
| N2 教学可靠性 | T6、T7、T8、T9、T20 | AC11、AC15、AC19 |
| N3 等待、取消和重试 | T4、T10、T11、T12、T14、T17、T18 | AC16 |
| N4 目标环境与可用性 | T15、T16、T17、T18、T21 | AC17 |
| N5 使用限制与服务状态 | T1、T4、T5、T6、T10、T11、T12 | AC2、AC16、AC18 |

Plan 的客户端、身份隐私、资料、教学、检索引用、语音、任务生命周期全部有实现任务；T20–T22 覆盖真实完整流程 AC19 及证据汇总。

## 6. 当前任务状态与审批

| 任务 | 当前状态 |
|---|---|
| T1 | 已完成；15 项后端测试、两组类型检查、构建、1 项客户端测试及实际进程健康检查通过 |
| T2 | 未开始 |
| T3 | 未开始 |
| T4 | 未开始 |
| T5 | 未开始 |
| T6 | 未开始 |
| T7 | 未开始 |
| T8 | 未开始 |
| T9 | 未开始 |
| T10 | 未开始 |
| T11 | 未开始 |
| T12 | 未开始 |
| T13 | 未开始 |
| T14 | 未开始 |
| T15 | 未开始 |
| T16 | 未开始 |
| T17 | 未开始 |
| T18 | 未开始 |
| T19 | 未开始 |
| T20 | 未开始 |
| T21 | 未开始 |
| T22 | 未开始 |

本任务拆解只细化已批准设计的实现文件、辅助测试和构建入口，没有改变 Spec 的功能与费用边界。22 个任务全部带有可观测结果、明确依赖和验证方式。

请审批任务粒度、依赖和步骤。批准后进入 checklist.md 验收设计；Checklist 获批前不编写实现代码。


## 2026-11-17 接续说明

原文档末尾审批提示保留为历史阶段记录。用户已明确要求读取现有四份文档并从 T1 开始开发，本次按该授权进入实现，不重复请求批准；需求与设计范围不变。当前进度和实际证据以 task.md 及 docs/verification.md 为准。
