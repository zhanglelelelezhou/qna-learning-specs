# 问答式学习 Plan

- 版本：v0.1
- 日期：2026-11-16
- 状态：2026-11-17 用户授权按现有四份文档从 T1 开始开发
- 需求基线：Spec v0.2，用户已于 2026-11-16 明确批准。
- 本阶段交付：架构、数据结构、接口、交互、文件组织、技术决策、费用估算和验证策略。
- 项目根目录：E:/kaifa/kaifa-spec/qna-learning/；四份文档保留在 docs/v1/。

## 1. 架构与实施边界

### 1.1 首版技术选择

| 层次 | 选择 | 用途与理由 |
|---|---|---|
| 微信客户端 | 原生微信小程序，TypeScript、WXML、WXSS | 只实现微信端；直接使用录音、播放、文件选择和登录能力，减少跨端适配层。 |
| 服务端 | Python 3.12、FastAPI、Pydantic、httpx | 提供鉴权、资料处理、教学会话和语音接口；结构化校验 AI 输出；外部调用集中管理。 |
| 持久数据 | SQLite，Python 标准库 sqlite3，版本化 SQL 迁移 | 适合首版单机、小范围体验；无需额外部署数据库。生产数据放在同机持久磁盘，不放临时目录或网络共享盘。 |
| 文件 | 私有数据目录中的原始资料和合成音频；录音只在内存及内存临时目录处理 | 原资料可清理、音频可重播；不生成公开资料 URL，不持久保存用户录音。 |
| 后台任务 | SQLite 任务表＋FastAPI 生命周期内的异步调度器；PDF 解析使用可终止子进程 | 长任务不占据单次小程序请求；具有明确超时、取消和重启恢复规则。首版只启动一个服务进程。 |
| 检索 | 百炼 text-embedding-v4，1024 维；当前资料范围内做精确余弦排序 | 每份资料只有数百个块，无需单独向量数据库；引用仍来自保存的原文块。 |
| 教学模型 | 阿里云百炼 qwen-plus，非思考模式 | 处理目录、教学计划、讲解和评价；JSON 输出经过服务端验证，不让模型直接改数据库或执行工具。 |
| 语音识别 | 百炼 qwen3-asr-flash；完整录音提交后识别 | 使用 Base64 音频输入，避免把用户录音上传到公开存储。 |
| 语音合成 | 百炼 qwen3-tts-flash，固定系统音色 Cherry | 文本与播报分开处理；长文本分段合成，音频故障不撤销文字回答。 |
| 交付与运行 | 本地 Python 服务；可选 Docker 单容器＋持久目录；反向代理 HTTPS | 先交付可运行源码和微信开发者工具配置。真机体验需要项目方提供真实 AppID、服务凭据和可访问的 HTTPS 服务地址。 |

所选模型集中放在配置和独立适配器中。技术设计批准不代表已经开通或购买服务；本阶段不调用收费推理接口。

不用 Redis、Celery、独立向量服务或复杂 Agent 框架。SQLite 采用默认回滚日志、短事务、busy_timeout 和 secure_delete；写入串行，事务不跨外部网络请求。首版不配置正文备份或内容日志，以免删除后留下未受控副本。更高并发、集群和自动备份不在本版范围。

### 1.2 模块职责

| 模块 | 职责 | 依赖 |
|---|---|---|
| 小程序界面与客户端服务 | 导入、预览、目录、教学、复习、来源查看、语音状态和删除确认 | HTTP API、微信基础能力 |
| 身份与隐私 | 微信登录交换、会话令牌、隐私确认、所有权校验 | 微信服务端登录接口、数据库 |
| 资料服务 | 校验、提取文字、原文定位、目录与索引生成 | 解析器、文件存储、模型和向量适配器 |
| 教学控制器 | 执行初学和复习状态转换；限制题数、追问和复述时机 | 资料检索、上下文、模型适配器、数据库 |
| 检索与引用 | 只在当前资料中取依据，验证引用，区分补充内容 | 原文块、向量适配器 |
| 语音服务 | 录音上传校验、转写、分段合成、私有下载和清理 | 内存录音池、ffmpeg/ffprobe、语音适配器 |
| 任务与生命周期 | 幂等、取消、限流、截止时间、原子结果提交、删除清理 | 数据库、时钟、各领域服务 |

依赖方向为“路由→领域服务→存储或外部适配器”；外部适配器不导入路由，纯状态转换不调用网络。

### 1.3 微信页面

1. 资料页：资料列表、当前处理状态、继续学习、导入入口及删除入口。
2. 导入页：选择文件或粘贴正文，显示隐私说明、格式及大小限制。
3. 预览页：提取文字、PDF 页码或段落、图片未纳入提示、确认生成目录。
4. 目录页：学习小节及目标，选择初学起点或复习范围、老师或面试官。
5. 学习页：共用对话界面；显示当前模式、小节或题目进度、录音、播报、取消等待及模式对应操作。
6. 原文页：根据已验证定位打开原文块；按页或段落滚动。首版无需嵌入完整 PDF 阅读器。
7. 记录页：历史会话、未结束复习、轮次总结及回看入口。

初学操作包括继续讲解、再讲一次、开始复述、跳过复述、下一节。复习操作包括回答、跳过、查看答案、结束本轮。普通用户界面不出现模型密钥、内部任务标识或工程配置。

## 2. 数据结构与持久化

所有数据库主键使用服务端生成的 UUID 字符串；时间使用 UTC，客户端按本地时区显示。字符数统一按 Unicode 码点计数，不按 UTF-8 字节或 JavaScript UTF-16 单元计数。文件大小上限 20 × 1024 × 1024 字节。所有内容对象都有 owner_id，并通过父对象所有权再次检查。

### 2.1 核心对象

