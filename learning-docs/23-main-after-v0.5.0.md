# v0.5.0 之后官方 main 的主要提交（2026-10-22）

## 范围

通过 `git ls-remote https://github.com/volcengine/OpenViking.git refs/heads/main refs/tags/v0.5.0` 核实远程位置，再读取本地已有提交对象；未切换分支或合并代码。

- 基线：v0.5.0 = `20f9c11362ad16193b0daa4a414e1095061af427`，2026-10-09。
- 截止：官方 main = `4c29ed7ff0859c991f1184f0af30973f3d679840`，2026-10-22。
- `v0.5.0..4c29ed7ff` 共 210 个提交，全部为非 merge 提交。
- 本地 main 为 `f34a54fd1`，包含合并官方 main 的提交；本报告排除该个人 fork 合并提交。
- v0.5.1 = `e855796c8`，2026-10-19，是当前官方 main 祖先；其后尚有 59 个 main 提交。
- v0.5.1.1 解引用到 `c2dea8005`，2026-10-20，不是该 main 的祖先。main 中另有同题修复 `35b51da9a`；不能用热修复标签是否为祖先推断修复是否存在。

## 主要演进

### 1. Compile 从一次性生成走向持续更新

- `7967eca58` / #5750：可规划 Map/Shuffle/Reduce 编译流水线，按来源处理、分组路由与归并生成产物。
- `65e86ccb8` / #6304：增量更新、调度与版本历史。
- `a9c21c2fd` / #6133：编译账号锁隔离与卡住的 review 循环处理。

源码定位：`openviking/service/compile_service.py:323` 配置增量生命周期，`:425` 提交定时增量任务；`openviking/service/compile_scheduler.py:106` 为调度实现。

意义：对知识库、SOP 和 Wiki 的价值从单次生成扩展到后续维护；并不等于可无验证直接用于生产知识维护。

### 2. 账号与系统资产更加独立

- `4c29ed7ff` / #6374：Account 级 system 目录和内置资产同步。
- `90df14547` / #5933：账号存储 provider 重构。
- `6cf676788` / #5955：Redis + MySQL 账号存储。
- `ce42a166b` / #6396：trusted group header。

源码定位：`openviking/core/namespace_policy.py:10` 定义 `viking://system`；`openviking/system_assets/sync.py:27` 同步打包资产；`openviking/system_assets/reconcile.py:19` 按账号协调同步；`openviking/server/account_stores/redis_mysql.py:162` 为 RedisMySQLAccountStore。

意义：对部门部署，账号持久化、统一身份组与内置资产生命周期比单机试玩更重要。但 system 资产机制不应直接等同于任意部门自定义 skill 的自动推送。

### 3. 后台任务与内容写入更强调可恢复性

- `ec56b9443` / #6043、`013e988bb` / #6136：统一 task 与 queue work 的跟踪和契约。
- `c01bb34d8` / #6258：MySQL task work store。
- `8540d3209` / #6352：持久化内容写入任务流程。
- `ff790205a` / #5741：multi-write v2 元数据分区。
- `c90dae83d` / #5897：batch write RNFV。
- `b0d1d633f` / #4548、`c89efb7c3` / #6300、`2e4936b7a` / #6424：锁等待、提前消费导致锁泄漏、释放顺序等问题。

源码定位：`openviking/storage/content_write.py:305` 的 write 入口，以及 `:431` 的持久化 root task 入队逻辑。

意义：多实例、进程恢复和大批量写入能力在继续建设；这些提交是可靠性改进的证据，不是经过生产规模验证的保证。

### 4. 会话与记忆提取闭环更统一

- `89fbcb968` / #6046：Redis 支持的分布式 idle auto-commit 调度。
- `d1984f16e` / #6131：删除独立 extract，统一 commit。
- `84cc2b412` / #6120：精简提取提示与输入去重。
- `b281505a6` / #6332：Phase 2 与 live session 文件解耦。
- `c0e35a54c` / #6398：临时读取失败时重试，不重新提取。
- `c973ad1c4` / #6104：patch merge 遵守 thinking 配置，并修复提取预算耗尽后的恢复。
- `3266cd865` / #6429：跳过不能修复的 event resolution 重试。

