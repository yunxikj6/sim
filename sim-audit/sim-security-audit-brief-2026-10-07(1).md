# Sim 安全审计简报

- 审计基线：sim 仓库提交 546d4e7e5d506bfbc2b74abe5fcb6d6e5f7ef6e1（v0.9.14）
- 截止时间：2026-10-07
- 审计方式：静态检查路由、middleware、use-case、服务端集成和关键数据流；本轮未修改业务代码。
- 结论口径：下表“已确认问题”表示从当前代码路径可以直接推出安全、资源隔离或计费不变量失效；不等同于已在生产环境执行利用。

## 1. 已确认问题

| 编号 | 严重度 | 覆盖面 | 结论与影响 | 关键证据 |
|---|---|---|---|---|
| C-01 | P1 / High | Legacy workflow-MCP 管理 API | 创建、更新和删除 workflow-MCP server 的旧路由使用 write 权限，但参数允许设置 isPublic。具备 workspace write 权限且具备 deploy.mcp 能力的成员可以把工作流 MCP 暴露为匿名公共端点。匿名调用随后以创建者身份进入工作流工具执行链路，存在数据暴露、未授权副作用以及提供商/计费滥用风险。 | apps/sim/app/api/mcp/workflow-servers/route.ts；apps/sim/app/api/mcp/workflow-servers/[id]/route.ts；apps/sim/lib/mcp/middleware.ts；apps/sim/app/api/mcp/serve/[serverId]/route.ts。v2 operation registry 已将同类创建、更新、删除和部署操作限制为 admin，形成权限不一致。 |
| C-02 | High | Neo4j DNS 校验与连接重连 | 加密连接路径在完成 DNS/私网校验后将 usePinnedIp 关闭，并重新使用原始主机名连接；校验对象与实际连接对象不一致，重连时仍可能再次解析到不同地址，削弱 DNS rebinding 和私网访问防护。 | apps/sim/lib/internal/neo4j/client.ts |
| C-03 | High | 直接工具调用与 copilot in-band 计费 | 部分内置工具直调和 copilot in-band 执行路径没有统一进入 usage reservation、commit 和 billing attribution 链路，存在漏计量、错归属或执行结果与费用记录不一致的可能。前一轮 S01–S06 中涉及的计费/模型归属问题已归并到本项及 C-09，避免重复计数。 | apps/sim/app/api/v2/tools/[toolId]/execute/route.ts；apps/sim/app/api/copilot/tools/execute/route.ts；apps/sim/lib/internal/tool-operations/* |
| C-04 | High | PII 服务 HTTP 接口 | /analyze、/analyze_batch、/anonymize、/anonymize_batch、/redact 和对应 health/supportedentities 接口未见认证门槛。只要服务网络边界可达，调用方即可提交任意文本进行处理，形成未授权数据处理、资源消耗和结果暴露风险。 | apps/pii/server.py |
| C-05 | High | Realtime 身份与房间/资源授权 | 连接身份校验与房间、协作资源授权没有完全对齐，部分事件处理依赖连接层身份而非逐资源授权。存在未授权接收协作状态、发布事件或修改资源的风险，实际影响取决于 realtime 服务的可达性和房间配置。 | apps/realtime/src/middleware/auth.ts 及 room/event handlers |
| C-06 | Medium-High | OOXML/归档浏览器预览 | 浏览器预览防护在调用 JSZip 前依赖归档声明的大小；代码已明确承认声明值可能失真。恶意归档可绕过声明值检查，在客户端解压和 DOM 处理阶段放大 CPU/内存消耗，造成浏览器级 DoS。 | apps/sim/lib/file-parsers/ooxml-preview-guard.ts |
| C-07 | Medium-High | 文件类型/解析器循环边界 | 文件解析路径存在缺少有效进度或终止上限的循环，攻击者可通过特制输入触发长时间 CPU 占用，形成可用性风险。 | apps/sim/lib/file-parsers/* 中的文件类型/解析循环 |
| C-08 | Medium | Runtime secrets 日志上下文 | secret 重试和最终失败日志携带 secretId 及底层异常上下文，日志面可能暴露敏感配置定位信息或供应商返回内容。当前未将 plaintext SecretString 作为已确认泄露证据，但日志应按敏感信息处理并统一脱敏。 | packages/runtime-secrets/src/index.ts |
| C-09 | Medium | Fal 音频模型选择 | generateFalAudio 直接将调用方提供的 model 拼接为 Fal queue endpoint，缺少服务端 allowlist、模型类型约束和按模型计费限制。可控调用方可能选择未批准模型，造成托管密钥成本滥用、模型策略绕过或计费失真。 | apps/sim/lib/media/falai-audio.ts |

## 2. 已检查、暂未发现可确认问题

| 覆盖面 | 检查结论 |
|---|---|
| Workspace fork/import、跨组织与 delegated use-case | 已检查 workspace、组织边界、委托调用和凭据复制路径；当前未发现可直接确认的跨租户读取、越权写入或凭据外泄。 |
| Credential group、公开 enrollment、OAuth state/token | 已检查 token hash、verified email、state 绑定、一次性 Redis TTL 和回调消费；当前未发现可直接确认的重放或跨账户绑定绕过。 |
| Legacy credentials、用户/组织 API key、组织 secret/member 路由 | 已检查创建、读取、轮换、删除和成员范围；当前未发现可直接确认的越权读取或跨组织访问。 |
| 文件分享、原始文件服务、下载/预览 | 已检查密码、SSO/OTP、访问上限、公共前缀和路径 containment；当前未发现可直接确认的未授权文件读取或路径穿越。OOXML 预览的放大问题单独列为 C-06。 |
| Workflow public API、MCP OAuth、SSRF 与出站请求 | 已检查资源绑定认证、逐跳 DNS/私网校验、重定向处理和 pinned fetch；当前未发现可直接确认的跨租户 SSRF 或资源越权。Legacy workflow-MCP 的管理角色错配已单列为 C-01。 |
| CLI device handoff、OAuth2 authorize/register/token/revoke | 已检查 approval/scopes、requestId 与 pollSecret 绑定、PKCE、redirect/client 校验、state 轮换以及速率/尺寸限制；当前未发现可直接确认的设备授权劫持或 token 重放。 |
| Webhooks、schedules、chat/public chat、Mothership chat/manage/read/restore/fork/events | 已检查 session/write 权限、公开触发器边界、provider 与跨租户路径、所有权和 workspace scope；当前未发现可直接确认的高影响越权。个别 DELETE 变更锁和接口语义差异暂未形成可复现绕过。 |
| Mothership sandbox/internal delegation/execute、v2 workflow execute、API keys 与 billing attribution | 已检查路径/遍历、scope token/API key、签名委托、workflow/run 绑定和统一计费归属；当前未发现可直接确认的跨租户执行或身份伪造。 |
| Knowledge connectors、MCP search/registry/serve/discover/test connection、connected accounts | 已检查 knowledge.use、资源绑定授权、重新授权和组织连接账户边界；当前未发现可直接确认的资源越权。 |
| 组织管理员操作、BYOK、connected accounts、data drains、permission groups/domains/session policy | 已检查管理权限、组织范围和策略落点；当前未发现可直接确认的管理员边界绕过。 |
| Trello OAuth、folders/resume、inbox sender、Confluence tool、内部 file/doc、upload session/multipart | 已检查回调状态、资源归属、内部调用凭据、上传会话和 multipart 边界；当前未发现可直接确认的高影响越权。 |
| Table/custom tools、wand、workflow variables/state/status/deployment、chat 管理和 public shares | 已检查资源所有权、workspace scope、部署状态和公开分享入口；当前未发现可直接确认的跨租户访问或未授权状态变更。 |
| AgentMail settings/dynamic integration/dashboard 及组织 Generic Secrets 语义 | 已复核角色/能力门槛和管理路径；现有证据不足以脱离产品策略确认漏洞，相关 admin/write 语义应由产品权限矩阵和回归测试进一步固化，因此未计入已确认问题。 |

## 3. 优先整改建议

1. 立即将 legacy workflow-MCP 的创建、更新、删除和公开化操作统一提升为 admin，或全部收敛到 v2 mcpServerOperations；补充 write 成员拒绝和匿名 serve 回归测试。
2. 统一所有工具执行入口的 usage reservation、commit、失败回滚和 billing attribution，覆盖直调、copilot in-band 以及模型选择分支。
3. 修复 Neo4j 实际连接地址与 DNS 校验结果不一致的问题；对 PII 服务增加认证、网络隔离和请求体/并发上限。
4. 为 OOXML/归档预览和文件解析器增加基于实际解压字节、条目数、时间和进度的硬上限；为 realtime 事件增加逐资源授权。
5. 对 runtime secrets 日志做统一脱敏；对 Fal 模型使用服务端 allowlist、类型约束、租户级配额和可审计的价格映射。