| 对象／表 | 字段及约束 |
|---|---|
| User / users | id；wechat_openid 唯一；consent_version、consented_at 可空；created_at。不把客户端传入的 openid 当作身份依据。 |
| AuthSession / auth_sessions | id；user_id；token_hash；expires_at；revoked_at。随机令牌只返回一次，库中保存摘要，默认 7 天有效；过期后重新微信登录。 |
| Material / materials | id；owner_id；title；kind 为 pdf/txt/markdown/paste；original_path 可空；byte_size；text_length；nonblank_length；status；warnings；active_outline_id 可空；deleted_at 可空；created_at。 |
| SourceBlock / source_blocks | id；material_id；owner_id；ordinal；text；page_number 可空；paragraph_number；heading 可空；start_char、end_char。页码由 PDF 解析器生成，段落编号由服务端生成。 |
| Chunk / chunks | id；material_id；owner_id；ordinal；text；spans，为原文块 ID 与各自字符范围；embedding_model；embedding_dim；embedding，为 1024 个 little-endian float32；content_hash。 |
| Outline / outlines | id；material_id；owner_id；revision；status；created_at。内部保留旧目录以保护已有会话，不提供面向用户的版本编辑功能。 |
| Section / sections | id；outline_id；owner_id；ordinal；title；objectives；source_ranges。每个范围包含 start_block_id、end_block_id；只能指向当前资料。 |
| SectionLesson / section_lessons | id；section_id；owner_id；key_points；steps，包含顺序、标题、来源范围；model；created_at。一个小节的讲解步骤在开始时确定，不能由每轮回答随意增加。 |
| LearningSession / learning_sessions | id；owner_id；material_id；outline_id；mode 为 learn/review；role 为 teacher/interviewer 或空；selected_section_ids；current_section_id；phase；step_index；current_question_index；revision；status；created_at、updated_at。 |
| SectionProgress / section_progress | owner_id＋section_id 唯一；taught；recall_completed；needs_review；last_session_id；updated_at。界面优先显示需要回看，其次已完成复述、已讲解未复述、未开始；避免一个状态覆盖真实历史事实。 |
| Message / messages | id；owner_id；session_id；turn_id；role 为 user/assistant；input_kind 为 text/voice/action；content_blocks；status 为 accepted/completed/failed/cancelled；created_at。只保留有效用户文字；录音字段不保存音频。 |
| ReviewQuestion / review_questions | id；owner_id；session_id；ordinal 1–5；section_id；question；private_reference_answer；private_key_points；citations；status；followup_question 可空；followup_count 0–1；independent_answer。私有字段禁止序列化给客户端。 |
| ReviewAttempt / review_attempts | id；question_id；owner_id；kind 为 answer/followup/skip/reveal；user_message_id 可空；feedback_message_id 可空；correct_points、missing_points、misconceptions；created_at。评价为文字反馈，不输出认证分数。 |
| Job / jobs | id；owner_id；chain_id；parent_job_id 可空；kind；scope_type、scope_id；idempotency_key；request_hash；status；phase；input_refs；result_refs；error_code；created_at；deadline_at；started_at；finished_at；expected_revision。input_refs 只保存现有对象引用，不复制原始录音、凭据或完整模型上下文。 |
| ActiveSlot / active_slots | owner_id＋category 唯一；job_id；expires_at。category 为 material/learning/asr；用于跨点击和跨请求限制在途任务。 |
| AudioAsset / audio_assets | id；owner_id；material_id；message_id；part_index；part_count；voice；model；text_hash；private_path；expires_at；status。音频按回复所属资料隔离，不跨用户共享缓存。 |
| UsageEvent / usage_events | id；owner_id；job_id；provider_request_id；model；input_tokens；output_tokens；audio_seconds；characters；result_code；created_at。只保存计费元数据。 |
| Deletion / deletions | id；owner_id；material_id；requested_at；deadline_at；status；retry_count；last_error_code。只留删除证据，不留被删正文。 |

原文、对话和索引存储在同一私有 SQLite 文件中，原资料和语音文件在同机私有目录。连接启用 foreign_keys；删除顺序和外键级联共同保证不留下可访问的关联内容。

### 2.2 公共消息结构

~~~typescript
type ContentBlock =
  | { kind: "source"; text: string; citations: Citation[] }
  | { kind: "supplement"; text: string; label: "AI 补充" }
  | { kind: "uncovered"; text: string; label: "资料未涉及" }
  | { kind: "control"; text: string };

interface Citation {
  sourceBlockId: string;
  start: number;
  end: number;
  quote: string;
  pageNumber: number | null;
  paragraphNumber: number;
}

interface PublicMessage {
  id: string;
  role: "user" | "assistant";
  blocks: ContentBlock[];
  status: "accepted" | "completed" | "failed" | "cancelled";
  createdAt: string;
}

interface JobView {
  id: string;
  chainId: string;
  kind: string;
  status: "queued" | "running" | "succeeded" | "failed" | "cancelled" | "expired";
  phase: string;
  deadlineAt: string;
  result: object | null;
  error: { code: string; message: string; retryable: boolean } | null;
}
~~~

模型不能指定可信页码。引用由模型返回 source_block_id、引用文本及偏移候选，服务端与原文逐字核对后生成 Citation；页码和段落号从数据库补充。源文性质的解释块必须有有效引用；无依据的段落改为明确的补充或资料未涉及提示，不把错误引用当作有效内容发送。

引用校验能验证原文存在及位置，不能单独保证整段解释在语义上正确。教学样例仍需要真实模型运行及人工核对。

### 2.3 内部边界类型的完整约定

以下名称供第 4 节接口共用。字段中的 ID 均为字符串，列表默认为空列表；除注明可空的字段外，字段必填。错误通过统一领域异常返回，不在成功类型中混入错误文字。

| 类型 | 字段／定义 |
|---|---|
| UserContext、AuthResult | UserContext：user_id、consent_version 可空、consented_at 可空。AuthResult：token、expires_at、user={id, consent_required}。 |
| UploadInput、PrivateFileRef | UploadInput：title、filename、declared_mime、bounded_stream。PrivateFileRef：material_id、owner_id、private_path、detected_kind、byte_size；private_path 只能由文件存储生成。 |
| ReplaceOutlineInput | replace_confirmed 布尔值、expected_outline_id 可空；第一次创建时必须为空，替换时必须匹配当前目录。 |
| SourceQuery、SourcePage | SourceQuery：block_id/page/cursor 均可空、limit。SourcePage：material_id、blocks 为 SourceBlock 公开字段列表、next_cursor 可空、warnings。互斥查询条件冲突时拒绝请求。 |
| ParsedDocument、MaterialView | ParsedDocument：kind、blocks、text_length、nonblank_length、warnings。MaterialView：id、title、kind、status、byte_size、text_length、warnings、active_outline_id 可空、latest_job_id 可空、created_at、updated_at；不含原文件路径和完整正文。 |
| RetrievedChunk | chunk_id、score、text、source_spans；source_spans 指向允许发送给模型的 SourceBlock 及字符范围，score 仅作排序，不作为教学可信度。 |
| StartSessionInput | material_id、outline_id、mode、section_ids、role 可空、resume_session_id 可空。learn 恰选一个起点；review 至少一个小节且必须指定合法 role。resume 字段出现时其余选择只能与原会话一致。 |
| TurnInput、TurnAction | TurnInput：action、text 可空、expected_revision。TurnAction：UTTERANCE、CONTINUE、ASK、SIMPLIFY、EXAMPLE、REPEAT、RECALL、SKIP_RECALL、NEXT_SECTION、ANSWER、SKIP、REVEAL、END_REVIEW。直接提问、回答和复述需要非空 text；动作与当前 mode/phase 不相容则拒绝且不推进状态。 |
| SessionState、SessionView | SessionState：LearningSession 全字段，加当前 SectionLesson／ReviewQuestion 的服务端信息及现有进度。SessionView：id、material_id、outline_id、mode、role、phase、revision、status、current_section 公开字段、current_question 公开字段或空、progress、allowed_actions、last_job_id 可空；不含私有答案、未来题目和模型上下文。 |
| ModelRequest | task、system_rules、mode_context、source_blocks、history、user_input 可空、expected_schema、model、max_output_tokens、thinking=false。task 只能为 outline/lesson_plan/classify/teach/recall_feedback/review_plan/review_feedback/review_summary。 |
| ModelAnswer | parsed_output、model、provider_request_id、usage={input_tokens,output_tokens}。parsed_output 按 task 校验：outline 为标题／目标／来源范围列表；lesson_plan 为要点／步骤；classify 为 intent；其余为 blocks，加任务对应 feedback、followup_proposal 或 questions。 |
| Feedback、ReviewPlan | Feedback：correct_points、missing_points、misconceptions，每个条目包含 text 和 citations；additional_explanation 为 ContentBlock 列表；needs_review 布尔值。ReviewPlan：questions 长度必须为 5，每题包含 section_id、question、private_reference_answer、private_key_points、citations；仅私有保存。 |
| ValidatedAnswer、ValidatedTeachingProposal | ValidatedAnswer：经过校验的 blocks、model、usage。ValidatedTeachingProposal：intent、answer、feedback 可空、followup_question 可空、lesson_plan 可空、review_plan 可空。转换器不能接受未校验的 ModelAnswer。 |
| SessionTransition、DomainResult | SessionTransition：expected_revision、next_session_fields、progress_updates、question_events、public_messages。DomainResult：job_id、public_result、transition 可空、new_artifacts、usage_events；持久化由 commit_result 一次完成。 |
| JobInput、JobLease、CommitOutcome | JobInput：kind、chain_id、parent_job_id 可空、scope、input_refs、request_hash、deadline_at、expected_revision 可空。JobLease：job_id、owner_id、chain_id、expected_revision、deadline_at、lease_token。CommitOutcome：committed 布尔值、job_view、reason_code 可空；旧 lease 或已取消任务不能提交。 |
| EphemeralAudioInput、Transcript | EphemeralAudioInput：bounded_bytes、detected_format、actual_seconds、expected_revision；不含持久路径。Transcript：text、actual_seconds、provider_request_id、model、valid_speech 布尔值。转写合法性通过后才建立用户消息。 |
| SynthesizedAudio | audio_bytes、mime_type、actual_seconds、characters、provider_request_id、model；供应商下载 URL 在适配器内部消化，不进入公共类型。 |
| DeletionView、PurgeResult | DeletionView：id、material_id、status、requested_at、deadline_at、completed_categories、finished_at 可空、error_code 可空。PurgeResult：deletion_id、status、category_counts、checked_at、error_code 可空；不包含被删正文。 |