意义：重点是减少重复提取、读写竞争和无效重试，而非新增一种用户可见的记忆类型。

### 5. 模型网关继续完善部署、接入和观测

- `8a74caf5b` / #5960：支持嵌入 server，区分管理访问。
- `9baf83f04` / #6100、`8483463c4` / #6101：精确上游 URL、Azure、URL 内 key。
- `c79600ae5` / #6118：用实际模型生成验证上游。
- `26fa00b91` / #6026：减少每请求 CPU 成本。
- `27db44123` / #6267：多种上游缓存 usage 读取与缓存缺失提示。
- `e855796c8` / #6295：DSH 与无工具旁路请求引发的重复召回、新建会话修复。
- `a8bd02bac` / #6334：按 Codex request_kind 分类，避免元数据关键词误判。
- `8640600f4` / #6333：允许运维修改客户端识别规则。

源码定位：`openviking/server/gateway_host.py:23` 构造嵌入网关；`openviking_gateway/protocols.py:372` 为请求分类入口。

注意：本区间是网关演进，不是网关首次出现。不能把已有的完整网关能力都归为 v0.5.0 之后新增。

### 6. 插件的捕获与协议兼容性持续修复

- `400b70eee` / #5254：DSH 在 compaction/start 提交 pending 消息。
- `7355b926e` / #5660：Codex PreCompact 保持规范 session identity。
- `15c9a339a` / #6296：删除 Codex PreToolUse 上下文注入，避免 function_call 与 output 之间插入 developer 消息导致部分 Responses 上游 400。
- `621b1ddd9` / #5937：captureFilters 替换规则覆盖 tool input/output。
- `f8d36722a` / #5689：刷新 pending replay claim 时间戳。
- `d7b32ac5b` / #6394：Claude Stop 重放 pending writes。
- `6006fc340` / #6092：新增 ov-resources skill。

源码定位：`examples/codex-memory-plugin/hooks/hooks.json:3` 当前 hook 配置；DSH 逻辑位于 `examples/dsh-memory-plugin/runtime.mjs`。注意仓库 examples 的修改不自动代表当前安装的插件包已更新。

### 7. 检索、模型与外部框架

- `855d0928a` / #6128：多模态混合查询和 AML payload。
- `8d52018cf` / #5833：按 document ID 对齐 VikingDB rerank score。
- `938ef31f6` / #6007：约束会话检索意图分析。
- `5680dac89` / #5982：避免已服务目录使后来新增文件被排除。
- `b15426b28` / #5950、`acd7caf0d` / #6047：统一模型重试策略、错误分类与重试边界。
- `7f69dd24c` / #5578：Microsoft Agent Framework context provider。
- 多个 LangChain 修复覆盖文本块、角色/工具历史、namespace discovery 与 Store 语义。

### 8. Studio 与运维

- `4c9a755bf` / #5730：启动配置编辑和 ROOT restart。
- `bd26e83b2` / #6066：workspace-first setup 与双 provider 体验。
- `0a2578e7e` / #6308：连接切换与账号身份核验。
- `ee07304d9` / #6413：文件管理与 Markdown 预览。
- `1c4399988` / #6337、`2b77e30aa` / #6341：隐私配置树与相关 MCP 工具。

## 升级前需要单独检查的兼容性变化

