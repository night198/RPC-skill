# 原生 RPC 操作参考

这是执行方法，不是自动调度客户端。启动前阅读当前 `omp --help`、版本以及实际安装包内 RPC 类型与实现；下列字段来自原 omp-independent-dev 的 OMP 18.8.7 参考，升级后必须核对。

## 启动与环境

使用参数数组启动，避免拼接 shell 字符串。从已初始化的 SKILL.md 参数表读取值；下列 `<…>` 是每批运行变量，不是待替换的安装参数。

```text
<omp-executable> --mode rpc --cwd <absolute-project>
  --session-dir <private-persistent-session-directory>
  --model <configured-provider/id> --thinking <configured-thinking-level>
  --no-extensions --no-title --tools <slice-tool-list>
  --append-system-prompt <controller-boundaries>
```

恢复追加 `--resume <exact-session-file>`。Profile 按项目明确选择。不要用裸 --continue、最近会话、模糊 id 找恢复目标；不得使用 --no-session。不要默认 --no-rules、--no-skills、--auto-approve、yolo。所需 LSP 保留；只有明确不需要时才 --no-lsp。

启动前落实初始化表的 extended-context 和窗口设置。先查当前版本实际键及作用范围，优先使用本次运行或项目限定配置；不编造 `--extended-context` 开关，不通过减小显示值假装关闭扩展。涉及持久化设置需用户确认。Fast 的设置由连接后 RPC 完成，不能把 `--thinking` 或 `--service-tier` 等同于 Fast。

只核对继承环境的相关变量名和已知非秘密绑定；清理主控递归/session、无关生产/Profile 绑定。保留 HOME、PATH 及既有认证来源，不读取或复制秘密；不修改主控全局环境。显式允许必要项目变量。

启动附加边界包括本片项目、Spec、允许文件/工具、主控唯一派发、禁止创建子 Agent 或递归会话、禁止未经授权的提交/推送/部署、输出证据与停止点。恢复时重新附加，不能假定启动提示完整持久化。

## JSONL 连接

stdin/stdout 用 UTF-8，一行一个 JSON 对象，每条请求唯一 id，发送后 flush。stdout、stderr 分开持续读取，避免塞满管道。stderr 和原生会话可能含源码或秘密，保存于私有目录并脱敏摘要。

控制端要求：唯一 stdin 写入者；按 id 关联请求/响应；超时/EOF/退出处理；有界缓冲或私有落盘；事件等待而非忙轮询；明确处理 extension_ui_request 等交互事件，不默认同意高风险操作。

可按当前接口用 set_event_filter 和 messageUpdates: "delta" 减小缓冲；不得过滤掉回合结果、settled 或交互请求。

## 无模型调用检查

```json
{"id":"state-before","type":"get_state"}
```

核对 sessionId、精确 sessionFile、model.provider/id、思考档位、Fast enabled/active、contextUsage.percent、空闲和队列。在当前 RPC 定义中确认这些字段；窗口与 extended-context 如果 state 不提供，使用当前版本的模型元数据和已生效配置证据，不编造字段。

Fast 请求必须由已确认的布尔参数构造，不直接发送占位字符串。例如控制端先把 SKILL.md 参数表解析为 `config`，校验 `config.fastModeEnabled` 类型后构造：

```javascript
if (typeof config.fastModeEnabled !== "boolean") throw new Error("Fast 参数未初始化");
const request = {id: "fast-config", type: "set_fast_mode", enabled: config.fastModeEnabled};
// 由唯一发送者将 JSON.stringify(request) + "\n" 写入 stdin 并 flush。
```

收到同 id 成功响应后，再发送 get_state 核对。要求 active 时核对真实 active；配置不匹配停止。不支持所选模式时让用户裁定，不自动换模型。set_model 可能涉及持久化语义，使用前检查当前实现；优先在启动参数中显式选模型。

核对实际 dumpTools。可用 get_available_commands 检查 shake/compact，用 get_available_models 只读取必要 provider/id，不能把完整连接配置输出到聊天。

## 派发与完成

```json
{"id":"slice-001","type":"prompt","message":"冻结切片的完整目标、契约、授权、验收与停止点"}
```

1. 同 id response success 只表示受理。
2. 等待同 id prompt_result，分别处理 completed/error/aborted。
3. 检查 sessionSettled；未 settled 时等事件，再读取 state 确认 isSettled、无流式输出、压缩、队列或异步任务。
4. 仅在该请求确认 completed 且 settled 后获取 get_last_assistant_text。禁止拿上一轮文字冒充本片结果。
5. 主控审查和行为验收，写入进度，才推进依赖片。

转向用 steer；用户明确立即停止时用 abort。abort 响应后仍需等停止并检查已发生副作用。模型失败保留脱敏类别/HTTP 状态和真实文件状态，不套无限重试。超时不能证明未执行。

## shake 与 compact

安全空闲边界执行，具体阈值见安装后的 SKILL.md。

```json
{"id":"shake-001","type":"prompt","message":"/shake"}
```

检查原生 slash 分派结果。18.8.7 参考响应含 agentInvoked:false；不能当作模型已完成开发。然后重读占用。

先写交接检查点，再发送：

```json
{"id":"compact-001","type":"compact","customInstructions":"保留有效Spec、授权、文件所有权、基线、真实改动、已验证与未验证项、请求结果、资源状态、下一片；不扩大权限，不重放副作用。"}
```

compact 按同 id 响应判断，不等待开发 prompt_result；可能调用模型。失败保留失败，成功检查 state 与占用，按已配置恢复上限决策。

## 停止与恢复

确认 settled 且无管理命令后关闭 stdin，限时等待退出。必要时只处理有证据归属的本批次进程；进程退出不证明它启动的外部资源已停止。

内核/管道丢失时，新父进程通常不能重新附着旧 stdin。核对启动时间、可执行文件、命令行、session 目录和当前活动，不仅看 PID。未确认旧写入者停止前不启动第二个写入者。

旧进程安全停止后可精确恢复 sessionFile，核对 sessionId、模型、模式、窗口、占用、队列。需要新上下文则保留旧 session，另开新 session，只提供有效交接、Spec 与必要源码，不重新灌整份历史。

## 当前版本核对入口

在实际 OMP 安装包定位 RPC 类型、rpc-mode 实现和 CLI help，原参考文件名为 `src/modes/rpc/rpc-types.ts`、`src/modes/rpc/rpc-mode.ts`。实际可能只有构建产物，按安装版本定位，不能猜路径。核对 prompt_result/settled、get_state、Fast、compact、slash、交互与退出语义；不匹配就修订本地参考后验收。