领域异常至少包含 CONSENT_REQUIRED、NOT_FOUND、UNSUPPORTED_FILE、TEXT_LIMIT、INVALID_AUDIO、NO_SPEECH、SERVICE_UNCONFIGURED、PROVIDER_QUOTA、PROVIDER_FAILURE、INVALID_MODEL_OUTPUT、JOB_EXPIRED、JOB_CANCELLED、SESSION_CHANGED、IN_FLIGHT、IDEMPOTENCY_CONFLICT、AUDIO_REUPLOAD_REQUIRED。对应用户提示在统一错误模块维护。

## 3. 对外 HTTP 接口

### 3.1 共同约定

- 路径前缀 /api/v1；JSON 字段使用 snake_case；上节 TypeScript 字段为客户端映射后的名称。
- 除 /health、/auth/wechat 和 /privacy 外均要求 Authorization: Bearer <会话令牌>。登录路由不调用 AI。
- 未确认隐私条款时，仅允许登录、获取公开隐私说明、确认条款及退出；资料上传、粘贴和处理返回 403 CONSENT_REQUIRED。
- owner_id 永远从令牌取得；跨用户访问和不存在资源统一返回 404。
- 非幂等创建或动作请求携带 Idempotency-Key。相同 key 和内容返回同一结果；同 key、不同内容返回 409 IDEMPOTENCY_CONFLICT。
- 产生长任务时立即返回 202 与 JobView；客户端每 1 秒轮询一次任务，进入后台后暂停轮询，回到前台读取最新状态。每次 HTTP 请求最多等待 15 秒，不用单次 HTTP 连接承载完整 90 秒模型任务。
- 上传使用 multipart；正文流式计数和限制体积，不能只信客户端 file_size 或 Content-Length。原资料接受 20 MiB；录音接受 4 MiB，正常两分钟压缩录音远低于此值。
- 文字发送上限为 4,000 个 Unicode 码点；语音转写若超过相同学习消息上限则明确提示分段重录，不自动截断语义。
- 消息和资料列表使用游标分页，默认 20 项、最多 50 项；预览每次返回最多 50 个原文块。
- 错误统一包含 code、message、retryable、request_id；不直接透传服务商原始请求、音频地址、堆栈或密钥。

### 3.2 路由契约

| 方法与路径 | 输入 | 输出／行为 |
|---|---|---|
| GET /health | 无 | 进程可用性与存储就绪；不暴露凭据或连接字符串。 |
| POST /auth/wechat | code | 服务端交换微信身份，返回 token、expires_at 和用户公开信息；不返回 session_key。 |
| POST /auth/logout | 无 | 撤销当前令牌。 |
| GET /privacy | 无；可公开访问 | 当前说明、外部服务名称、处理用途、录音临时处理及删除边界。 |
| POST /consents | version、accepted=true | 记录当前隐私确认；accepted=false 不记录同意，也不开始处理。 |
| GET /materials | cursor、limit | 当前用户未删除资料及处理状态。 |
| POST /materials/files | file、title | 202，建立资料与 parse 任务。 |
| POST /materials/text | title、text | 创建资料和提取结果，返回可预览资料；执行同样正文校验。 |
| GET /materials/{id} | 无 | 资料元数据和状态；不返回私有文件路径。 |
| GET /materials/{id}/source | page、cursor、limit 或 block_id | 原文块与定位；分页预览和来源页面共用。 |
| POST /materials/{id}/outline | replace_confirmed、expected_outline_id | 首次或确认替换后创建 prepare_outline 任务，返回 202；旧目录有会话时仍保留其内部引用。 |
| GET /materials/{id}/outline | outline_id 可空 | 默认当前目录；只能读取该用户资料的目录。 |
| POST /sessions | material_id、outline_id、mode、section_ids、role、resume_session_id 可空 | 新会话返回 202 并生成首段教学或首道复习题；resume 返回 200 的原会话，只恢复状态，不重建题目。 |
| GET /sessions | material_id 可空、cursor、limit | 会话列表与可恢复位置。 |
| GET /sessions/{id} | 无 | 当前模式、状态、公共当前题目或小节、进度及最后任务；不泄露未问问题和参考答案。 |
| GET /sessions/{id}/messages | cursor、limit | 公共消息历史。 |
| POST /sessions/{id}/turns | action、text 可空、expected_revision | 202，创建教学动作任务；action 见第 5 节。 |
| POST /sessions/{id}/voice-turns | audio、expected_revision、client_duration_ms | 202，转写任务；成功后原子建立有效用户消息和后续教学任务，result.next_job_id 用于接续轮询。 |
| GET /jobs/{id} | 无 | JobView；成功结果只包含已授权的资料、会话、消息或音频公开视图。 |
| POST /jobs/{id}/cancel | 无 | 幂等取消当前任务及其未完成子任务；取消链后不能继续自动创建教学子任务。 |
| POST /jobs/{id}/retry | 无 | 明确用户重试；已成功任务返回原结果，不重新计费。失败的纯文字或资料任务建立新任务并关联原输入。录音原件已清除时返回 AUDIO_REUPLOAD_REQUIRED。 |
| GET /materials/{id}/progress | 无 | 当前资料各小节事实状态与可恢复会话。 |
| POST /messages/{id}/speech | voice="Cherry" | 202，合成整条回复的音频片段；缓存命中直接返回原资产或同一任务。 |
| GET /messages/{id}/speech | 无 | 片段清单、就绪／失败状态和有效期，不返回供应商下载链接。 |
| GET /audio/{id} | 无 | 鉴权下载当前用户尚未过期的音频；检查资料未删除。 |
| DELETE /materials/{id} | confirm=true | 202，立即封锁访问、取消关联任务，返回删除任务状态。 |
| GET /deletions/{id} | 无 | 删除进度及已清理类别；不返回被删除内容。 |