| 提交 | 改动 | 自定义接入的检查点 |
| --- | --- | --- |
| `df2b6df27` / #5947 | trusted 强制 root API key | 旧的无 key trusted、localhost 豁免不能继续假定存在。 |
| `4496edb55` / #6048 | 移除等待专用 timeout | 检查旧 REST/SDK 参数；不表示全部 HTTP 网络超时设置都被移除。 |
| `d1984f16e` / #6131 | 移除 `/sessions/{id}/extract` | 改为 commit，并跟踪返回 task ID。 |
| `621b1ddd9` / #5937 | 替换规则覆盖工具输入输出 | 检查脱敏是否按预期影响工具轨迹；d/k 规则仍基于文本判断。 |
| `6c1107527` / #6011 | 顶层 service_name 统一命名，删除部分 namespace/cache_key_prefix | 核对 Redis/cache/队列/锁隔离配置，不能照搬旧配置。 |
| `8a74caf5b` / #5960 | 嵌入式网关与管理访问分离 | 核对原有 sidecar 配置、访问地址和管理 key。 |

## 对当前目标的优先级

1. 直接相关：Codex/DSH 捕获修复、网关重复召回与分类修复、会话提取闭环。
2. 部门部署：账号与 task store、trusted 身份契约、service_name、锁与内容写入恢复。
3. 持续知识维护：Compile 增量、调度、版本历史与 system 资产。

结论是 main 明显在向可持续运行的上下文基础设施推进；不据提交数量推断应该立即上生产。

验证边界：阅读 Git 对象、提交 diff 与当前源码，未执行升级或测试。Graphify `explain sync_system_assets` 返回无匹配节点，没有为这些新模块提供图谱证据；未重建索引。以下完整清单来自同一固定范围的 git log。

## 完整提交清单

