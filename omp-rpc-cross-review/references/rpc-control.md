# 交叉审核 RPC 连接参考

此文档是主控执行方法，不是客户端程序。先核对当前 `omp --version`、`omp --help` 及真实 RPC 接口。示例沿用普通版已使用的原生 JSONL 协议；接口可能随版本改变，不猜字段或命令。

## 启动前

先完成 [Skill](../SKILL.md) 的参数、身份、权限与费用门禁；准备项目外私有 session 目录，每个会话独立保存。确认已有会话与所有权，未解决运行不能盲目新开写入者。

从当前可用模型元数据只取得必要 provider/id 和思考能力，不输出完整连接配置或读取凭据。模型参数及项目路径作为参数数组传给实际 OMP 可执行文件，不拼接用户字符串到 shell。

```text
<omp> --mode rpc --cwd <project>
  --session-dir <private-session-directory>
  --model <exact-provider/model-selector>
  --thinking <confirmed-thinking>
  --no-extensions --no-title --tools <approved-tools>
  --append-system-prompt <role-and-scope-boundaries>
```

审查 selector 必须精确映射已确认 review_provider 与 review_model。用当前 CLI 支持的选择格式，不能模糊匹配简称。review_thinking 原样映射至当前支持接口；仅当用户确认且能力证明不可调时省略 thinking 参数，不发送 `not-applicable`。开发使用本批授权配置，与审查参数分开。Fast/服务档位不从开发角色强制复制到审查角色，不改全局模型和上下文设置。

Profile 按项目可信配置选择；保留 HOME/PATH 与现有认证来源，不读取/复制秘密，只清理已确认无关的递归/session 或生产环境绑定。保留项目规则及必要 Skills/LSP，不默认自动批准或跳过规则。附加角色边界明确任务、文件/工具范围、主控唯一派发、不得递归调度、不得未授权提交或部署、完成即停。审查额外默认只读、不改业务或正式台账、不批准发布。工具约束不能冒充系统沙箱。

## 无模型调用状态检查

```json
{"id":"state-before","type":"get_state"}
```

核对真实 sessionId/sessionFile、provider/model、实际思考强度、工具、settled/队列与上下文占用；字段按当前实现确认。审查配置与用户确认不一致或无法核对必要状态时停止，不发送业务 prompt。不可调强度用模型能力证据和未发送设置记录核对。不能用请求参数回显代替生效状态。窗口字段缺失不构成失败。

get_available_models 用于确认非秘密模型身份/能力；get_available_commands 用于确认 Skill、shake/compact 等命令；核对实际 dumpTools。若当前接口不能提供某项证据，查 CLI/可信能力定义，仍未知则报告，不宣称已验证。新进程发现 Skill 不等于审查模型已成功运行。

## 传输与完成

stdin/stdout 使用 UTF-8 JSONL，每条请求唯一 id，发送后 flush；stdout/stderr 分开持续消费，有界缓冲或私有落盘，不能堵住其他并发会话。主控唯一持有各 stdin，按 `(sessionId, request id)` 匹配请求、响应和事件。

```json
{"id":"review-slice-001","type":"prompt","message":"指定Spec、稳定候选、静态审查范围、证据格式、只读权限与停止点"}
```

1. 同 id response success 仅为受理；等待同 id prompt_result。
2. completed/error/aborted 分别保留。completed 后仍需 sessionSettled 或 get_state 的实际 isSettled，无队列、流式、压缩或异步工作。
3. 只有本次 completed 且 settled 后才能读取 get_last_assistant_text，不能使用上一轮文本代替新结果。审查报告不足或未完成不算通过。
4. 主控核对对应候选的发现与处置，满足静态门禁后接收切片；完整功能齐备后集中冒烟，二者分别记录。

按当前接口处理交互请求，不默认授权高风险操作；事件过滤不能丢弃结果、settled 或交互。steer 用于限定转向；abort 需等待真实停止并检查已发生副作用。错误、限额、超时或 EOF 保留现场，不自动重复可能已执行的操作，不无限重试或换收费模型。

## 压缩、停止和恢复

在 settled 且无管理操作的边界，依据本批策略执行 `/shake`，检查原生分派结果与真实占用，不把 slash 受理当作开发完成。compact 前保存台账：

```json
{"id":"compact-001","type":"compact","customInstructions":"保留有效Spec、授权、模型身份集合、审查参数和确认来源、稳定候选、发现与处置、所有权、静态接收和运行验收区别、未完成项与恢复点。"}
```

compact 可能调用模型，需处于授权费用范围；按同 id 管理响应判断，不等待开发 prompt_result。失败保留失败，成功后检查占用和已确认恢复策略，不循环压缩。

计划停止先写交接，确认 settled 且无排队任务，再关 stdin 并等待退出。只操作有证据属于本批的进程；进程退出不证明外部资源已停止。失联时核对 PID、启动时间、命令行、session 文件与实际写入，不能凭超时重放任务。

安全停止后使用 `--resume <exact-session-file>` 恢复并重新附加角色约束、核对真实身份和参数。不要用裸 continue、最近会话或模糊 id。需要更小上下文时另开 session，提供有效交接和必要源码，保留旧记录。恢复与新开发模型加入均重新检查异模型门禁，历史开发模型集合不能丢弃。