音频经小程序 wx.downloadFile 携带用户令牌下载到临时文件，再由 InnerAudioContext 播放；不把需要鉴权但无法加请求头的 URL 直接交给播放器。原文查看使用原文块接口，不向客户端开放服务端目录。

## 4. 核心内部接口

接口定义为实现契约，以下仅描述签名与结果，不包含实现代码。

~~~python
class IdentityService:
    async def login(self, code: str) -> AuthResult: ...
    def require_user(self, bearer_token: str) -> UserContext: ...
    def require_consent(self, user: UserContext) -> None: ...

class MaterialService:
    async def import_file(self, user: UserContext, upload: UploadInput, key: str) -> JobView: ...
    def import_text(self, user: UserContext, title: str, text: str, key: str) -> MaterialView: ...
    def get_source(self, user: UserContext, material_id: str, query: SourceQuery) -> SourcePage: ...
    async def prepare_outline(self, user: UserContext, material_id: str,
                              replace: ReplaceOutlineInput, key: str) -> JobView: ...

class DocumentParser:
    def parse(self, source: PrivateFileRef, deadline: float) -> ParsedDocument: ...

class Retriever:
    async def retrieve(self, user: UserContext, material_id: str, section_ids: list[str],
                       query: str, limit: int = 8) -> list[RetrievedChunk]: ...

class CitationValidator:
    def validate(self, user: UserContext, material_id: str,
                 proposal: ModelAnswer, allowed_block_ids: set[str]) -> ValidatedAnswer: ...

class LearningService:
    async def start(self, user: UserContext, data: StartSessionInput, key: str) -> JobView: ...
    async def turn(self, user: UserContext, session_id: str,
                   data: TurnInput, key: str) -> JobView: ...
    def resume(self, user: UserContext, session_id: str) -> SessionView: ...

class TeachingEngine:
    def decide(self, session: SessionState, action: TurnAction,
               proposal: ValidatedTeachingProposal) -> SessionTransition: ...

class VoiceService:
    async def submit(self, user: UserContext, session_id: str,
                     audio: EphemeralAudioInput, expected_revision: int, key: str) -> JobView: ...
    async def synthesize(self, user: UserContext, message_id: str, key: str) -> JobView: ...

class JobService:
    def create(self, user: UserContext, request: JobInput, key: str) -> JobView: ...
    def cancel_chain(self, user: UserContext, job_id: str) -> JobView: ...
    def commit_result(self, lease: JobLease, result: DomainResult) -> CommitOutcome: ...

class DeletionService:
    def request(self, user: UserContext, material_id: str, key: str) -> DeletionView: ...
    async def purge_due(self, now: float) -> list[PurgeResult]: ...

class ModelProvider:
    async def generate(self, request: ModelRequest, deadline: float) -> ModelAnswer: ...

class EmbeddingProvider:
    async def embed(self, texts: list[str], deadline: float) -> list[list[float]]: ...

class ASRProvider:
    async def transcribe(self, wav_bytes: bytes, deadline: float) -> Transcript: ...

class TTSProvider:
    async def synthesize(self, text: str, deadline: float) -> SynthesizedAudio: ...
~~~

UserContext 包含内部 user_id 和 consent_version；不包含模型凭据。所有领域输入输出用 Pydantic 模型定义；路由和内部服务共享请求校验。TeachingEngine 为无网络、无数据库的纯状态转换，便于验证学习逻辑。

## 5. 教学控制与上下文

### 5.1 初学状态

会话 phase 为 preparing_lesson / explaining / awaiting_recall / feedback / completed。持久保存 step_index，和事实型 SectionProgress 分开。

1. 进入小节时，模型基于该节原文生成 key_points 和有顺序的 steps；每步必须关联有效原文范围。讲解步骤不能包含强制测试节点。
2. 生成第一个讲解步骤，显示“继续讲解”和自由提问入口。
3. CONTINUE 只推进一个步骤。ASK、SIMPLIFY、EXAMPLE 解释当前内容，均不推进步骤，也不要求复述。
4. 最后一个步骤完整生成后，由服务端追加固定的整体复述邀请，phase 改为 awaiting_recall，taught=true。
5. RECALL 接收用户复述，模型输出正确内容、遗漏、偏差和解释；引用验证后保存反馈。提交成功的复述记录 recall_completed=true；存在关键遗漏或偏差时 needs_review=true。
6. REPEAT 清零该会话讲解步骤并从当前节重新讲，保留过去复述记录；SKIP_RECALL 不设置 recall_completed；NEXT_SECTION 进入目录中的下一节。
7. 用户可以在没有复述时明确选择下一节，此时记录已讲解未复述；讲解尚未完成就离开时不设置 taught=true。
8. 对同一节再次完成有效复述时可清除本次已修正的 needs_review；其他复习产生的未解决问题仍保留回看提示。

客户端明确操作直接映射到 CONTINUE / REPEAT / RECALL / SKIP_RECALL / NEXT_SECTION / ASK / SIMPLIFY / EXAMPLE。自由文字或语音以 UTTERANCE 提交，模型先输出受当前 phase 限定的意图；不确定时按普通提问处理、保留当前进度，必要时询问用户。意图分类不允许修改资料范围、账号或角色。

### 5.2 复习状态

phase 为 preparing_review / awaiting_answer / awaiting_followup / showing_feedback / completed。

1. 开始一轮时，基于已选小节和角色生成 5 道主问题及私有参考要点；服务端检查恰好 5 题、有来源、主题在范围内、提问中没有直接附上答案。
2. 私有题目计划保存在服务端，客户端每次只能读取当前已发布题目，初始发布第 1 题。
3. ANSWER 生成反馈。模型可建议追问；TeachingEngine 仅在 followup_count=0 且建议合法时发布追问，并把次数设为 1。
4. 回答追问后只给反馈并发布下一道主问题，不能再追加追问。没有追问时直接发布下一题。主问题索引始终为 1–5。
5. SKIP 标记 skipped 并进入下一题；REVEAL 发布参考答案并标记 answer_revealed。之后的作答不再计为独立作答。
6. 第 5 题结束或 END_REVIEW 后，基于已发生记录生成总结，包含已答题目、文字薄弱点及回看小节。正常完成恰好 5 道主问题；提前结束不补造未答内容。
7. 总结生成失败时，保留全部回答及轮次状态，允许重试总结，不重新出题。

课堂追问、复习主问题和初学复述是不同状态事件。不能因为一个自由提问就把复习进度增加，也不能把初学的随口回答当成完成复述。

### 5.3 检索、长资料和目录

