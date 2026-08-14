# Agent Note: Configurable developer-role compat switch for pi-ai routes

Status: implemented

English | [中文](2026-08-14-pi-ai-developer-role-compat-switch.zh.md)

## Problem

pi-ai sends a reasoning model's system prompt under the `developer` message role whenever it reads the endpoint as OpenAI-compatible, which it infers from the endpoint URL: every host outside its short non-standard list (nvidia, cerebras, xai, together, deepseek.com, and a handful more) is assumed to accept the role. Nothing else is on that list, a private gateway and a vendor's OpenAI-compatibility endpoint alike, so an endpoint whose backend allows only `user`/`assistant` — a Bedrock-fronting corporate gateway, for one — rejects the whole request with `Unexpected role "developer"` the moment a route's model declares any reasoning level. `reasoningEfforts` was configurable and this inference was not, so such a route could serve the model only as non-reasoning (`reasoningEfforts: false`), which forfeits every selectable thinking level for a reason that has nothing to do with what the model can do.

The failure is invisible to the obvious verification. A curl probe against the gateway passes, because the role is chosen inside pi-ai from the model's `reasoning` flag and the URL, not by anything in the request an author writes by hand; only an assembled run shows it.

## Decision

`PiAiCompatProfile` carries a third switch, `supportsDeveloperRole`, resolving on the same path as the two before it: model entry → route → installed catalog entry → pi-ai's URL-derived guess. `false` keeps system prompts on the `system` role, which is what lets a role-restricted gateway declare reasoning at all.

The switch shares the existing `compat` block and its openai-completions-only rule, so the vocabulary that block uses generalizes: it holds compat switches, not "reasoning-dispatch switches". A message role is not reasoning dispatch, and the two rejection diagnostics now name `compat switches` without listing fields, so adding a fourth switch does not require restating a list in two error strings. `definesCompatSwitch` answers "does this layer decide any switch" for both the model-level rejection and the route-level one, which previously spelled the same condition out twice.

## Consequences

A deployment behind a role-restricted gateway can declare `reasoningEfforts` for a model that reasons, instead of stripping reasoning to keep the route usable. The rest of pi-ai's compat surface (`supportsStore`, `maxTokensField`, …) stays auto-detected and deliberately unconfigurable: each switch is offered only when an endpoint's URL provably misleads the inference, which is the evidence bar `packages/AGENTS.md` sets for a public configuration field.

Two error messages changed text. They are load-time diagnostics with no wire or durable role, and the tests that pinned them moved with the change.

## Testing

`tests/catalog.spec.ts` covers the new switch's route-then-model resolution and both refusals (a model-level switch on `anthropic`, and route switches no model on the route can take) with only `supportsDeveloperRole` set, so the new field counts in each condition rather than riding the older two. `tests/adapter.spec.ts` pins the behavior on the wire against the mock server from both sides: pi-ai's inferred `developer` role for a reasoning model on an unrecognized host, and `system` under `supportsDeveloperRole: false` while `reasoning_effort` still goes out. That pairing is what proves the switch is not a no-op.

## Alternatives considered

- **Leave it inferred and document `reasoningEfforts: false` as the workaround.** Rejected because the workaround costs the capability the deployment is paying for: the model reasons, the gateway accepts `reasoning_effort` beside function tools, and only the role blocks it. The gap is a missing configuration field, not a model limitation.
- **Expose pi-ai's whole `OpenAICompletionsCompat` as a pass-through dict.** Rejected because it would import an external package's field set as this package's configuration surface, with no validation, no diagnostics, and no way to keep the openai-completions-only rule honest. Each switch earns its place with a case where URL inference is wrong.
- **Derive the role from the gateway's own error.** Rejected because the request has already failed by then, and a retry-on-diagnostic path would turn a load-time configuration fact into per-request state — the deployment knows what its backend accepts and can say so once.
- **A dedicated field outside `compat` (`systemRole: system | developer`).** Rejected because the resolution order, the protocol restriction, and the catalog-inheritance behavior are exactly the existing block's; a parallel field would duplicate all three and let the two drift.