```text
4c29ed7ff 2026-10-22 feat(system): 建立 Account 级 system 目录与资产同步 (#6374)
3266cd865 2026-10-22 fix(memory): skip unfixable event resolution retries (#6429)
c973ad1c4 2026-10-22 fix(session): respect configured thinking in patch merge and repair budget-exhausted extraction (#6104)
d7b32ac5b 2026-10-22 fix(claude-plugin): replay pending writes on Stop (#6394)
e2966201c 2026-10-22 fix(claude-plugin): capture only new turns when a subagent stops again (#6323)
e63eb0d84 2026-10-22 fix(hermes-plugin): stop list recall from repeating the session-start profile (#6397)
15c9a339a 2026-10-22 fix(codex-plugin): stop injecting PreToolUse context between a tool call and its output (#6296)
97c0ceb77 2026-10-22 fix(memory): allow omitting optional supersedes field (#6421)
65e86ccb8 2026-10-22 feat(compile): add incremental updates, scheduling, and version history (#6304)
2e4936b7a 2026-10-22 fix(pathlock): 调整 pathlock release 顺序，先执行内存 Registry lease 的移除，再删除 pathlock 文件 (#6424)
6227af510 2026-10-21 fix(studio): preserve calendar date ranges across daylight saving (#6419)
ee07304d9 2026-10-21 feat(studio): improve filesystem management and Markdown previews (#6413)
f754fa90f 2026-10-21 fix(session): accept string edits in replace mode (#6286)
cffe101ca 2026-10-21 fix(memory): render existing objects as context records (#6347)
8540d3209 2026-10-21 feat(storage): add durable content write task workflow (#6352)
401c1f7ef 2026-10-21 fix(pi): capture divergent sibling branches (#6381)
b5a389245 2026-10-21 fix(embedding): support stable Gemini Embedding 2 (#6383)
3edc0b7f7 2026-10-21 feat(plugins): forward an event time-decay window from recall config (#6390)
f8d36722a 2026-10-21 fix(plugins): refresh pending replay claim timestamps (#5689)
82882c26e 2026-10-20 fix(ci): preserve resource failure details and update import tests (#6340)
7355b926e 2026-10-21 fix(codex-plugin): preserve canonical PreCompact session identity (#5660)
d9db19ab6 2026-10-20 fix(gateway): tolerate non-string "type" values in client history (#6306)
6ec6116b7 2026-10-21 fix(langchain): preserve Store reads and namespace discovery (#6342)
c0e35a54c 2026-10-21 fix(session): retry transient memory reads without re-extraction (#6398)
ce42a166b 2026-10-21 feat(auth): support trusted group header (#6396)
c133b9467 2026-10-21 fix(storage): 升级后继续隐藏旧版多写元数据 (#6395)
feb051c70 2026-10-21 fix(bot): translate POSIX compile sandbox commands on Windows (#6372)
6545bb2a4 2026-10-21 fix(compile): keep the compile workspace root absolute (#6376) (#6387)
0a2578e7e 2026-10-20 fix(studio): redesign connection switching and verify account identities (#6308)
a07c02cd2 2026-10-20 fix(agent-evolution): preserve explicit Experience read attribution (#6365)
03b9aecfe 2026-10-20 fix(auth): accept OIDC issuers with a trailing slash (#6254)
e6e5f7d2f 2026-10-20 docs(localfs): document case-insensitive alias behavior (#6053)
5d1027468 2026-10-20 fix(ov_cli): trust native CA roots (#6361)
aa49650a2 2026-10-20 fix(vectordb): bound ordered delta replay batches (#6259)
102ff5925 2026-10-20 feat(admin): O(1) account/user existence endpoints (#6325)
1455cb128 2026-10-20 fix(cli): update openviking-server init default VLM models to Seed 2.1 Lite (#6348)
a9cb3b4d4 2026-10-20 fix(ov_cli): show the underlying cause of connection errors (#6312)
0902110a0 2026-10-20 test(claude-code-plugin): fit recall fixture within the full context budget (#6353)
d9bc1e4b4 2026-10-20 docs: list NVIDIA cuVS in partner projects (#6351)
87ffde99e 2026-10-20 fix(bot): restrict exec for user chat requests (#6344)
2b77e30aa 2026-10-20 feat(mcp): expose privacy tree, resolve and upsert tools (#6341)
21a3ca15c 2026-10-20 fix(bot): mark web fetch failures as errors (#6311)
35b51da9a 2026-10-20 fix: detach shared batch and snapshot fallback task contexts (#6331)
7c12d3062 2026-10-20 fix(langchain): preserve recall inputs and context metadata (#6338)
3a5733fba 2026-10-20 fix(claude-plugin): skip api-request-blob transcript lines when capturing turns (#6297)
172c8648f 2026-10-20 fix(recall): mark recalled memory as the user's own history (#6285)
1c4399988 2026-10-20 feat(privacy): add current configuration tree overview (#6337)
d568563f8 2026-10-20 feat(parser): add network adapter extension point (#6302)
a8bd02bac 2026-10-20 fix(gateway): classify Codex requests by request_kind, not keywords in the turn metadata (#6334)
b281505a6 2026-10-20 fix(session): decouple Phase 2 extraction from live session files (#6332)
c267ca48c 2026-10-20 fix(gateway): classify harness side requests and loop nudges (#6327)
8640600f4 2026-10-20 feat(gateway): let operators add to or remove from the packaged client rules (#6333)
8d52018cf 2026-10-20 fix(rerank): align VikingDB scores by document ID (#5833)
3ef6c5f7f 2026-10-20 fix(session): avoid duplicate trajectory extraction across cases (#6273)
d8f485496 2026-10-20 fix(langchain): align Store deletion, validation and removal errors (#6324)
aa47c878f 2026-10-20 fix(session): retry memory extraction after safe read failures (#6301)
68e66ec89 2026-10-19 fix(hermes-plugin): retry the health probe when a gateway redirects to its login page (#5984)
59ef89730 2026-10-20 fix(studio): ignore AppleDouble metadata during lint (#6316)
a3169a824 2026-10-20 fix(studio): forward admin key for bot chat requests (#6215)
e855796c8 2026-10-19 fix(gateway): stop duplicate recall and new sessions for DSH and tool-less side requests (#6295)
b51cba425 2026-10-19 Fix/remove case only conflict check (#6298)
c89efb7c3 2026-10-19 fix(storage): 修复写入协调消息提前消费导致的锁泄漏 (#6300)
f5ea83b64 2026-10-19 fix(studio): localize remaining UI text and enforce i18n checks (#6292)
2718f9657 2026-10-19 feat(admin): allow deleting nonempty user groups (#6282)
c476a3184 2026-10-19 Fix/cross process recovery (#6281)
59fff75e8 2026-10-19 docs(gateway): dedicated Gateway docs section, rewritten for readers, with Studio deep links (#6287)
b1d0fa51e 2026-10-19 fix(langchain): preserve conversation text, roles and tool history (#6275)
c6d15e90a 2026-10-19 docs: refresh WeChat group QR code (#6294)
c01bb34d8 2026-10-19 feat(tasks): add MySQL task work store (#6258)
d85e2b221 2026-10-19 docs: refresh WeChat group QR code (#6291)
b15fa86f5 2026-10-19 fix(vectordb): read the items key HttpCollection's own server returns (#6193)
597db4fe2 2026-10-19 fix(memory): do not write an upsert from a stale snapshot when the fresh read fails (#6075)
27db44123 2026-10-19 fix(gateway): read cache usage from every upstream form and warn about long conversations missing the cache (#6267)
6bd89a36b 2026-10-19 fix(hermes-plugin): drop the query-token overlap boost from recall ranking (#5712)
66f528de3 2026-10-19 fix(opencode-plugin): derive session id from parentID only on session events (#6243)
e78127400 2026-10-19 fix(langchain): extract text blocks in viking_store (#6070)
288617236 2026-10-19 perf(session): 减少消息写入和 Session 接口的重复读取 (#6270)
5b96302de 2026-10-19 fix(hermes-plugin): upgrade Quick Local to OpenViking 0.5 (#6114)
b727b8163 2026-10-19 fix(tasks): 移除文件任务存储的统一 I/O 限流 (#6265)
ac0f1809c 2026-10-19 fix(privacy): handle file roots in no-split resource imports (#6260)
dfc22d1de 2026-10-19 fix(storage): settle AGFS copies before propagating cancellation (#6178)
a9c21c2fd 2026-10-19 fix(compile): isolate account locks and stop stalled review loops (#6133)
d16ce2bb6 2026-10-19 fix(studio): improve model settings and private connection tests (#6177)
9cd778617 2026-10-19 feat(metrics): track connector imports and ingestion token sources (#6245)
0fcc16638 2026-10-18 fix(working-memory): use tool_choice auto for thinking-mode VLMs (#5636)
8d28ffcdb 2026-10-17 test(hermes-plugin): cover native patch on current Hermes (#6113)
7c34e43c5 2026-10-16 fix(session): release archive lock before task registration (#6175)
013e988bb 2026-10-16 refactor(tasks): converge task/work store contract (#6136)
2ef422561 2026-10-16 perf(session): skip content loading and vector cleanup during deletion (#6165)
97bf0fdc3 2026-10-16 feat: optimize ov privacy CLI commands (#6161)
4c976135a 2026-10-16 Feat/vikingbot steer interrupt (#6115)
fa37232e4 2026-10-16 docs: refresh Studio README screenshots (#6166)
855d0928a 2026-10-16 feat(search): support multimodal hybrid queries and AML payloads (#6128)
f64fb5ba7 2026-10-16 fix(resource): report a queued reason commit as accepted (#6157)
842ae41e4 2026-10-16 docs: point hosted Studio links at canonical domain (#6156)
26fa00b91 2026-10-16 perf(gateway): cut per-request CPU in request preparation, reply relay and reply lookup (#6026)
119eba57d 2026-10-16 feat(mcp): add download tool with single-use links, proxied by the gateway (#6119)
8483463c4 2026-10-16 feat(gateway): Azure and key-in-URL upstreams on top of exact endpoints (#6101)
1c7b86d09 2026-10-16 feat(gateway): quick mode on the Studio overview and lost management key docs (#6107)
b0d1d633f 2026-10-16 fix(storage): avoid blocking executor on PathLock waits (#4548)
0230cc104 2026-10-16 fix(resource): return from on_resource_added once the reason commit is queued (#6077)
015aa74c3 2026-10-16 fix(session): 在任务结果保存后写入归档完成标记 (#6153)
594f5f25d 2026-10-16 fix(studio): restore saved setup and fix connection branding (#6148)
e24295fd6 2026-10-16 fix(session): record SessionCommit Phase-2 worker failures as terminal task errors (#5622) (#5772)
6006fc340 2026-10-15 feat(agent-plugins): add ov-resources skill (#6092)
83f6831bd 2026-10-16 fix(bot): use home alias for session tool URIs (#6121)
6ff9ae461 2026-10-16 fix: replace deprecated datetime.utcnow() with datetime.now(timezone.utc) (#6123)
84cc2b412 2026-10-15 perf(memory): streamline extraction prompts and deduplicate inputs (#6120)
ee908e763 2026-10-15 fix(vlm): default affected GPT tool calls to no reasoning (#6137)
54e199f41 2026-10-15 fix(embedding): truncate plain list inputs to max_input_tokens (#5760) (#5771)
c1d3ad90b 2026-10-15 fix(openwebui): parse bracketed IPv6 bind addresses (#6106)
465b14661 2026-10-15 refactor(embedding): remove unused token estimator (#6134)
d1984f16e 2026-10-15 refactor(session)!: 移除独立 extract 入口，统一使用 commit (#6131)
c79600ae5 2026-10-15 feat(gateway): test upstreams with model generation (#6118)
621b1ddd9 2026-10-15 fix(plugins)!: apply captureFilters substitutions to tool call inputs and outputs (#5937)
afc516932 2026-10-15 refactor(session): 清理无生产调用的旧实现 (#6117)
1241f0c6b 2026-10-15 fix(tasks): preserve task owner in queue work (#6122)
cc9dff6e8 2026-10-15 fix(retry): classify Ark "timed out" and bare TimeoutError as transient (#6102)
e05452ee8 2026-10-15 fix(gateway): log the capture status a request ends with (#5981)
a0c357417 2026-10-15 fix(opencode-plugin): drop local session mapping when the server no longer has the session (#5935)
60629f92b 2026-10-15 fix(opencode-plugin): prune stale fully-captured sessions from local state (#5936)
5680dac89 2026-10-15 fix(retrieve): stop served directories from hiding new files in recall (#5982)
ec56b9443 2026-10-15 refactor(tasks): unify task and queue work tracking (#6043)
6d1145ae5 2026-10-15 refactor(integrations): organize framework packages (#6017)
f0743df9e 2026-10-15 fix(storage): propagate Volcengine deletion request failures (#5268)
7928018bf 2026-10-15 fix(vectordb): validate snapshot payload in PersistentIndex newest-version selection (#5595) (#5784)
f75b6141d 2026-10-15 fix(vectordb): surface native store mutation failures (#5803)
229098416 2026-10-15 fix(acl): resolve ACLs from records hidden by the caller's scroll filter (#5983)
bd26e83b2 2026-10-15 feat(web-studio): workspace-first setup and dual-provider experience (#6066)
acd7caf0d 2026-10-15 refactor(models): 统一模型错误分类与重试边界 (#6047)
9baf83f04 2026-10-15 feat(gateway): allow exact upstream endpoint URLs (#6100)
89fbcb968 2026-10-15 feat(session): distributed Redis-backed idle auto-commit scheduler (#6046)
46e15a248 2026-10-14 docs: 新增数据组织指引，并支持 MCP add_resource parent 自动建目录 (#6060)
6cf676788 2026-10-14 Feat/redis mysql account store (#5955)
ae1e45e00 2026-10-14 fix: 兼容索引 ACL 列表中的空字符串 (#6067)
f2b09cd1c 2026-10-14 fix(memory): preserve Markdown structure in resource links (#6054)
a55e423b8 2026-10-14 fix(ci): 修正 API 契约断言和工作流配置 (#6062)
f047d8051 2026-10-14 fix(openwebui): validate session IDs before forwarding requests (#6059)
5242d28d7 2026-10-14 test(queuefs): support script-aware cache mocks (#6055)
89006aba4 2026-10-14 fix(snapshot): 复用外层目录锁完成加密快照恢复 (#6052)
ad4d99de1 2026-10-14 feat(cli): support list and context search modes (#5951)
4496edb55 2026-10-14 refactor(api)!: 移除等待专用 timeout 参数 (#6048)
4f712cc94 2026-10-14 fix(svg): 补齐 SVG 转 PNG 所需的运行依赖 (#6049)
3385dd596 2026-10-14 fix(api): 修正输入错误与模型失败的 HTTP 状态码 (#6045)
b8ec9d821 2026-10-13 perf(retrieval): align default intent planner with compact v4 prompt (#6039)
e05460914 2026-10-14 fix(memory-plugin): carry OPENVIKING_EXTRA_HEADERS on the runtime path and fail on a non-JSON 2xx (#5988)
2d43f20ca 2026-10-13 fix(storage): sanitize persisted search tags on index writes (#5862)
da5e61b18 2026-10-14 fix(langchain): recognize SDK system health flag (#5878)
6c802cd94 2026-10-14 fix(storage): 保持写入锁直到异步索引完成 (#6016)
b15426b28 2026-10-14 refactor(model): 统一 VLM 与 Embedding 重试策略 (#5950)
bd78cbe57 2026-10-13 refactor(cache): 统一 cache key format (#6014)
dc9f14537 2026-10-13 docs: update readme (#6021)
6c1107527 2026-10-13 feat(cache): 增加顶层 service_name 配置， 删除 namespace/cache_key_prefix  ，使用service_name 统一管理 (#6011)
938ef31f6 2026-10-13 fix(retrieval): bound session search intent analysis (#6007)
8a74caf5b 2026-10-13 feat(gateway): embed runtime and separate management access (#5960)
7f69dd24c 2026-10-13 feat(integrations): add Microsoft Agent Framework context provider (#5578)
0f8c5c538 2026-10-13 feat(docs): send copy and search events to Google Analytics (#5990)
c90dae83d 2026-10-13 Feat/batch write rnfv (#5897)
ff790205a 2026-10-13 feat(storage): implement multi-write v2 metadata partitioning (#5741)
676c4038d 2026-10-13 fix(auth): reject oversized new account IDs before persistence (#5987)
1d95584c1 2026-10-13 feat(bot): 支持关闭 Direct 命令执行 (#5999)
2da844644 2026-10-13 fix(deps): require openviking-sdk 0.1.15 and guard Compile write options in Docker (#5993)
b218cd805 2026-10-13 perf(embedding): speed up token guards without changing estimates (#5997)
65d89a957 2026-10-13 docs(agents): add pi, ZCode and Kimi Code to the agent catalog (#5996)
2f077bbec 2026-10-13 docs(agent-integrations): put install first on the pi and OpenCode pages (#5995)
172d4d9e2 2026-10-13 docs(plugins): point READMEs at the one-line installer (#5994)
d41cd6db1 2026-10-13 fix(tasks): derive processing time from work timestamps (#5991)
5ff645a4e 2026-10-13 fix(compile): bound job and shuffle retry attempts (#5989)
7b7654e80 2026-10-13 fix(resources): skip unsafe repository paths during upload (#5953)
8fe792dd1 2026-10-12 fix(langchain): accept mapping values in LangGraph store (#5980)
5ece19654 2026-10-13 fix(storage): preserve staged source on worker shutdown (#5888)
79ecf74f9 2026-10-13 fix(hermes): defer pending recovery in non-primary contexts (#5917)
f79732497 2026-10-13 fix(session): serialize initialization with message writes (#5978)
b96230303 2026-10-13 feat: 优化studio对话的chatbox样式交互 (#5956)
ba335574f 2026-10-13 feat(hermes-plugin): support extra HTTP headers (ovcli.conf `extra_headers` / `OPENVIKING_EXTRA_HEADERS`) (#5884)
400b70eee 2026-10-13 feat(dsh-plugin): commit pending messages on compaction/start (#5254)
22ea8c7b3 2026-10-12 fix(plugins): parse CR-delimited MCP event streams (#5973)
70d3e1d30 2026-10-12 feat(docs): add sitemap, robots, canonical/hreflang, page descriptions and GA (#5975)
92285e5b3 2026-10-12 fix(pdf): serialize PDFium rendering across parser instances (#5952)
90df14547 2026-10-12 Refactor/account store provider (#5933)
3cdfbc22f 2026-10-12 fix(auth): escape LDAP direct bind username in DN (#5949)
34187bc0c 2026-10-12 fix(auth): stop issuing personal keys in trusted mode (#5946)
df2b6df27 2026-10-12 fix(auth)!: trusted 模式强制要求 root API key (#5947)
4a2909b44 2026-10-12 fix(memory): preserve code fences in extraction content (#5852)
9179d5bab 2026-10-12 fix(studio): use managed root key for trusted Feishu connections (#5500)
df222c0c1 2026-10-12 fix(vector): apply tenant filter to debug scroll and count (#5945)
db0d241b7 2026-10-12 fix(vectordb): prune delta records after each index persist (#5940)
4e7581009 2026-10-12 fix(config): remove misleading storage path deprecation warnings (#5941)
5b695058d 2026-10-12 fix(bot): reject untrusted identities in rootless trusted mode (#5929)
84cb68c07 2026-10-12 fix: 限制语义与向量任务的失败重入队次数 (#5934)
a99013ae4 2026-10-12 fix(langchain): run once-only init on explicit async handle initialize (#5908)
2e706d719 2026-10-12 fix(gateway): detect client OpenViking tools under any MCP server name (#5930)
8c3ef511f 2026-10-11 fix(plugins): retain Unicode letters in capture text (#5804)
9093583b3 2026-10-12 fix(recall): factual compression digests, Haiku / gpt-6-luna compressors, custom Codex providers (#5732)
35fd8c8b9 2026-10-11 fix(plugins): forward recall exclusions through agent hooks (#5616)
0cdce9690 2026-10-11 fix(deps): require openviking-sdk>=0.1.14 for Compile (#5928)
50ca28dc2 2026-10-12 fix(search): reject non-positive retrieval limits at the API boundary (#5926)
23fa33b9b 2026-10-12 fix(ci): install pnpm for DSH host compatibility smoke (#5859)
3ecfa0a3d 2026-10-12 fix(storage): classify append_file failures instead of collapsing them to IOError (#5688)
9d6b1f601 2026-10-11 fix(plugins): strip CRLF from doctor command paths (#5567)
b35c3f2aa 2026-10-11 test(opencode): make plugin path assertions portable (#5562)
48781b9c8 2026-10-11 docs: refresh WeChat group QR code (#5925)
4c9a755bf 2026-10-10 feat(studio): edit startup configuration and support ROOT restart (#5730)
28511a553 2026-10-10 fix(sdk): expose optional grep traversal depth (#5555)
c89ded12f 2026-10-10 fix(ragfs): skip vanished localfs entries during listing (#5590)
3de713f87 2026-10-10 docs(agents): update MCP setup and skills installation guidance (#5875)
107cf0dc7 2026-10-10 feat(docs): preserve UTM attribution across documentation links (#5870)
589c0d1de 2026-10-10 fix(security): reject unverifiable remote DNS targets (#5864)
144033826 2026-10-10 docs(gateway): update hidden tool loop limits to no default limit (#5839)
7967eca58 2026-10-09 feat(compile): add a plannable Map/Shuffle/Reduce pipeline (#5750)
```