- PDF 按页提取，再按段落建立 SourceBlock；TXT、Markdown 和粘贴文字按标题和自然段建立块。纯图片页产生警告，不进行 OCR。
- 保存稳定块 ID 和字符范围。按自然段合并为约 1,000 字符的检索块，超长块切分并保留 100 字符重叠；块内可含多个原文 span，不能丢失来源映射。
- 预览确认后才向模型或向量服务发送正文；prepare_outline 任务先建立所需向量，再生成目录。向量按 content_hash 和模型名在当前资料内复用，目录失败不丢弃已确认的原文。
- text-embedding-v4 单批最多 10 个块、每块低于其 token 上限；采用 1024 维，并验证数量、维度及数值有限性。一个任务内最多并发 2 批。
- 目录生成发送完整的 200,000 字符以内正文及稳定块 ID，利用所选模型的长上下文；服务端先按实际输入长度和模型预算检查，输出要求为带来源范围的有序小节。不得用只覆盖开头的一小段摘要冒充全篇目录。
- 目录输出最长 8,000 tokens；需覆盖所有有正文的连续原文范围，范围允许交叠但不得遗漏尾部。主要主题是否合理仍通过 Spec 指定样例人工核对。
- 目录准备沿用文字模型任务 90 秒总截止时间，包含向量和目录生成；剩余时间不足时失败并提示重试，不无限延长。已存成功向量可在显式重试时复用。
- 日常教学只装入当前小节及按问题检索到的最多 8 个块；若小节本身过长，则按已保存讲解步骤的来源范围取内容，确保不是只检索最相似一句就讲完整节。
- 向量候选先按 owner_id、material_id 和授权范围过滤，再计算精确余弦相似度。不可先全库搜索再在客户端剔除其他用户资料。
- 默认上下文包含系统规则、当前模式与状态、资料与当前节目标、依据块、最近 6 轮已完成对话。更早历史从存储按明确引用回取，不整段无限追加。
- 对完全超出资料的问题，生成“资料未涉及”块和明确的“AI 补充”块；检索不到依据不当作知识可靠性的证明，不编造外部来源。对于需要资料支持却未找到来源的回答，提示无法从当前提取内容确认。

### 5.4 模型输出校验

模型按 JSON schema 返回 blocks、intent、feedback、followup_proposal 等候选字段。Pydantic 使用严格字段和禁止未知字段设置；所有状态改变由 TeachingEngine 在当前允许动作内决定。

拒绝不存在的来源 ID、跨资料引用、错误 quote、未标识的来源外回答、超出 5 题或追加第二次追问。允许在同一总截止时间内进行最多 1 次结构修复请求；仍失败则本任务失败，保留此前状态。

模型没有文件系统、联网搜索、数据库查询或任意工具执行能力。检索内容放入明确标注为不可信学习资料的区域，和控制规则、用户动作分开。凭据永不放入上下文，其他账号数据在进入上下文前已被排除。

## 6. 语音流程

### 6.1 录音与转写

1. 只有用户点击录音才请求麦克风；拒绝时保留文字入口。
2. RecorderManager 计划参数为 duration=120000、sampleRate=16000、numberOfChannels=1、format=aac、encodeBitRate=48000。从 onStart 计时，110 秒显示即将结束提示；onStop 统一处理手动结束和达到上限的自动结束，防止双重提交。
3. 点击取消、录音出错或切到后台时停止录音并丢弃，不自动发送；清理客户端临时文件。重新进入不自动录音。
4. 提交前停止当前 AI 播报。后端限制原始音频体积 4 MiB，使用 ffprobe 核验有效音轨、实际格式和时长，不能信任 client_duration_ms。
5. 通过 ffmpeg 在内存临时目录转为单声道、16 kHz、16-bit PCM WAV，转换后总时长不超过 120 秒；正常文件约 3.84 MB，Base64 后约 5.12 MB，低于所选转写接口的 10 MB 输入边界。
6. 录音字节保存在 API 进程内的受限临时池及 tmpfs 转码目录，任务表只记录随机引用。原始音频、WAV 和 Base64 在成功、失败或取消后的 finally 阶段清理，不写入资料目录、数据库或日志。单机异常退出后由启动清扫删除 tmpfs 残留。
   multipart 的临时溢写目录同样必须配置到该 tmpfs，不能让框架默认写到持久 /tmp；生产启动时检查配置，不满足时拒绝开启语音功能并给出服务不可用提示。Docker 配置提供至少 128 MiB 的内存临时空间；音频文件不放入持久卷。
7. 转写使用完整音频和中文配置；通过音频有效性检查、基础静音检测及非空结果检查后，才建立用户消息。已知静音样例不得交给教学模型生成回答；检测不能宣称识别所有嘈杂环境中的无效语音。
8. 一次语音提交预留学习在途名额；转写成功后在事务中把名额交给后续教学任务，自动继续，不要求用户再次发送。转写 60 秒与教学文字 90 秒是连续的独立阶段，各自显示状态。
9. 服务重启时，丢失录音临时引用的任务标记 AUDIO_REUPLOAD_REQUIRED，不用空音频或旧转写伪装恢复。

转写、转换和内存等待共用 60 秒截止时间；网络上传单次请求仍有 15 秒超时，失败可重录。客户端清理临时文件是应用行为，不承诺操作系统层面的取证级擦除。

### 6.2 合成与播放

1. 文本回答完整提交后，客户端按自动播报设置请求 speech；关闭自动播报不会发起这次合成请求。
2. 把正文按句末标点切分为每段不超过 500 个 Unicode 字符，给供应商 600 字符限制留出余量。段落不可被静默省略；去掉仅供阅读的来源标记，不去掉实质讲解内容。
3. 服务端为各片段调用非实时 TTS，最多并发 2 个片段，在本条回复的 60 秒准备期限内取得全部片段；验证返回内容是合法音频后写入私有缓存，再向客户端发布就绪清单。
   客户端首次下载及播放器就绪也受这次 60 秒总等待约束，按剩余时间限制下载；即使服务端已完成，客户端仍未准备好播放也应明确提示语音暂不可用，不重置成另一个 60 秒等待。
4. 单条教学回复的语音正文目标不超过 1,200 字符；更长知识点按讲解步骤自然分条。对模型返回的更长有效回复仍分段合成，不能截断文字；若无法在 60 秒内准备完成则提示语音失败并保留全部文字。
5. TTS 返回的供应商 URL 仅服务端使用；只允许已配置供应商的下载域名、限制重定向和响应体积，不允许模型或用户指定任意抓取 URL。
6. 客户端下载私有片段后用一个 InnerAudioContext 依次播放；停止时中止后续播放和未完成下载。每次播放使用递增 generation_id，旧下载完成也不能自动接着播放。
7. 重播命中缓存时不再次合成；缓存过期可重新生成，保留当前语音任务幂等关系。成功音频默认缓存 24 小时，到期清理；删除资料时立即失去访问权限并纳入清理。
8. 部分片段失败时清理本次不完整资产，显示“文字回复已保留，语音暂不可用”；重试只做语音，不再调用教学模型。
9. 接听系统电话、切后台或播放器出错时停止音频；首版不做后台播报。

微信开发者文档抓取存在访问限制；录音参数参考腾讯云官方微信集成示例和微信官方类型仓库，并保留 Android/iPhone 真机验收。不得把参数设计视为已通过真机验证。

## 7. 任务、取消与一致性

### 7.1 截止时间和调度

