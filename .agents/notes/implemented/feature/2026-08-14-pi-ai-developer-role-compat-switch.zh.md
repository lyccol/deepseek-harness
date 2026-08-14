# Agent Note: Configurable developer-role compat switch for pi-ai routes

Status: implemented

[English](2026-08-14-pi-ai-developer-role-compat-switch.md) | 中文

## Problem

只要 pi-ai 把端点读作与 OpenAI 兼容，它就会把具备推理能力的模型的系统提示放在 `developer` 消息角色下发送；而这一判断来自端点 URL：凡不在它那份很短的非标准主机名单（nvidia、cerebras、xai、together、deepseek.com 等少数几家）之内的主机，都被假定接受该角色。名单之外的一律不在其中，私有网关与厂商自家的 OpenAI 兼容端点同样如此，因此一个后端只允许 `user`/`assistant` 的端点——例如前置 Bedrock 的公司网关——只要路由上的模型声明了任何推理档位，就会以 `Unexpected role "developer"` 拒绝整个请求。`reasoningEfforts` 可配置，这项推断却不可配置，于是此类路由只能把模型当作不推理的模型来服务（`reasoningEfforts: false`），为一个与模型能力毫无关系的原因放弃全部可选思考档位。

这个故障在最直观的验证方式下不可见。用 curl 探测该网关会通过，因为角色是在 pi-ai 内部依据模型的 `reasoning` 标志和 URL 选定的，而不是由作者手写请求中的任何字段决定；只有一次装配完成的真实运行才会暴露它。

## Decision

`PiAiCompatProfile` 增加第三个开关 `supportsDeveloperRole`，解析路径与前两个完全相同：模型条目 → 路由 → 已安装 catalog 条目 → pi-ai 按 URL 得出的猜测。填 `false` 会让系统提示留在 `system` 角色上，这正是角色受限的网关能够声明推理的前提。

该开关共用现有的 `compat` 块及其「仅 openai-completions」规则，因此这个块的措辞随之泛化：它承载的是 compat 开关，而不是「推理分派开关」。消息角色不属于推理分派；两条拒绝诊断现在只说 `compat switches` 而不再逐个列出字段，于是再加第四个开关时，不必在两条错误字符串里重述同一份清单。`definesCompatSwitch` 为模型级拒绝与路由级拒绝共同回答「这一层是否决定了任何开关」——此前这同一个条件被写了两遍。

## Consequences

位于角色受限网关之后的部署，现在可以为真正会推理的模型声明 `reasoningEfforts`，而不必为了让路由可用而剥除推理。pi-ai compat 面的其余部分（`supportsStore`、`maxTokensField`……）保持自动检测且特意不开放配置：只有当端点的 URL 确实会误导该推断时，才会开放对应开关，这就是 `packages/AGENTS.md` 为公开配置字段设定的证据门槛。

有两条错误消息的文本发生了变化。它们是加载期诊断，不承担任何协议或持久化职责，原先固定这些文本的测试已随本次改动一并更新。

## Testing

`tests/catalog.spec.ts` 覆盖新开关的「路由 → 模型」解析以及两条拒绝路径（在 `anthropic` 上设置模型级开关，以及路由级开关无任何模型可承接），且用例只设置 `supportsDeveloperRole`，从而证明新字段本身参与了每个判断，而不是搭前两个字段的便车。`tests/adapter.spec.ts` 借助 mock server 从两侧固定协议上的行为：未被识别的主机上，pi-ai 为推理模型推断出的 `developer` 角色；以及 `supportsDeveloperRole: false` 下变为 `system`、而 `reasoning_effort` 依然照常发出。正是这一对用例证明该开关不是空操作。

## Alternatives considered

- **保持推断不变，并把 `reasoningEfforts: false` 记为规避手段。** 被否决，因为这个规避手段牺牲的正是该部署付费换取的能力：模型会推理，网关也接受 `reasoning_effort` 与函数工具同时出现，唯一的阻碍只是角色。这是缺一个配置字段，不是模型能力不足。
- **把 pi-ai 的整个 `OpenAICompletionsCompat` 作为透传字典开放。** 被否决，因为那会把一个外部包的字段集合当作本包的配置面引入，既没有校验，也没有诊断，更无从维持「仅 openai-completions」这条规则的可信度。每个开关都要用一个「URL 推断确实出错」的实例来换取自己的位置。
- **从网关自身的错误中推导角色。** 被否决，因为到那时请求已经失败，而「据诊断重试」的路径会把一个加载期的配置事实变成逐请求的状态——部署方知道自己的后端接受什么，一次说清即可。
- **在 `compat` 之外新增专用字段（`systemRole: system | developer`）。** 被否决，因为解析顺序、协议限制与 catalog 继承行为与现有块完全一致；平行字段会把这三件事重复一遍，并让两者有机会漂移。