| 任务 | 总上限 | 成功提交点 | 失败后的保留内容 |
|---|---:|---|---|
| parse | 120 秒 | 完整原文块、警告及预览状态 | 原资料、错误状态，可重新解析或删除 |
| prepare_outline | 90 秒 | 完整合法目录；原子更新 active_outline_id | 原文、已成功向量、旧目录和旧会话 |
| lesson/start/turn/review/summary | 90 秒 | 合法回复、题目事件及进度的同一事务 | 有效用户消息和之前已完成的历史，不提交半段 AI 内容 |
| asr | 60 秒 | 有效转写消息与教学子任务 | 先前历史；录音清除，失败需重录 |
| tts | 60 秒 | 完整片段清单可下载 | 文字回复，清除未完成合成资产 |
| deletion | 24 小时 | 项目持有的关联内容清除并记录证据 | 仅最小删除元数据，不恢复用户访问 |

上限从服务端接受该阶段任务计时，含排队、外部调用和结构修复。客户端根据 deadline_at 显示剩余等待；服务端每秒扫描过期任务，测试使用可控时钟验证边界。上传本身使用独立短请求上限，未完成上传不创建可学习资料。

调度器对每个用户执行 material/learning/asr 名额限制；语音链同时占用相应学习名额。全局初始上限为 2 个教学／目录任务、2 个 ASR 任务、2 个 TTS 任务。录音内存池最多 4 个待处理对象，超过则拒绝接收并提示稍后重试。TTS 对同一消息不重复启动。

### 7.2 幂等与取消

- 幂等标识在用户点击提交时生成，网络重试复用；服务端同时保存规范化请求摘要。重复请求返回同一个 job_id。
- 在 BEGIN IMMEDIATE 短事务中创建任务并占用名额；提交外部请求时不持有写事务。成功时再次开启事务，核对 job 状态、deadline_at、资料未删除、expected_revision。
- 回复、引用、问题事件、进度、任务成功状态一起提交；任何核验失败全部不提交。完整持久化回复可以在恢复时读取，不把流式半段内容作为已完成结果。
- 取消先持久化 cancelled 并取消子链，再尽力停止网络请求或子进程。无法撤回的外部服务处理可能已产生费用，但结果不得更新当前会话或播放。
- 会话切换、页面离开和手动取消均增加客户端 generation_id，停止自动播报及旧轮询。已提交的完整历史可在恢复时重新读取；客户端单纯进入后台不自动建立新题目。
- 任务已成功时取消返回“已完成”，不能假装撤销已提交的学习记录；前端仍停止声音。消息不会因停止声音而被删除。
- expected_revision 不匹配返回 SESSION_CHANGED，要求刷新会话，不把旧回答应用到新小节。
- 网络超时后先查询原任务，不自动新建收费请求；明确失败后才让用户选择重试。原任务已成功则返回已有结果。
- 重启后将正在执行、结果不确定的外部调用标记 INTERRUPTED，不自动重复收费；queued 的持久资料／文字任务仅在截止时间内可恢复。录音引用不存在时一律要求重录。

### 7.3 删除

确认删除时事务内标记 material.deleted_at，封锁所有下游查询，取消关联任务并建立 deletion。正在返回的任务必须在提交前再次检查删除标记。

清理器每分钟执行一次，依次删除音频与原文件、目录与进度、消息与题目、块与向量，再删除资料内容；失败按 1、5、15、60 分钟退避重试，距离 24 小时不足时提高重试优先级。完成时记录分类计数和时间。

配置 SQLite secure_delete，短事务结束后不保留正文副本；缓存与客户端本地临时内容也按资料关联清理。首版不创建脱离此清理流程的正文备份。这里的“清除”指项目正常读取及存储对象已删除，不承诺存储硬件的物理不可恢复。

验收需真实记录删除前后对象数量及文件存在性。24 小时清理不能仅凭代码检查判定通过；运行清理并核对时间证据。

## 8. 配置、部署与访问控制

### 8.1 服务端配置

| 配置 | 说明 |
|---|---|
| APP_ENV | dev/test/production。生产禁止测试身份和模拟模型。 |
| PUBLIC_BASE_URL | 真机可访问的 HTTPS API 地址。 |
| WECHAT_APP_ID、WECHAT_APP_SECRET | 微信身份交换所需；AppID 可写入小程序项目配置，AppSecret 只在服务端。 |
| DASHSCOPE_API_KEY、DASHSCOPE_WORKSPACE_ID | 项目方百炼账号配置，禁止进入客户端、源码仓库或文档示例。 |
| DASHSCOPE_COMPATIBLE_BASE_URL、DASHSCOPE_NATIVE_BASE_URL | 项目业务空间的兼容接口及原生语音接口地址；支持配置，不把单个文档示例域名硬编码为所有账号通用值。 |
| TEXT_MODEL、EMBEDDING_MODEL、ASR_MODEL、TTS_MODEL | 默认使用本计划模型名；每次调用记录所用模型名，不记录正文。 |
| DATA_DIR | 受限持久目录，仅运行用户可读写。 |
| AUDIO_TMP_DIR | Linux tmpfs 目录，不属于持久卷；启动时检查与清扫。 |
| CONSENT_VERSION | 当前隐私说明版本；不同意时禁止处理资料与音频。 |

限制和时限集中定义为服务端常量，并通过受认证配置接口或会话响应告知前端，避免录音显示 120 秒而服务端仍按 60 秒拒绝。前端不拥有放宽限制的权限。

本地开发使用开发者工具连接开发服务；真机正式体验采用 HTTPS 和微信平台要求的 request/uploadFile/downloadFile 合法域名配置。微信 AppID、录音隐私能力配置、体验成员和账号资格由项目方在其账号下准备；公众发布不在本版交付范围。平台要求可能随账号和版本变化，接入时核对控制台实际要求。

### 8.2 测试与生产隔离

测试身份和模拟服务通过依赖注入只在测试进程装配，没有客户端可调用的“测试登录”或通用模拟开关。生产发现缺少必需配置时明确失败，不启用演示回答。

FastAPI 入口仅监听反向代理后的服务端端口；上传与下载经过鉴权。反向代理不记录带正文的请求，也不把私有目录映射成静态目录。会话令牌和供应商下载 URL 均不写入访问日志。

首版只支持单服务进程和同机持久盘；启动多 worker 或无持久卷部署会违反本计划。后续若需要水平扩展，先另行设计数据库、共享对象存储和任务队列迁移。

### 8.3 当前环境的验证边界

当前接续环境为 Windows PowerShell。T1 已建立项目内 Python 3.12 虚拟环境并锁定实际依赖，使用 Node 构建客户端；ffmpeg、微信工具、双真机和真实服务需在对应任务实际检查，不沿用原设计环境的可用性假设。

最终开发验收须分别报告：
- 本地自动化已运行部分；
- 已连接真实服务的部分；
- 开发者工具及两台真机实际观察部分；
- 因条件缺少仍未验证的部分。

缺少 AppID 或服务凭据不妨碍先完成源码、配置模板、测试和本地启动说明；不得因此把真实集成项目标为通过。若缺少依赖安装条件，开发时先记录具体错误和可用替代验证，不虚构测试成功。

## 9. 项目文件组织

下列均为计划新建文件，当前只有需求与设计文档。实际文件清单在 Task 阶段逐项确认。

| 目录 | 文件 | 职责 |
|---|---|---|
| 项目根目录 | README.md、.gitignore、.env.example、Dockerfile、compose.yaml | 启动、交付、秘密排除、单实例与持久目录配置。 |
| docs/v1/ | spec.md、plan.md、task.md、checklist.md | 四阶段基线文档；开发时按实际状态维护。 |
| docs/ | setup-wechat.md、verification.md、privacy-notice.md | 微信配置、真实验证记录和用户隐私说明。 |
| backend/ | pyproject.toml、requirements.lock | Python 依赖与开发时确定的可复现版本。 |
| backend/app/ | main.py、config.py、contracts.py、errors.py | 入口、配置、公共契约、错误边界。 |
| backend/app/storage/ | database.py、repositories.py、files.py、ephemeral_audio.py | 事务、带用户范围的查询、私有文件和临时音频。 |
| backend/app/migrations/ | 001_initial.sql | 上述数据表、索引、约束及迁移版本。 |
| backend/app/api/ | auth.py、materials.py、sessions.py、jobs.py、speech.py | HTTP 路由、所有权和隐私依赖检查。 |
| backend/app/services/ | identity.py、materials.py、learning.py、voice.py、deletion.py | 领域流程与事务协调。 |
| backend/app/learning/ | engine.py、intents.py、context.py、prompts.py、citations.py | 两种模式控制、意图、上下文、提示词和来源校验。 |
| backend/app/documents/ | parser.py、parser_worker.py、chunking.py、retrieval.py | PDF 子进程提取、TXT/MD、块映射和向量检索。 |
| backend/app/providers/ | base.py、wechat.py、qwen_text.py、qwen_embedding.py、qwen_asr.py、qwen_tts.py | 独立外部服务适配器与统一错误。 |
| backend/app/tasks/ | scheduler.py、lifecycle.py、cleanup.py | 任务调度、取消与截止时间、启动与周期清理。 |
| backend/tests/ | test_auth.py、test_documents.py、test_learning.py、test_retrieval.py、test_voice.py、test_jobs.py、test_deletion.py、test_api.py | 标准库 unittest 自动化；通过真实领域入口与 HTTP 入口验证行为。 |
| backend/tests/fixtures/ | source.txt、source.md、known-facts.json、injection.txt、make_fixtures.py | 教学事实、注入样例及可重复生成的小型 PDF／WAV 样例；生成器只供测试，不代替真实服务。 |
| miniprogram/ | app.ts、app.json、app.wxss、project.config.json、tsconfig.json、package.json、package-lock.json | 小程序入口、页面注册、类型与构建配置。 |
| miniprogram/types/ | contracts.ts | 客户端公共视图与请求类型，匹配服务端契约。 |
| miniprogram/services/ | api.ts、auth.ts、materials.ts、sessions.ts、jobs.ts、recorder.ts、playback.ts | 网络、资料、会话、轮询、录音和播放封装。 |
| miniprogram/state/ | learning.ts、voice.ts | 页面可观察状态、generation_id、按钮和互斥控制。 |
| miniprogram/pages/library/ | index.ts、index.wxml、index.wxss、index.json | 资料与继续学习。 |
| miniprogram/pages/import/ | index.ts、index.wxml、index.wxss、index.json | 文件和文字导入。 |
| miniprogram/pages/preview/ | index.ts、index.wxml、index.wxss、index.json | 原文预览和确认。 |
| miniprogram/pages/outline/ | index.ts、index.wxml、index.wxss、index.json | 小节、模式和复习范围。 |
| miniprogram/pages/study/ | index.ts、index.wxml、index.wxss、index.json | 教学／复习对话、语音与动作。 |
| miniprogram/pages/source/ | index.ts、index.wxml、index.wxss、index.json | 来源原文定位。 |
| miniprogram/pages/history/ | index.ts、index.wxml、index.wxss、index.json | 历史和复习总结。 |
| miniprogram/tests/ | recorder.test.ts、playback.test.ts、learning.test.ts、jobs.test.ts | 可控微信适配层和假时钟，验证限时、取消和页面状态，不冒充真机。 |

不新增 Web 网站、公开产品页面或独立运营后台。

## 10. 技术决策与取舍

| 决策点 | 本版选择 | 理由与限制 |
|---|---|---|
| 原生还是跨端 | 原生小程序 | 已确定只做微信；减少跨端兼容和语音封装问题。 |
| Python 还是全栈 JavaScript | Python 后端＋TypeScript 小程序 | 资料解析和模型服务容易分开验证；客户端保留静态类型。 |
| SQLite 还是 PostgreSQL | 单机 SQLite | 小范围首版部署少一个服务；代价是单写入者、无水平扩展，明确不支持多实例。 |
| 精确向量还是独立向量库 | 文本块向量随 SQLite 保存，应用内精确排序 | 单资料数百块足够简单；后续资料量或并发增加再迁移，当前不引入 pgvector。 |
| 临时后台任务还是完整队列服务 | 持久任务状态＋单进程异步调度 | 支持进度、取消、幂等；重启时不擅自重放结果不确定的收费调用。 |
| 全文问答还是 RAG | 目录读取全篇；日常教学按小节和问题取原文 | 避免每一问重发整份资料，保留章节覆盖和引用。 |
| 自由 Agent 还是明确流程 | 受约束的状态转换＋模型候选输出 | 题数、追问次数和复述时机可被确定性验证，减少模型自行改变流程。 |
| 语音是否与教学一次生成 | ASR、文字教学、TTS 三阶段 | 语音可单独失败或重试；文字历史保持可靠。 |
| 是否公开存储录音 | 不公开、不持久化；转码后 Base64 输入 | 满足隐私范围；代价是进程重启后必须重新录音。 |
| 长回复如何播报 | 500 字符片段、完整准备后依次播放 | 避开单次 600 字符限制；准备超时保留文字，不截断内容。 |
| 输出是否流式显示 | 首版对客户端返回完整校验后的文字 | 便于引用检查与原子进度；短任务轮询显示明确等待，后续再考虑流式体验。 |
| 是否训练模型 | 不训练、不微调 | 通过资料上下文、状态规则和验收样例完成首版。 |

## 11. 费用估算

### 11.1 官方计费基线

以下为 2026-11-16 查到的百炼华北2（北京）公开原价；不计免费额度、促销、缓存折扣和税费差异。实际账单以项目账号、模型及地域为准。

| 服务 | 估算单价 | 约束与官方依据 |
|---|---:|---|
| qwen-plus 非思考，单次输入不超过 128k tokens | 输入 0.8 元／百万 tokens；输出 2 元／百万 tokens | 大于 128k 时按更高输入档计费，不能沿用短上下文单价。[模型价格与档位](https://help.aliyun.com/zh/model-studio/qwen-plus) |
| qwen-plus，128k–256k 输入档 | 输入 2.4 元／百万 tokens；输出 20 元／百万 tokens | 目录读取长资料时可能进入此档，依据同上。 |
| qwen-plus，大于 256k 输入档 | 输入 4.8 元／百万 tokens；输出 48 元／百万 tokens | 长资料生成目录须按实际输入量记录成本，依据同上。 |
| text-embedding-v4 | 输入 0.5 元／百万 tokens | 资料建索引与每次语义检索查询。[官方价格](https://help.aliyun.com/zh/model-studio/text-embedding-v4) |
| qwen3-asr-flash | 0.00022 元／音频秒 | 单段完整 120 秒约 0.0264 元。[官方价格](https://help.aliyun.com/zh/model-studio/qwen3-asr-flash) |
| qwen3-tts-flash | 0.8 元／万字符 | 播报 300 字符约 0.024 元。[官方价格](https://help.aliyun.com/zh/model-studio/qwen3-tts-flash) |

### 11.2 一次学习的设计估算

这只是带明确输入量的算例，不是实际已测账单或费用保证：

- 假设每轮讲解／评价累计输入 8,000 tokens、输出 600 tokens，且处于 128k 内的非思考档：约 0.0076 元。
- 自由语音的额外意图分类按输入 500、输出 100 tokens：约 0.0006 元。
- 用户每轮录音 30 秒：约 0.0066 元。
- AI 每轮播报 300 字符：约 0.024 元。
- 合计每轮约 0.039 元，20 轮约 0.78 元，另加检索查询、开场计划和目录费用。
- 若 20 轮每轮都录满两分钟，其他输入量不变，约 1.17 元。
- 例如一次目录调用实际输入 200,000 tokens、输出 5,000 tokens：约 0.58 元；对 200,000 tokens 正文建向量索引约 0.10 元，重叠块会略增加输入量。20 万字符不等于 20 万 tokens，不能混用。
- 结构修复、用户重试、重新合成和更长回复可能增加费用；取消请求不保证供应商不收费。UsageEvent 用来记录真实用量，完成真实联调后再更新估算。

服务器、HTTPS 域名、微信账号相关费用未包含在上述推理算例中。部署供应商和具体套餐尚未购买，不能把算例当作项目月账单。当前设计可先在本地运行；真机体验需要可访问的服务地址。Plan 审批是对服务选择和计费方式的确认，任何实际购买或账户开通另由项目方进行。

## 12. 验证策略和需求覆盖

### 12.1 验证分层

1. 自动化：使用 unittest＋FastAPI TestClient 验证鉴权、输入限制、资料解析、引用、学习规则、任务、取消与删除；测试服务只装配在测试进程。
2. 小程序逻辑：类型检查和 TypeScript 测试验证录音 120 秒、单次提交、过期结果丢弃、分段播放、停止与页面状态。
3. 真实外部服务：对预设事实资料分别验证目录、讲解、误解纠错、老师／面试官、转写和合成；记录模型名、调用时间、用量、已脱敏结果。
4. 微信端到端：开发者工具和两台真机完成 Spec AC17、AC19，记录设备与版本、按钮位置和实际语音体验。
5. 生命周期：使用可控时钟测试 60／90／120 秒边界；实际运行删除清理并核对 24 小时内完成的时间证据。模拟截止时间测试与真实删除运行证据分别记录。

### 12.2 覆盖矩阵

| 需求 | 本计划机制 | 对应验收 |
|---|---|---|
| F1 | 微信身份交换、摘要令牌、服务端所有权过滤 | AC1 |
| F2 | 上传校验、Unicode 正文计数、UTF-8 严格解码、粘贴入口 | AC2、AC18 |
| F3 | 逐页／段落提取、原文预览、明确解析错误和警告 | AC3 |
| F4 | 完整正文目录、来源范围覆盖、旧目录内部保留与替换确认 | AC4 |
| F5 | 保存讲解步骤、完成小节才邀请复述、三类反馈和跳过事实 | AC5、AC6 |
| F6 | 私有 5 题计划、当前题公开视图、一次追问限制和真实轮次总结 | AC7、AC8 |
| F7 | 120 秒手动录音、自动结束提交、转写后教学链、文字入口 | AC9 |
| F8 | 独立 TTS、500 字符分段、鉴权下载、播放互斥和重试 | AC10 |
| F9 | 原文块 ID、逐字引用检查、来源类型标签和原文查看页 | AC11 |
| F10 | 会话快照、事实进度、模式独立、原子结果与恢复 | AC6、AC8、AC12 |
| F11 | 删除确认、立即封锁访问、关联清理与时间证据 | AC13 |
| N1 | 隐私前置确认、所有权、凭据仅服务端、录音临时池和日志脱敏 | AC1、AC9、AC13、AC14 |
| N2 | 不可信资料区、无自由工具、严格引用和真实事实样例核对 | AC11、AC15、AC19 |
| N3 | JobView、截止时间、幂等、取消链和 revision 检查 | AC16 |
| N4 | 原生微信界面、竖屏布局、两台真机和文字兜底 | AC17 |
| N5 | 大小／长度／时长限制、在途名额、缺配置明确报错、用量元数据 | AC2、AC16、AC18 |

本阶段检查结果：16 条 F/N 需求均有设计对应；19 条 AC 均有设计及验证入口。技术方案没有改变已批准的 120 秒录音上限、两种模式、资料类型或教学补充边界。外部服务和真机尚未联调，本文件不包含“已实现”或“已通过端到端”的结论。

## 13. 官方参考与尚需实测的边界

所有服务接入的输入输出与价格仅引用官方来源；实现时把实际依赖版本锁定在项目文件中，不无条件跟随 SDK 最新版本。

- [腾讯云官方微信录音集成示例](https://cloud.tencent.com.cn/document/product/269/75316)：RecorderManager 与 AAC 录音参数参考。
- [微信官方 API 类型仓库](https://github.com/wechat-miniprogram/api-typings)：原生 TypeScript 能力定义参考；不代替真机权限检查。
- [百炼非实时语音识别指南](https://docs.modelstudio.console.alibabacloud.com/zh/model-studio/non-realtime-speech-recognition-user-guide)：所选 ASR 支持完整短录音、音频格式和时长限制。
- [百炼 Qwen-ASR API](https://help.aliyun.com/zh/model-studio/qwen-asr-api-reference)：Base64 Data URI、输入限制与业务空间域名配置。
- [百炼 Qwen-TTS API](https://help.aliyun.com/zh/model-studio/qwen-tts-api)：单次 600 字符、音频结果和下载有效期。
- [百炼向量同步 API](https://help.aliyun.com/zh/model-studio/text-embedding-synchronous-api)：向量维度、单批行数及单行输入上限。
- [SQLite 事务文档](https://www.sqlite.org/lang_transaction.html)：单写入者与短事务并发约束。
- [pypdf 文本提取文档](https://pypdf.readthedocs.io/en/stable/user/extract-text.html)：逐页文本提取及无法代替 OCR 的边界。
- [FastAPI 后台任务文档](https://fastapi.tiangolo.com/tutorial/background-tasks/)：进程内后台处理的适用范围；本项目额外保存任务状态以支撑取消及恢复。

仍需实测的关键边界是中文 PDF 提取质量、真实教学反馈、两分钟 AAC 真机录音、语音片段播放、供应商额度和微信账号可用能力。设计阶段已经给出实现和验证路径；这些条目必须等实际接入后才能报告通过。

## 14. 审批节点

请审阅架构、接口、两种教学模式的控制方式、外部服务选择、费用算例和运行前提。

本 Plan 批准后才进入 task.md 任务拆解；Task 批准后生成 checklist.md。四份文档全部批准后，再开始项目实现。


## 2026-11-17 接续说明

原文档末尾审批提示保留为历史阶段记录。用户已明确要求读取现有四份文档并从 T1 开始开发，本次按该授权进入实现，不重复请求批准；需求与设计范围不变。当前进度和实际证据以 task.md 及 docs/verification.md 为准。
