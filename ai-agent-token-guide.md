# A Design Guide to Minimizing Token Consumption in AI Agents

[日本語](./ai-agent-token-guide.ja.md) | English

October 25, 2026 · [Kinopee](https://x.com/kinopee_ai)

This English edition is translated from the Japanese original. If the two versions differ, the Japanese version takes precedence.

## About this guide

This is a design guide to cutting wasteful token consumption by reviewing an AI agent's input, context, tool execution, verification, and model selection. It covers any setup in which models and tools work together, such as Claude Code, Codex CLI, Cursor, Gemini CLI, Devin, and custom-built agents. The ideas carry over, but setting names, billing, and available features differ by product, model, and version.

The goal is to **lower the total cost per task that passes verification, while meeting the required quality and completion time**. Shorter input is not an improvement if it brings more failures and reruns. Measure token counts, money, usage allowances, and wait time separately.

There are eight principles.

| Principle | Main measures | Metrics that confirm the effect |
| --- | --- | --- |
| 1. Reduce input | Narrow what gets read, instructions, searches, and tool results | Input breakdown, number of rereads, search success rate |
| 2. Manage context lifetime | Session splitting, conditional delegation, external memory, planned compaction | Combined parent and child consumption, cost of handoffs and rediscovery |
| 3. Reduce round trips | Make specs clear; move deterministic and intermediate processing into programs | Number of model calls, completion time |
| 4. Stop rework early | Divide work into units, verify in stages, set execution limits | Rework rate, time to detect failure, rate of hitting limits |
| 5. Reduce output | Avoid duplicating edits; report only what's needed | Breakdown of visible output, reasoning, and truncation |
| 6. Don't break the prompt cache | Keep the reused prefix stable | Cost by cache reads and writes, total cost |
| 7. Match the model and reasoning effort to the task | Choose, per task, a configuration that meets the quality bar | Cost per successful task, including retries |
| 8. Measure before you optimize | Compare one measure at a time under the same evaluation conditions | Changes in success rate, total cost, and completion time |

The numbers give the order of explanation, not a ranking of impact. Start with Principle 8 to see the breakdown, and tackle the dominant cost first. Where caching works well, a small amount of output or reasoning can make up a large share of the bill.

Each subsection opens with its recommended action in bold. If you're in a hurry, you can get by with just those lines, the "Applicability and failure conditions" note at the end of each principle, and the checklist at the end. The body text gives the reasoning behind each recommendation and its exceptions.

### How to read the evidence

Sources fall into the four types below. This is not a simple ranking of reliability. Each claim uses the kind of evidence that fits it: official documentation for API specifications, comparative experiments with clear conditions for savings, and observational logs for operational issues.

| Source type | What it tells you | Caveats when generalizing |
| --- | --- | --- |
| Preprints and research | Experimental conditions, comparison results, methods | Check whether it was peer-reviewed, the number of models and tasks, and how success was defined |
| Companies' own data | Trends in real-world use | Even with a large number of API calls, the data may come from a single person or company |
| Practitioner reports | Procedures you can try, concrete failure cases | If the comparison conditions or measurement methods are unclear, don't carry the savings figures over |
| Official specs and vendor evaluations | Supported features, settings, billing, the vendor's own evaluations | Keep evidence about specifications apart from evidence about performance, and attach the evaluation conditions to the latter |

This guide also distinguishes between comparative experiments, observational results, and design reasoning. It does not infer causation from an observed correlation alone. Specifications were checked on October 25, 2026. An individual savings figure doesn't guarantee the same savings across all of your tasks.

CLI, desktop, cloud execution, and direct API use are treated as separate surfaces. The settings below are official specifications for the product and execution mode named; don't assume that a feature with the same name is available on other surfaces. The official documentation used to verify each specification is linked at the relevant point in the text.

## Why agents consume tokens

Every time the model decides on its next action, the instructions, tool information, conversation, and execution results at that point become its input. Some implementations send the history from the client; others refer to conversation state stored on the server. **Skipping the resend over the network doesn't mean you stop paying for model input.** OpenAI's `previous_response_id` doesn't make past input free either. [OpenAI docs: Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)

The number of tool executions doesn't match the number of model calls either. A single model response can bundle several tool calls, and fetching and aggregation can be finished inside a program. Conversely, retries, compaction, and integrating the work of parent and child agents all add model calls.

### Separate token counts from cost

When you use an API directly, the basic formula is the sum of the cost of each model call.

```text
API cost = Σ (regular input × its unit price
            + cache writes × their unit price
            + cache reads × their unit price
            + output × its unit price)
          + additional fees for tools, execution environments, etc.

Total cost per successful task
  = total cost of the task set ÷ number of tasks that passed verification
```

Normalize the token categories so they don't overlap. With APIs that report reasoning as part of `output_tokens`, don't add the reasoning tokens a second time. Match unit prices to the conditions that apply, such as the model, cache retention period, service tier, and context length.

Failed attempts, model switches, parent and child calls, and fixes made after a review sends work back all go into the numerator. If needed, add human review time and execution environment costs as separate line items. On flat-rate plans, don't equate estimated API prices with what you're actually billed; record allowance consumption and any additional payments.

For example, suppose that on 10 tasks of similar difficulty, A costs $20 in total and passes 8, while B costs $24 and passes 10. The cost per passed task is $2.50 for A and $2.40 for B. This is a made-up example to illustrate the calculation, but it shows that a cheap request doesn't mean a cheap result. If nothing passes, the ratio is undefined; report the failure rate and the cost as they are.

### What measurements do and don't tell us

- Manus reports an input-to-output ratio of about 100:1 in its own operations. This isn't a ratio shared by all agents, and it isn't a cost ratio. [Manus's operational report](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- Across 287,748 API calls from a single developer, LangWatch observed that the cumulative token consumption needed to reach a given context length grew roughly with that length to the power of 2.55. The relationship includes the growth in the number of steps; it is not a law saying that doubling the context always makes a single API call about six times as expensive. [LangWatch's observational analysis](https://langwatch.ai/blog/context-tax-when-to-compact)
- In a specification-comparison experiment with Kimi K3, output was only 2.7% of processed tokens yet 51.1% of the cost. This breakdown reflects that experiment's conditions, including a 96.3% cache hit rate. Whether to prioritize cutting input or output can't be decided by volume alone. [Smékal paper, §4.3](https://arxiv.org/html/2608.25399v1)

In a simple model without caching, where each step's input grows by a fixed amount, cumulative input grows quadratically with the number of steps. In practice, though, compaction, retrieval, parallel processing, and pricing categories all change the picture. Use "number of calls × average input" as a rough estimate of consumption, and use actual usage data and unit prices to evaluate the bill.

## Principle 1: Reduce input

Reach the information you need by a short path, and don't pile unneeded information into the history. Cutting important specifications or errors leads to more re-exploration, so think "pass only what's needed" rather than "make it shorter."

### 1-1. Specify what to read, and separate exclusions by purpose

**Put the target files, functions, symptoms, and related tests in the request, and exclude generated artifacts through the product's permission settings. Protect secrets with a separate mechanism.**

Pointing to the target file, function, symptom, and related tests gives exploration a clearer starting point than "fix the auth stuff." That said, the file you name isn't necessarily the cause. Leave the agent free to read dependencies that turn out to matter during the investigation.

```text
Investigate refreshSession() in src/auth/refresh.ts.
Repro: POST /session/refresh with an expired refresh token returns 500.
Related test: tests/auth/refresh.test.ts.
Read this scope first, and expand to dependencies only if the cause is in another layer.
```

Build artifacts, the contents of installed dependencies, and large generated files are candidates for exclusion from routine exploration. Don't exclude lockfiles or generated code across the board, though. They're needed for dependency updates, vulnerability investigations, and bugs in code generators.

**A `.claudeignore` file has no effect in Claude Code.** To deny the built-in file reads, use `permissions.deny` in `.claude/settings.json`. The following example denies reads of generated artifacts. It is a denial, not just a hint about search priority. [Claude Code docs: Read and Edit](https://code.claude.com/docs/en/permissions#read-and-edit)

```json
{
  "permissions": {
    "deny": [
      "Read(./dist/**)",
      "Read(./coverage/**)"
    ]
  }
}
```

This setting is a stronger restriction than saying "don't normally read these" in the request. Under the current spec, a `Read` deny also denies `Edit` and `Write` on the same paths, so revisit the targets for tasks that need them. Don't treat `Read` rules as complete access control over arbitrary programs run in the shell, or over MCP. Curbing exploration and protecting secrets are separate designs.

Cursor's `.cursorignore` does work: it blocks the listed files from Agent, Tab, and @ references. Because it blocks access rather than lowering search priority, keep it, like a `Read` deny, from covering files that some tasks need, such as lockfiles. It doesn't apply to the terminal or MCP. If you need strong access control, combine the product's permission settings, OS permissions, and a sandbox. [Cursor docs: Ignore file](https://cursor.com/docs/reference/ignore-file)

In dedicated environments, place only the repositories you need. Splitting huge files by responsibility is worthwhile for maintainability too. Still, there's no need to start a sweeping restructuring just to save tokens in the short term.

### 1-2. Keep instruction files short and audit outdated instructions

**Limit always-on instructions to constraints, verification commands, and key paths, and move area-specific rules into conditional rules or documents read on demand. Prune outdated instructions with every model update.**

With AGENTS.md, CLAUDE.md, `.cursor/rules`, and similar files, whatever gets loaded affects all subsequent input. Application conditions, hierarchy, and whether files are loaded lazily vary by product.

- Keep the constraints, verification commands, and key paths that are always needed in the main file.
- Move rules for specific areas into conditional rules, and detailed procedures into documents or skills that are read only when needed.
- When models are updated, review instructions written to make up for old weaknesses, such as "always verify twice" or "always read every file." Compare the same tasks before and after removing them to confirm quality.
- Clean up unused skills and plugins based on the actual context breakdown. The full text of everything installed isn't necessarily loaded every time.

The official Claude Code docs suggest keeping each CLAUDE.md under 200 lines. But line count is not token count, and it isn't a limit shared by other products. **`@path` imports in CLAUDE.md load the referenced files as well, so splitting alone saves nothing.** Distinguish between referencing a file when needed and importing it automatically. [Claude Code docs: Memory](https://code.claude.com/docs/en/memory)

In Codex, global instructions are combined with instruction files discovered along the path from the project root to the current working directory. At each directory, Codex selects at most one file, checking `AGENTS.override.md`, then `AGENTS.md`, then configured fallback names. The combined-size limit, `project_doc_max_bytes`, defaults to 32 KiB, not a token count. **Splitting files does not reduce input if the resulting files are still on the discovery path.** Keep shared constraints at the root, area-specific rules in their directories, and long procedures in documents or skills read on demand. Check which instructions load from each starting directory; don't lower the cap to discard necessary rules. [Codex docs: AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md)

In Cursor, use AGENTS.md for instructions in plain Markdown. Product-specific project rules are `.mdc` files in `.cursor/rules/`; putting a regular `.md` file in the same place doesn't make it a project rule. [Cursor docs: Rules](https://cursor.com/docs/rules)

In Devin, the first 16 KiB of each AGENTS.md file is automatically included in context; the rest can be read on demand. Put critical constraints and verification commands first, and move long procedures into Skills or referenced files. Memory carries personal preferences and lessons across sessions; put mandatory team procedures in AGENTS.md or Skills. [Devin docs: AGENTS.md](https://docs.devin.ai/onboard-devin/agents-md), [Skills](https://docs.devin.ai/product-guides/skills), [Memory](https://docs.devin.ai/product-guides/memory)

```markdown
## Always
- Keep the public API backward compatible.
- Run verification that matches the change; if you skip it, say why.
- For DB changes, read docs/db-migrations.md.
- For auth changes, read docs/auth-contract.md.
```

The point that outdated procedures add unnecessary tool round trips and output also matches Anthropic's official guidance on auditing prompts. Audit regularly and keep instructions short but necessary. [Anthropic docs: Audit prompts against the current model](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence#audit-prompts-against-the-current-model)

### 1-3. Use a code map that fits the purpose

**Use a partial Read for known locations, `rg` for fixed strings, and LSP for references; reserve indexes and graphs for cross-cutting investigations.**

There's no need to consult a code graph first every time. Use the criteria below to pick the method that reaches the information you need at the lowest exploration cost.

| What you're looking for | What to try first | When to go further |
| --- | --- | --- |
| Known file and change range | Read the relevant range | Widen the scope if assumptions or callers are unclear |
| Identifiers, error messages, fixed strings | `rg`, symbol search | Add structural information if there are many matches with the same name |
| Definitions, references, call relationships | LSP, code graph | For dynamic calls, also check the implementation and runtime results |
| Related code by meaning | Semantic search | Confirm the meaning in the candidates' actual code |
| Cross-cutting dependencies and impact | Graph plus reading the needed parts | Check when the index was last updated and which languages it covers |

For example, use `rg -n -F 'refresh token expired' src tests` for a fixed error message, and the LSP's find-references for callers. Don't just aim for fewer search results; also record what was missed and which searches had to be repeated.

| Tool type | Examples | Caveats |
| --- | --- | --- |
| Structure and symbol information | Serena, Aider repo map, Tree-sitter-based tools, code-review-graph | Strengths vary by language and analysis method |
| Semantic search | Embedding-based code search, Claude Context, etc. | Results are related candidates, not proof of actual dependencies |
| Repository packing | Repomix, etc. | Packing everything into one file is not the same as making the input smaller |

Separating exploration from implementation, and reading at the function or class level, are useful design options. Don't conclude from secondary articles alone that research has established a general effect; compare the options in your own environment. The official docs also describe how navigating by symbol cuts unnecessary file reads. [Claude Code docs: Code intelligence](https://code.claude.com/docs/en/costs#install-code-intelligence-plugins-for-typed-languages)

Cursor's current Search documentation describes Instant Grep, which searches for strings and regular expressions, and the Explore subagent, which handles broad exploration. The old "Codebase Indexing" URL now leads to this page as well. Don't equate the current built-in search with the embedding approach described in older blog posts; keep it separate from semantic search in general. [Cursor docs: Search](https://cursor.com/docs/agent/tools/search)

### 1-4. Trim tool output, but keep a way back to the original

**Save test and build output to a file, and return only the exit code, the failure locations, and the path to the full log to the model.**

There's no need to pass along the list of passing tests, huge diffs, or repeated warnings every time. Return the status, the important parts, and where the full output is saved.

```bash
# Bash example. Replace npm test -- auth with your project's command.
mkdir -p .agent-logs
test_log=$(mktemp .agent-logs/auth-test.XXXXXX)
if npm test -- auth >"$test_log" 2>&1; then
  test_status=0
else
  test_status=$?
fi
printf 'exit_code=%s full_log=%s\n' "$test_status" "$test_log"
if [ "$test_status" -ne 0 ]; then
  tail -n 80 "$test_log"
fi
exit "$test_status"
```

The last 80 lines is just an example, not a length that guarantees the diagnostics you need. If the cause isn't visible, read the relevant range of the full log. Filtering down to `ERROR` lines can also drop the preceding command, stack traces, and related warnings. Preserve the exit code, and design things so that short output is never taken as a sign of success.

Output wrappers such as RTK are another option. Its "up to 90% reduction" mainly refers to fewer bytes of command output, not a reduction in your bill. RTK's token figures are estimates computed as `bytes / 4`. Don't treat them as exact billable token counts, especially for code or for non-English text such as Japanese. [RTK repository: How RTK Savings Work](https://github.com/rtk-ai/rtk/blob/develop/docs/guide/resources/savings-explained.md)

In Claude Code, `BASH_MAX_OUTPUT_LENGTH` is set in **characters**, while `MAX_MCP_OUTPUT_TOKENS` is set in tokens. The former is ignored if the `bashOutputMaxChars` setting is present. As for the latter, if an MCP tool declares `anthropic/maxResultSizeChars`, that character limit is applied to its text instead, and large successful results may be saved to a separate file. These variables alone can't cap every tool result at the same length. Select the information you need on the search or test side first. [Claude Code docs: Environment variables](https://code.claude.com/docs/en/env-vars)

### 1-5. Judge tool definitions by how much actually gets loaded

**Check the context breakdown to see how much space tool definitions actually take up, then remove the MCP servers and plugins you don't use.**

Tool names, descriptions, and argument schemas add to the input, and too many options also increase exploration. You can check how much they occupy with Claude Code's `/context`, the context usage display in Cursor's chat, and similar tools (section 8-2). Connections the task doesn't need are candidates for removal, but it isn't necessarily true that every MCP definition is sent in full every time. In Claude Code, with default settings on supported models and platforms, tool search loads definitions only when they're needed. Older models, configurations with tool search disabled, and some platforms load them up front, so don't treat deferred loading as behavior shared by every environment. [Claude Code docs: MCP / Configure tool search](https://code.claude.com/docs/en/mcp#configure-tool-search)

For custom agents, compare a small set of dedicated tools, deferred loading through tool search, and caching of stable definitions. The effect of changing definitions partway through is covered in Principle 6.

**Applicability and failure conditions:** Evaluate input reduction including the rereads and retries caused by missing information. Use a partial Read when the target is known and an index for unknown, cross-cutting investigations. If the goal is protecting secrets, verify the access boundary separately from search exclusions meant to save tokens.

## Principle 2: Manage context lifetime

Don't keep carrying unrelated history; draw boundaries in a way that lets the necessary state carry over. Splitting, delegation, external memory, and summary-based compaction each have their own costs, and none of them is the right choice every time.

### 2-1. Start a new session when the task changes

**When you move to an independent task, open a new session and carry over only the necessary state, in the format from section 2-3.**

If you move on to an unrelated UI adjustment after fixing an auth bug, a new session is a good candidate. If the next task closely continues the previous work, continuing with the working state intact may be cheaper.

Use Claude Code's `/clear`, a new chat in Cursor, `/clear` in the Cursor CLI, and so on. `/resume` resumes a past conversation; it doesn't shrink the history. If you need a handoff, save it in the format described in section 2-3.

Coordinator-style features such as "Projects" in Cursor and Claude Code automate this separation by launching a worker with a fresh context for each task. They don't reduce total consumption, though. The coordinator reading worker reports and making decisions adds cost, as do parallel workers and restarts for PR monitoring. Sending more instructions to an old worker whose cache has expired means rereading the entire conversation, so for new work a fresh worker can be cheaper. Shared instructions and memory grow along with the Project, so also check whether the instruction-file bloat discussed in Principle 1 is happening at the Project level. Availability, default models, and the impact on usage allowances are summarized in "Don't confuse product usage allowances with API billing" and in the comparison table.

In Codex, continue the current conversation for follow-up investigation of the same bug, and start a new conversation with a handoff for unrelated implementation work. For a single long task with growing context, preserve the state listed in section 2-4 and consider `/compact`. Resuming, starting fresh, and compacting are different operations; compaction does not reset usage allowances. A new conversation still needs shared instructions and may reread code, so include both sides of the switch in the comparison. [Codex docs: Developer commands](https://learn.chatgpt.com/docs/developer-commands?surface=cli)

In Devin Cloud, consider continuing the existing session for follow-up work on the same fix and starting a new one for independent work. A new session boots from a snapshot; changes made in a previous session do not automatically update that snapshot. Hand off commits, artifacts, and verification results rather than assuming that a local path also shares the file or running state. [Devin docs: Environment](https://docs.devin.ai/onboard-devin/environment)

### 2-2. Use subagents when the benefit of isolation outweighs the cost

**Delegate only independent investigations whose large intermediate results can be boiled down to a small conclusion, and write the scope, return format, and limits into the instructions.**

Subagents suit cases where you want to bring back only the needed conclusion without leaving a large volume of investigation results in the parent's history. A child's consumption isn't free, though. Compare costs including the child's startup, instructions, exploration, and report, as well as the parent's work integrating the results.

| When delegation fits | When a single agent fits |
| --- | --- |
| An independent investigation with clearly bounded deliverables | Short, sequential work that depends on the previous decision |
| Large intermediate results can be reduced to a small deliverable | The file is known and only a few lines need fixing |
| Parallelization cuts wait time | Parent and child would just reread the same information |
| Ownership can be divided and the integration verified | Frequent conflicting edits to the same files |

```text
Scope: investigate the refresh token expiry check only. Do not edit anything.
Input: src/auth/refresh.ts, related tests, current symptoms.
Return: relevant files and symbols, evidence, unverified points, and next checks, in about 10 lines.
Keep the evidence behind open questions; for long logs, give the saved location.
```

Prefer a fresh context, and pass only the necessary specs and state from the parent. Use a fork that inherits the full context only when the re-exploration it avoids outweighs the cost of duplicating it. A fresh context still loads shared instructions and tool information, so its input is never zero.

Reusing the same child for related follow-up investigations can sometimes avoid re-exploration. Rather than making "always discard after one use" a rule, decide each child's scope, lifetime, and call and budget limits. Don't pile unrelated work onto a long-lived child. See Principle 7 for model selection and section 4-5 for stop control. [Claude Code docs: Subagents](https://code.claude.com/docs/en/sub-agents), [Codex docs: Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents), [Cursor docs: Subagents](https://cursor.com/docs/subagents)

### 2-3. Use the file system as external memory

**Write the goal, constraints, changes, verification results, open issues, and next action to a progress file, and read them back from the file rather than from the history.**

Save progress, decisions, and reproduction steps, and read back only the parts you need. A URL or file path alone won't survive the content being changed or deleted, so when you need it for reproduction, also keep the retrieval time, the commit, and an excerpt or snapshot. Externalizing state is a design that shifts cost to saving and retrieving it. [Manus's external memory design](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

```markdown
# progress.md
Updated: 2026-10-25 / Commit: record the actual SHA
Goal: return 401 for expired refresh tokens.
Constraints: keep the public response format; add no new dependencies.
Changes: added an expiry check to src/auth/refresh.ts.
Verification: related tests pass. Full type check not run yet.
Open: consistency with the existing expiry boundary tests.
Next: run the type check and review differences in boundary conditions.
Evidence: the relevant logs in .agent-logs/, paths to the related tests.
```

If the original sources, the working notes, and the current code disagree, re-check the current state. There's no need to copy secrets into handoff documents.

### 2-4. Compact long single tasks deliberately

**For tasks that can't be split, don't wait to hit the limit; compact at a natural breakpoint, leaving enough headroom for the remaining work. Save verbatim information to a file first.**

Summary-based compaction replaces past history with a shorter representation so the work can continue. If tasks are independent, split them; if a single long task has to continue, compact while keeping headroom for output and reasoning. There's no need to put it off until you reach the limit.

LangWatch observed that user corrections increased right after compaction. However, which information was used was estimated by an LLM judge, and corrections were detected by matching phrases; compaction may also simply happen more often at difficult moments. It isn't an experiment that establishes a causal effect of compaction. The figures below are estimates from the same source that include the cost of recovering quality, and they are reference values from one person's setup with a window of about 1 million tokens. [LangWatch: observations and estimates on compaction](https://langwatch.ai/blog/context-tax-when-to-compact)

| Work | Reference threshold in the source |
| --- | --- |
| PR work | 200K to 250K tokens |
| Investigation | 250K to 300K |
| QA | 250K to 350K |
| Implementation, code comprehension | 300K to 450K |

Don't use these as default settings or scale them proportionally to smaller windows. Nor can they be turned into general rules like "always compact late" or "compaction is a last resort." Base the decision on the following.

- The amount of work remaining, and the headroom needed for the input, output, and reasoning ahead.
- Whether you need verbatim information such as exact strings, unfinished edits, or failing tests.
- The cost of generating the compaction, rebuilding the cache right afterward, and rereading and rework.
- The input cost you can save on the remaining calls after compaction.

Before compacting, preserve the goal, constraints, changed files, exact errors, verification results, open issues, and next action. There are two ways to do this. Either write them to the progress file from section 2-3 first and then compact, or specify what to keep through the compaction command's arguments or an instruction file, as with Claude Code's `/compact <instructions on what to keep>`. If your implementation can keep a short summary alongside recent history, compare that option too. There's no universally optimal length for the history you keep.

**Applicability and failure conditions:** If the parent's input goes down but the total goes up once the children and rediscovery are added, don't adopt it. If the agent rereads many of the same files right after compaction or a session switch, fix the content of the handoff or where you draw the boundary.

## Principle 3: Reduce round trips

Fewer model calls can cut input, generation, and wait time all at once. Keep the judgments that are actually needed, and reduce how often deterministic processing has to return to the model midway.

### 3-1. Don't cut the meaning out of the specification

**Cut only greetings and redundant background from requests, and keep the expected behavior, constraints, boundary conditions, relevant locations, and completion criteria.**

The things to keep are the expected behavior, constraints, important boundary conditions, relevant locations, and completion criteria. Greetings and redundant background can go. Locking in an implementation approach whose rationale you haven't confirmed can send the investigation on a detour. Cutting the meaning out of a spec to shorten the input tends to increase round trips and retries, pushing the total cost up.

The supporting evidence is Smékal's experiment: 2,700 runs covering Kimi K3, five SWE-bench Verified tasks, 12 kinds of specification, three reasoning-effort levels, and 15 repetitions each. The paper's main analysis covers 11 of the specifications and estimates a 29.7% cost increase when a full specification is cut down to just a user story. The abstract describes this in terms of token spend, but the body uses cost at list prices as its primary metric. The uncertainty is wide: the paper reports a 90% credible interval of +5.1% to +58.7%, and widening it to 95% gives −0.3% to +66.8%, which includes zero (the posterior probability that the effect is positive is 0.97). Variation across tasks is also large; the same reduction raised costs by 13% on one task and by 115% on another. This is a result from a single model on a limited set of tasks, and it doesn't mean longer prompts always win. [Paper §3 / Table D1 / Appendix C](https://arxiv.org/html/2608.25399v1)

### 3-2. State constraints and decision criteria up front

**Put settled constraints, the public API, and completion criteria at the top of the request, and keep them distinct from points that may be decided during investigation.**

```text
Implement a token-bucket rate limiter in TypeScript.
No external dependencies. Use performance.now() for time, and make it injectable for tests.
The public API is async acquire({ timeoutMs, signal }).
Waiters are served in FIFO order. Distinguish timeouts from cancellations.
Prioritize compatibility with the existing API. If there are contradictions that need a decision, raise them all together before starting.
Done when: tests for capacity limits, refill, wait order, timeout, and cancellation pass.
```

Specifying all of this doesn't guarantee a correct result on the first try. Separate settled conditions from those that may be decided during investigation, to cut down on costly wrong assumptions.

### 3-3. Leave deterministic work to tools

**Leave formatting and mechanical auto-fixes to formatters and linters, and build them into on-save actions, pre-commit, and CI.**

Move formatting and fixes that can be determined mechanically to Prettier, ESLint's auto-fix, Ruff, and similar tools. Building them into pre-commit, CI, or the editor's on-save actions cuts the round trips spent asking the model to format code.

For rules whose auto-fix might change meaning, check with diffs and tests. Split verification such as type checks and tests into the stages described in section 4-3.

### 3-4. Turn prompts that worked into shared procedures

**Save prompts that worked in the repository with their scope and last-updated date, and have the agent read them only for tasks that need them.**

Keep requests with complete reproduction steps, investigation scope, constraints, and completion criteria as procedures in the repository. Instead of pasting the full text every time, have the agent read only the procedures needed for the related work.

Give procedures a scope and last-updated date too. Don't keep excessive verification steps or fixed search orders written for older models forever just because "it worked before."

In Devin, reusable procedures can live in `.agents/skills/<skill-name>/SKILL.md`. Define environment setup in a blueprint and reuse it through snapshots to reduce repeated setup discovery and environment repair. `initialize` covers tools, `maintenance` covers dependencies, and `knowledge` holds reference information such as verification commands. `maintenance` runs during builds; at session start it is surfaced as reference, not automatically executed. After approving and building an environment change, verify it in a new session. [Devin docs: Skills](https://docs.devin.ai/product-guides/skills), [Blueprints](https://docs.devin.ai/onboard-devin/environment/blueprints), [Blueprint reference](https://docs.devin.ai/onboard-devin/environment/blueprint-reference)

### 3-5. Consolidate tool calls and keep intermediate results away from the model

**Do bulk fetching, filtering, and aggregation in a script, and return only the aggregated results and references to the originals to the model.**

Carry out fetching, filtering, aggregation, and similar steps inside a program, and return only the final information needed to the model. Compared with simply returning several full results at once, this can cut intermediate data.

```text
Separate round trips: model → search → model → fetch → model → aggregate → model
Consolidated:         model → program [search → fetch → aggregate/extract] → model
```

For example, to check 50 CI jobs, a program fetches the status of each job and returns only the number of failures, the affected jobs, key errors, and links to the originals. There's no need to pass the huge logs of successful jobs to the model.

```python
# Example showing the design. Implement jobs, get_job, and extract_error with your own APIs.
import asyncio

gate = asyncio.Semaphore(4)


async def inspect(job):
    async with gate:
        try:
            result = await get_job(job, timeout_seconds=30)
            return {
                "job": job,
                "status": result["status"],
                "error": extract_error(result),
                "source": result["url"],
            }
        except Exception as exc:
            return {"job": job, "status": "fetch_failed", "error": str(exc)}


async def build_report(jobs):
    results = await asyncio.gather(*(inspect(job) for job in jobs))
    return {
        "checked": len(results),
        "passed": sum(r["status"] == "passed" for r in results),
        "needs_attention": [r for r in results if r["status"] != "passed"],
    }


if __name__ == "__main__":
    report = asyncio.run(build_report(jobs))
    # Return report to the model. Don't treat fetch failures as successes; keep the originals reachable.
```

Independent reads can be parallelized with a concurrency cap. Dependent steps run in order inside the program. If a judgment about meaning is needed midway, hand control back to the model at that point. Operations that write data or require approval don't need to be automated unconditionally just for the sake of consolidation.

Anthropic's Programmatic Tool Calling provides this approach by combining a code execution environment with tool integration. It requires a supported code execution tool and the `allowed_callers` setting on the tools to be called. **Tools reached through the same API's MCP connector, and the computer use and browser use toolsets, can't be called from the program.** Distinguish between a design in which an ordinary script calls your own APIs and this particular Anthropic API feature. [Anthropic docs: Programmatic tool calling / Tool restrictions](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling)

In the official evaluation, input on search benchmarks fell by 24% on average, while on τ²-bench, which makes one or two sequential calls per turn, scores were unchanged and costs rose by about 8%. These results come from different benchmarks and metrics; they aren't two sides of the same conditions. The evaluation conditions are described in the same official documentation.

When deciding whether to adopt it, compare costs including code generation, containers, external APIs, and reruns on failure. Distinguish "parallel calls," which simply issue ordinary tool calls at the same time, from approaches that keep intermediate results out of the model.

**Applicability and failure conditions:** Promising for bulk fetching, filtering, and aggregation. For a few short calls, or work that needs the model's judgment at every step, the cost of setting up the program may outweigh the gains.

## Principle 4: Stop rework early (task design and verifiers)

Find mistakes early, and don't keep repeating a failed approach out of inertia. Verification also costs runtime and input, so set up stages that match the impact of the change.

### 4-1. Split work into verifiable units and settle key decisions first

**Break large changes into units whose behavior can be checked, and confirm a short plan first only when there are design choices or breaking changes.**

For "replace auth with OAuth," break the work into units whose behavior can be checked, such as token acquisition, the callback, and wiring up the UI. Move on only after each unit passes verification. Adding more checkpoints doesn't mean each one needs human approval or a new session.

When there are multiple design choices, breaking changes, or unsettled specs, do a short plan and confirmation first. For small fixes with a clear target, you can skip the round trips of drafting a plan. Claude Code's plan mode, Codex CLI's `/plan`, and Cursor's Plan Mode are options for separating investigation from implementation. Check how they behave in your product and version before relying on them. [Claude Code docs: Best practices](https://code.claude.com/docs/en/best-practices), [Codex docs: Developer commands](https://developers.openai.com/codex/cli/slash-commands), [Cursor docs: Plan Mode](https://cursor.com/docs/agent/plan-mode)

### 4-2. Hand over completion criteria and verification methods up front

**Provide the completion criteria, boundaries, verification to run, and how to report skipped checks with the request, and state explicitly that "not run" doesn't mean "passed."**

```text
Done when: an expired refresh token returns 401, and a valid one succeeds in the existing format.
Boundaries: distinguish exactly-at-expiry, missing, and malformed tokens.
Verification: run the related auth tests and the project's type check.
If there are pre-existing failures, check how they relate to this change, and report the results and where the logs are saved.
Mark any verification you can't run as "not run"; don't treat it as passed.
```

Passing tests doesn't prove that the whole spec is met. Depending on the change, include reproduction checks, diff review, and visual checks of the UI in the completion criteria. Also make it an explicit working condition that expected values and verification targets must not be improperly weakened to get tests to pass.

### 4-3. Run verification in stages, starting with light checks

**Run lint and syntax diagnostics on changed files right after editing, related tests and type checks when a unit of work is done, and full verification before the PR.**

| When | Example checks | How much to run |
| --- | --- | --- |
| Right after an edit | Formatting, lint, and syntax diagnostics on changed files | Focus on fast checks. Consecutive edits can be batched |
| When a small unit of work is done | Related unit tests, type checks | Pick a meaningful scope that includes dependencies |
| At task completion, before the PR | The full tests you need, integration tests, CI | Widen according to impact, such as public API or dependency changes |

Running a full build and the whole test suite every time repeatedly flags temporary inconsistencies from half-finished edits, adding wait time and output. Cheap static diagnostics, on the other hand, are worth running early. When several edits make up a single change, verify them as a group.

Here is an example in Claude Code's `.claude/settings.json` that runs lightweight checks after file edits. It targets environments where Bash is available. Merge it into your existing settings, and have `check-light.sh` look at the target files, the changes, and which versions it has already checked, so it avoids duplicate runs.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash",
            "args": ["${CLAUDE_PROJECT_DIR}/.claude/hooks/check-light.sh"]
          }
        ]
      }
    ]
  }
}
```

When `args` is specified, Claude Code launches `command` directly as an executable without going through a shell (exec form). If `args` is omitted, the `command` string is passed to a shell (shell form). When the command contains path placeholders, the official docs recommend exec form because its quoting is simpler. `${CLAUDE_PROJECT_DIR}` is the project root at session start. Even if the worktree or working directory changes during the session, the path to this script stays the same. Determine where the code to verify currently lives from fields such as `cwd` in the JSON the hook receives on standard input. Keep output brief on success, and return diagnostics and the path to the full log on failure. In Claude Code, `PostToolUse` can send feedback through exit code 2 and standard error, but it can't undo an edit that has already happened. Matching only `Edit|Write` won't catch changes made through the shell or external processes. [Claude Code docs: Hooks](https://code.claude.com/docs/en/hooks)

Codex has `PostToolUse` and Cursor has `postToolUse`, but their input formats, tool names, and return fields differ. In Codex, `Edit` and `Write` also work as matcher aliases for `apply_patch`, and additional context goes in `hookSpecificOutput.additionalContext`. Cursor uses `additional_context`. Cursor's `updated_mcp_tool_output` is for replacing MCP results; it isn't a field for replacing general terminal output. Merely adding context afterward doesn't reduce the input from the original huge result. [Codex docs: Hooks](https://developers.openai.com/codex/hooks), [Cursor docs: Hooks](https://cursor.com/docs/hooks)

### 4-4. Control pre-completion verification and stop-hook retries

**Have Stop-type hooks continue only when there are unverified changes or fixable failures, and end the run as incomplete when the same cause keeps recurring or the problem is external.**

Before finishing, make sure the verification this change requires has been done. You can skip rerunning a check that has already passed if the following hold.

- The code, tests, dependencies, and configuration under verification haven't changed since.
- The verification result is saved together with the revision or changes it applies to.
- There's no reason to rerun it, as there would be for checks that depend on the time or the external environment.

If you use Stop-type hooks to continue automatically, continue only when there are unverified changes or fixable failures. For external outages, insufficient permissions, or the same cause recurring, make sure the run can record what's incomplete and end.

Claude Code's `Stop` can continue the run with exit code 2, and Codex's `Stop` with `decision: "block"` and the like. Cursor's `stop` works by returning a `followup_message`. Even though they're framed as blocking the stop, all of these actually trigger additional model processing. Don't set up a hook that always returns failure. [Claude Code docs: Stop](https://code.claude.com/docs/en/hooks#stop), [Codex docs: Stop](https://developers.openai.com/codex/hooks), [Cursor docs: stop](https://cursor.com/docs/hooks)

On the CI side, make the required status checks a condition for merging. Local stop hooks and repository merge requirements play different roles. [GitHub docs: Protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)

### 4-5. Set execution limits separately from instructions

**For non-interactive runs and child agents, set limits on turns, time, and budget in configuration, and record hitting a limit separately from completion.**

Writing "do just this job and then stop" won't reliably prevent loops. Put limits on model calls, tool calls, elapsed time, and the combined parent-and-child budget.

| Implementation | Available controls | Limitations |
| --- | --- | --- |
| Claude Code | `--max-turns` and `--max-budget-usd` in print mode, `maxTurns` for subagents | Turns are not the same as tool calls. The budget is an estimate and can be exceeded mid-run |
| Cursor | `loop_limit` on `stop` / `subagentStop` | Caps automatic continuation, not the budget for the whole session. Defaults to 5 per script |
| Codex CLI | `agents.max_concurrent_threads_per_session` (the number of child threads that can be open at once; formerly `agents.max_threads`), plus time and call monitoring on the execution side | Limits concurrency, not turns or spend. Within the scope of this review, no general CLI setting that reliably caps the overall total could be confirmed |
| Custom agents | Budget checks before each call, timeouts, failure counts, budget allocation to children | The last request can push you over, so leave a margin |

[Claude Code docs: CLI reference](https://code.claude.com/docs/en/cli-reference), [Subagents](https://code.claude.com/docs/en/sub-agents), [Codex docs: Configuration Reference](https://developers.openai.com/codex/config-reference), [Cursor docs: Hooks](https://cursor.com/docs/hooks)

Claude Code's `--max-budget-usd` includes the cost of subagents during the current run, but not past runs restored with `--continue` or `--resume`. Track a task's total budget across multiple launches outside the tool. A subagent's `maxTurns` is also not a permanent cap on the whole task, since the subagent can be resumed after reaching it.

Claude Code's Stop hook has a cap on consecutive continuations (8 by default, changeable with `CLAUDE_CODE_STOP_HOOK_BLOCK_CAP`), but tool calls reset the consecutive count, so it can't guarantee that the session as a whole will stop. Check `stop_hook_active` and similar fields to avoid loops, and set an overall limit separately. [Claude Code docs: Stop input](https://code.claude.com/docs/en/hooks#stop)

As an operating rule, you might decide: "If the same error occurs twice in a row, update the hypothesis about the cause; if no new evidence turns up, return the task as incomplete." The count is just an example, and there's no need to treat transient network failures the same as logic errors in the code. Don't count hitting a limit as completion; record the remaining work and the conditions for resuming.

**Applicability and failure conditions:** Evaluate the cost of verification together with the rework it prevents. Hooks that repeat heavy checks when nothing has changed, wrappers that swallow failures, and Stop hooks that can never stop all need fixing. This staging is a design recommendation, not an experimental result that guarantees a particular savings rate.

## Principle 5: Reduce output

Keep the deliverables and verification results you need, and avoid generating the same content over and over. Output costs money on the spot and, to the extent that it stays in the history, it also affects the next input. Look at visible text, tool arguments, and internal reasoning separately.

### 5-1. Don't repeat edits in the final response

**Apply edits with editing tools, and keep the final response to the changes, verification results, and open issues. Repeat full files or diffs only when asked.**

For agents that can edit files directly, make changes with the editing tools, and keep the final response to a brief account of the changes, verification, and open issues. Always requiring full files or diffs to be repeated adds duplication.

```markdown
## Output style
- Apply file changes with the editing tools.
- In the final response, briefly report the main changes, verification results, and open issues.
- Show full edited files or diffs only when asked or when needed for review.
- For investigations, leave the conclusion, where the evidence is, and any uncertainties.
```

When a person applies changes from the chat, returning a unified diff or the range to replace works well. A full rewrite of a document or delivery of generated artifacts needs the complete file. Don't apply the partial-edit rule as is.

Telegraphic styles such as Caveman are also an option, but evaluate them including readability, ambiguity, and any increase in back-and-forth. There's no need to make the prose unnaturally terse or to drop evidence and conditions.

### 5-2. Distinguish verbosity from output limits

**Lower the level of detail in responses through verbosity settings or instructions, and keep output limits at values that don't cause truncation.**

Verbosity controls how detailed a response is; the output limit caps how much can be generated. Simply lowering the limit can cut responses off midway and lead to more regeneration.

| Product / API | Example setting | Notes |
| --- | --- | --- |
| Claude Code | `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | Output limit for most requests. Not a cap on the task total. The model's own limit also applies |
| Codex | `model_verbosity = "low"` on supported models | Reasoning effort is a separate setting. Doesn't have the same effect on every model |
| OpenAI Responses API | `max_output_tokens`, `text.verbosity` on supported models | The limit includes reasoning. Check for `incomplete` |
| Anthropic Messages API | `max_tokens` | In normal generation, if `stop_reason` is `max_tokens`, check whether the content was cut off |

[Claude Code docs: Environment variables](https://code.claude.com/docs/en/env-vars), [Codex docs: Configuration Reference](https://developers.openai.com/codex/config-reference), [OpenAI docs: Reasoning models](https://developers.openai.com/api/docs/guides/reasoning), [Anthropic docs: Create a Message](https://platform.claude.com/docs/en/api/messages/create)

### 5-3. Shorter visible output doesn't necessarily mean less reasoning

**To cut reasoning tokens, change the reasoning-effort setting rather than the response length, and confirm the result in the usage data.**

Telling the model to "answer in three lines" doesn't necessarily cut internal reasoning by the same amount. Adjust reasoning effort as described in Principle 7, and confirm it in the usage data. OpenAI bills reasoning tokens as output, and if the output limit is reached, the response may end before any visible answer appears. [OpenAI docs: Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)

### 5-4. For small changes, edit only what's needed

**Make small fixes with partial-edit tools, and rerun the generator for auto-generated files.**

Don't regenerate a huge file for a one-line fix; use a partial-edit tool. If it's a typo you can reliably fix yourself, fixing it by hand is also an option.

For auto-generated files, though, running the generator may be more accurate. Don't assume a fixed multiplier such as "partial edits are always an order of magnitude more efficient"; compare how much the model would have to generate with the verification required.

**Applicability and failure conditions:** Redundant explanations can go, but keep the verification results, open issues, and evidence needed to decide whether to accept the work. If truncation from limits increases, review the style instructions and the generation limit separately.

## Principle 6: Don't break the prompt cache

Prompt caching reuses past computation for the reusable leading portion (prefix) of the input. It doesn't return a previous response as is. Discount rates, write fees, minimum lengths, retention periods, and supported APIs differ by model and provider.

### 6-1. Keep the reused prefix stable

**Put shared instructions, tool definitions, and fixed material first, and information that changes every time, such as the time or the current request, after them.**

Place shared instructions, tools, and fixed material at the front, and information that changes only for this request at the back. If the time is needed, keep the correct time, but don't embed a value that changes every second at the top of the fixed instructions.

```text
Front: stable instructions, tool information, shared material
Back:  the current request, retrieved results, the current time, and other variable information
```

Keep the serialization order of JSON and similar data stable too. A change at one position can lose reuse of the identical prefix from that point on, but it doesn't necessarily wipe out the matching part before it.

### 6-2. Default to appending, but prune unneeded history deliberately

**While work continues, append to the history instead of rewriting it, and compact or switch only when the input saved over the remaining work outweighs the cost of rebuilding the cache.**

While work continues, don't carelessly rewrite past messages; append new information instead. That said, don't keep unneeded history forever just to protect the cache. Even if compaction or a switch reduces what's cached, it's worth adopting if the total cost from then on goes down.

Conceptually, compare the following.

```text
cost of generating the compaction + cost of rereading and rework + extra cost of rebuilding the cache
    < input cost saved over the remaining work
```

This is a formula for making the decision; you also need to confirm that quality stays the same. Manus's design of a stable prefix and append-centric history is operational know-how aimed at exactly this kind of reuse. [Manus's cache design](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

### 6-3. Weigh changing tool definitions against deferred loading

**With a few tools, fix their definitions and order; with many, use tool search or deferred loading. Don't change definitions mid-session.**

Changing tool names, schemas, or order affects cache matches for that part of the input. On the other hand, removing definitions you don't use has benefits of its own.

With a few tools, stable definitions are the candidate; with many, tool search or deferred loading. Don't break the pairing between tool calls and their results in the history, and use mechanisms the API you're on actually supports. Manus's logit masking is know-how from its own implementation, and not every API offers the same control.

### 6-4. Use each provider's and model's own specs

**Check the minimum cacheable length, retention period, write multiplier, and read multiplier for your model in the official pricing tables, and don't reuse values from other models.**

| API / model family | Caching method | Example pricing and conditions |
| --- | --- | --- |
| OpenAI GPT-5.6 and later | Supports automatic caching and explicit breakpoints | Writes with the default 30-minute retention cost 1.25× regular input. Reads cost 0.1× on most models and 0.05× on GPT-6.1 Sol. 12-hour retention in the Responses API costs 2.5× for writes |
| Earlier OpenAI models | Automatic. Retention, minimum length, and so on vary by model | Don't carry over the settings or write fees of the newer model family |
| Anthropic Messages API | Automatic caching via `cache_control`, or explicit breakpoints | 5-minute and 1-hour retention, a write premium, and a read discount. Multipliers and minimum lengths vary by model |
| Gemini API | Implicit caching on supported models. Older models and APIs also have an explicit method | The Interactions API, and Gemini 4 Argon and later, don't support explicit caching. In multi-turn use on Gemini 4 and later, if fewer than 2,050 new tokens are added, the last 2,050 tokens of the previous request are reprocessed as regular input. Also check storage fees for the older method |

[OpenAI docs: Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching), [Anthropic docs: Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching), [Google docs: Context caching](https://ai.google.dev/gemini-api/docs/caching)

For OpenAI GPT-5.6 and later, the minimum cacheable length is 1,024 tokens of visible input. Whether padding short input up to the minimum pays off depends on how many times it's reused and on the read and write prices. Don't make padding a standard tactic; decide based on the cost of the entire input. Anthropic's 20-block lookback is how far back from each breakpoint it searches for an earlier match; it doesn't mean "only 20 blocks can be stored." With growing history, check where the breakpoints are and where the previous write happened.

Writes and reads are priced separately. For example, if regular input costs 1, writes 1.25, and reads 0.1, writing the same prefix once and reading it once costs 1.35, versus 2 if it's sent as regular input both times. Content used only once gains nothing from reuse. This calculation covers only the prefix and excludes the rest of the input and the output.

### 6-5. Look at total cost at constant quality, not hit rate

**Judge caching by the total cost of writes, reads, regular input, and output together with the pass rate, and use the hit rate only for diagnosis.**

The hit rate is useful for diagnosis, but it can go up even when you're piling on unneeded history. Look at cache writes, reads, regular input, output, and the number of calls together.

PwC's preprint evaluation covers DeepResearchBench, a 10,000-token system prompt, four models from three providers, and more than 500 sessions. Under the conditions listed in Table 1, cost savings range from 41.4% to 79.6%, and TTFT improvements from 6.1% to 30.9%. These partly differ from the ranges given in the abstract, so this guide uses the figures in the table. The no-cache baseline deliberately breaks matches with a UUID, so this doesn't mean you'll get the same improvement over the automatic caching of normal operation. [PwC paper, Table 1 / Methodology](https://arxiv.org/html/2601.06007v1)

Deriv's 85.8% hit rate and 77% reduction in input costs are likewise results under its own pricing and input mix. Don't read input-cost savings directly as savings on the total task cost. [Deriv's production measurements](https://derivai.substack.com/p/prompt-caching-production-ai-agent-costs)

**Applicability and failure conditions:** You can expect benefits when there is long content to reuse. Include in the total the cost of writing content that isn't reused within the retention period, unneeded context that only raises the hit rate, and cache rebuilds caused by switching models. You can't necessarily change a CLI product's internal settings the way an API user can.

## Principle 7: Match the model and reasoning effort to the task

Choose based on the total cost of a configuration that meets the required quality, not on a low unit price. Sometimes a more capable model finishes in fewer round trips; sometimes a lighter model is enough.

| Nature of the work | What to try first | When to step up |
| --- | --- | --- |
| Clear-cut classification, extraction, routine edits | Small, fast models with lower reasoning effort | Requirements keep being missed or verification keeps failing |
| Multi-file implementation, complex bugs | Mid- to high-capability models with moderate reasoning effort | Hypotheses about the cause stall, or consistency problems remain |
| Design decisions, difficult reviews | Compare options, including high-capability models | Judge by whether you get a verifiable conclusion |
| Subagents | Choose according to the difficulty of the assignment | Don't settle on "it's a subagent, so it gets a small model" |

Document editing also demands high accuracy when it involves contracts or complex technical specifications. Even a single file isn't necessarily an easy job if it involves difficult algorithms or concurrency.

### 7-1. Compare delegation with adjusting a single model's reasoning effort

**Don't delegate short work that isn't independent; try lowering a single model's reasoning effort instead. Delegate when the conditions in section 2-2 are met.**

Having the parent handle planning and integration while handing clearly bounded work to children is one option. For short work that isn't independent, lowering a single model's reasoning effort may be cheaper than setting up delegation. Check the delegation conditions in section 2-2 first.

You can define roles with a subagent's `model` in Claude Code, settings in `.codex/agents/` in Codex, and `.cursor/agents/` and the like in Cursor. What gets inherited, such as permissions, tools, instructions, and default models, differs between products. Don't compare by changing only the model name; record reasoning effort too. [Claude Code docs: Subagents](https://code.claude.com/docs/en/sub-agents), [Codex docs: Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents), [Cursor docs: Subagents](https://cursor.com/docs/subagents)

### 7-2. Choose profiles at task boundaries

**Prepare a few profiles that bundle model, reasoning effort, and output style, and choose one at the start of each task. Decide in advance the criteria for stepping up.**

Prepare setups such as "routine edits," "regular implementation," and "hard problems," and choose model, reasoning effort, and output style together. Change them at task boundaries, and record the cache rebuilds and handoffs that come with switching midway.

When a small model fails, don't keep at it indefinitely; decide on criteria for switching to a stronger model, such as the number of verification failures or whether new evidence is emerging. Include the cost of the earlier failures in the total.

See “A practical Codex CLI workflow” below for the profile format and comparison procedure.

In Devin Cloud, use the agent selector to switch among options such as Lite, Normal, Ultra, and Fusion. Availability depends on admin settings and other conditions. Match routine work and hard problems to appropriate options, and compare usage and pass rates including failures and switches. The SWE-2 research preview offers Medium/High/Max reasoning effort; don't assume other modes expose the same setting. [Devin docs: Choosing an agent](https://docs.devin.ai/get-started/first-run#choosing-an-agent)

### 7-3. Compare open-weight models on total operating cost

**Before adopting one, compare on the total cost, including success rate, latency, and operating costs on representative tasks, rather than on unit price.**

Open-weight models, whether through hosting services or self-hosted, are also options. Even if the unit price drops, compare success rate, long-context handling, tool calling, latency, and operating costs separately.

The "80% or more cheaper" figure is Atlas Cloud's claim based on comparing specific models and prices; don't use it as a general guarantee of savings. If you self-host, include GPUs, idle time, maintenance, and so on. Start by trying representative tasks, then expand to an evaluation set that includes harder cases and failure examples. [Atlas Cloud](https://www.atlascloud.ai/blog/tips/reduce-ai-coding-token-cost)

### 7-4. Tune reasoning effort based on results

**Start with lower reasoning effort and raise it when verification fails or root-cause investigation stalls. Record the setting, the usage, and the pass rate together.**

Start low for simple typo fixes, and raise it as needed for difficult root-cause investigations. Make neither "always max" nor "always min" the default. More reasoning raises the cost of each call but can reduce retries.

`reasoning.effort`, `model_reasoning_effort`, and Anthropic's effort/thinking settings differ in both name and allowed values. Some models don't let you turn thinking off. Check the setting, the actual usage, and the pass rate together. [OpenAI docs: Reasoning models](https://developers.openai.com/api/docs/guides/reasoning), [Codex docs: Configuration Reference](https://developers.openai.com/codex/config-reference), [Anthropic docs: Effort](https://platform.claude.com/docs/en/build-with-claude/effort)

**Applicability and failure conditions:** If switching to a smaller model increases rereads of the same information or work being sent back, reevaluate. Verification that misses hidden failures can only measure apparent savings. Fix the acceptance criteria before comparing.

## Principle 8: Measure before you optimize

Record consumption, billing, quality, and time separately, and compare using the same task definitions. The starting point is keeping failed attempts in the totals.

### 8-1. Make total cost per successful task the primary metric

**Use total cost divided by the number of passed tasks as the primary metric, counting failed attempts, child agents, and sent-back work in the cost. Treat the other metrics as its breakdown.**

| Metric | What it shows | Caveats |
| --- | --- | --- |
| Total tokens | Consumption of input, output, cache, and reasoning | A simple sum across models is not a cost |
| Actual API cost and additional payments | Unit prices, caching, service and execution environment fees | Distinguish estimates from billed amounts |
| Task success rate | Share that passed fixed acceptance criteria | Distinguish tests not run, partial completion, and missed items |
| Total cost per passed task | Cost of all attempts divided by the number passed | Include failures, children, retries, and sent-back work |
| Time to completion | Time until the user gets a result | Reporting both the median and p95 makes slow outliers visible |
| Rework rate and review time | Quality loss that comes with cutting costs | Also track problems found after acceptance |
| Context usage | How much of the window the input occupies | Not the same as the share of information that actually contributed to decisions |
| Cache read rate | Share of input being reused | Look at the total, including write fees and unneeded history |
| Breakdown of calls and tool output | Heavy steps, noise, rereads | Distinguish bytes, characters, and tokens |
| Consumption by agent | Parent-and-child totals and outliers | A smaller parent conversation doesn't necessarily mean an overall improvement |

"Context utilization" is easily confused because it has two meanings. You can get how much of the window is occupied, but usage data alone can't tell you which information actually contributed to the model's decisions. LangWatch's figures for the latter are also estimates based on judgments, not direct measurements of the internal attention mechanism.

### 8-2. Start with built-in displays and add only the granularity you need

**First look at the breakdown in the product's usage display; for weekly reports and custom agents, normalize usage into a common format and record it.**

Start with a rough view of consumption from Claude Code's `/usage` and `/context`, its cost display and status line, Codex's usage display and logs, Cursor's Usage page, and so on. Check the available fields and commands in the version you use. Commands such as `/cost` don't necessarily work the same way in every environment. [Claude Code docs: Costs](https://code.claude.com/docs/en/costs), [Codex docs: Developer commands](https://developers.openai.com/codex/cli/slash-commands), [Cursor docs: Models & Pricing](https://cursor.com/docs/models-and-pricing)

In Devin, use Session Insights for per-session usage and the issue timeline. For account-wide usage, use Usage & limits on Self-serve or Consumption/Consumption Analytics on Enterprise, distinguishing quota and credits from ACUs. Relate environment problems and retries to verification results, and include parent/child work and review rework in comparisons. Insights analysis is a diagnostic aid, not a substitute for a controlled savings comparison or internal token breakdowns. [Devin docs: Usage](https://docs.devin.ai/admin/billing/usage), [Session Insights](https://docs.devin.ai/product-guides/session-insights)

For weekly reports, local log aggregators such as ccusage are an option; for detailed analysis, OpenTelemetry or a tracing platform. Check which products and versions an aggregator supports, and how it counts cache usage and subagents. Use RTK's `rtk gain` to diagnose output reduction, not as a substitute for your bill.

In custom agents, normalize each API's usage into a common format.

| API | How total input is represented | Main caveats |
| --- | --- | --- |
| OpenAI Responses | `input_tokens` includes reads and writes. Get the breakdown from `input_tokens_details.cached_tokens` and, on supported models, `input_tokens_details.cache_write_tokens` | Don't add the breakdown back into the total. Where supported, the breakdown by retention period is in `input_tokens_details.cache_write_tokens_details` |
| Anthropic Messages | `input_tokens + cache_creation_input_tokens + cache_read_input_tokens` | `input_tokens` alone is not the total input. Price 5-minute and 1-hour writes separately |

[OpenAI docs: Prompt caching usage](https://developers.openai.com/api/docs/guides/prompt-caching), [Anthropic docs: Understanding the token breakdown](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

Store fields that an API or SDK doesn't return as unknown. Don't turn missing values into zero; check them against the provider's pricing and usage specs. Treat absence as zero only when the API spec defines it that way. Also check whether reasoning is counted within output.

Match the denominator of the cache read rate to the API as well. With the response formats above, it's `cached_tokens / input_tokens` for OpenAI and `cache_read_input_tokens / (input_tokens + cache_creation_input_tokens + cache_read_input_tokens)` for Anthropic. Across multiple requests, sum the numerators and the denominators separately before dividing, and handle zero-input cases separately.

```text
Minimum fields to record
task_id / attempt_id / parent_task_id
actual model ID / reasoning settings / agent and config version
regular input / cache writes (by retention period) / cache reads / output / reasoning breakdown
model calls / tool calls / elapsed time / cost (measured or estimated)
version under verification / verification result / success, failure, or incomplete / retries and sent-back work
```

In Codex CLI, use `/status` for remaining allowances and the [usage dashboard](https://chatgpt.com/codex/settings/usage) for consumption trends. Context occupancy measures the current input size; cumulative tokens measure processing so far; remaining allowance measures available plan capacity. Don't convert these directly into one another. When aggregating detailed logs, check whether event values are incremental or cumulative, include child and resumed sessions, and deduplicate events. If several conversations ran concurrently, don't attribute the account-wide allowance change to a single task. [Codex docs: Pricing](https://learn.chatgpt.com/docs/pricing)

### 8-3. Compare one change at a time under the same conditions

**Fix the task set and acceptance criteria, change just one thing, and decide whether to adopt it based on total cost including failures, pass rate, and completion time.**

1. Fix a representative task set and acceptance criteria. Include hard problems and failure-prone cases, not just easy successes.
2. Record the baseline configuration's total cost, pass rate, and completion time. If random variation is large, repeat the runs.
3. Change just one thing, such as an output wrapper, search method, or reasoning effort.
4. Run the same verification and compare totals that include failures. Align the run order, cache warmth, and model version, or record them.
5. Adopt the change if it stays within acceptable quality and time and improves the cost per passed task.

For example, if output is 50% shorter but the pass rate drops from 95% to 75%, don't adopt the change on the strength of the output reduction alone. Set numeric targets per use case. Don't set a one-size-fits-all bar such as a cache hit rate of 70% or more.

**Applicability and failure conditions:** If task difficulty or the definition of success changes between before and after, the comparison doesn't hold. First check for deleted attempts, parent-only totals, double-counted reasoning, and confusion between token estimates and billed amounts.

## Comparison table by agent

This table compares where each principle is implemented. It was checked against the main official documentation on October 25, 2026, but what's available varies by version and execution environment. "Not confirmed" doesn't mean a feature doesn't exist. Before copying a specific setting, check the official sources listed below the table.

| Principle / use | Claude Code | Codex CLI | Cursor | Devin Cloud | Custom agents (direct API) |
| --- | --- | --- | --- | --- | --- |
| What to read (1) | Specify targets; `Read(...)` in `permissions.deny`. `.claudeignore` has no effect | Specify targets; AGENTS.md. Configure access separately, for example with the sandbox | `.cursorignore`, explicit file references. Distinct from restrictions on the terminal and MCP | Specify repositories and paths; distinct from access permissions | Design the tools' search scope, the information they return, and execution permissions |
| Lean instructions (1) | CLAUDE.md, conditional rules, on-demand skills. `@path` is loaded automatically | Account for AGENTS.md hierarchy and loading limits | AGENTS.md, application conditions of `.mdc` files in `.cursor/rules/`, skills | AGENTS.md (first 16 KiB per file auto-loaded), Skills | Separate fixed instructions from task-specific information |
| Code map (1) | LSP plugins, MCP, etc. | Search, plus MCP etc. as needed | Instant Grep, Explore, MCP as needed | Ask Devin code search, repository indexing | Choose AST, LSP, or semantic search to suit the purpose |
| Tool output (1) | Filtering on the command side, wrappers, output limits. Distinguish characters from tokens | Filtering on the command side, wrappers, instructions, hooks | Shape output on the command side. For MCP, replacing results with the corresponding hook is also an option | Filter on the command side; preserve originals and exit status | Save originals, extract key points, preserve exit status |
| Session splitting (2) | New conversation, `/clear`. Resume with `/resume`. Per thread in Projects | New session. Distinguish resuming from splitting | New chat; `/clear` in the CLI. Per worker in Projects, where the coordinator summarizes automatically | New session; distinguish snapshots from working state | Save state and carry only what's needed into a new conversation |
| Subagents (2) | Markdown files etc. in `.claude/agents/`. Design model, tools, and lifetime | TOML in `.codex/agents/`. Permissions, provider, and MCP are inherited from the parent | Markdown files etc. in `.cursor/agents/`. Each child has its own context | Managed Devins; assign scope, deliverables, and limits to children on separate VMs | Implement each child's input, limits, deliverables, and integration |
| External memory (2) | Progress files, memory. In Projects, project memory and project instructions | Progress files, supported memory features | Progress files, rules, etc. In Projects, the synced shared-context files | Progress files, personal Memory; hand off artifacts across VMs | Files or a DB, with references to originals and versions |
| Compaction (2) | `/compact`, auto-compaction. Specify what to keep | `/compact`, built-in compaction | Automatic summarization; `/summarize` in the CLI | A common manual-compaction setting was not confirmed in this review; save handoff state | A combination of summaries, recent history, and re-fetching |
| Offloading deterministic work (3) | Formatting and lint via `PostToolUse` hooks. pre-commit and CI are common to all | `PostToolUse` hooks. pre-commit and CI are common to all | `afterFileEdit` hook, format on save. pre-commit and CI are common to all | Formatting and lint via the shell; pre-commit and CI are common to all | Run formatting and mechanical fixes outside model calls |
| Shared procedures (3) | Skills (`.claude/skills/` etc.), on-demand references from instruction files | Skills (`.codex/skills/` etc.), on-demand references from AGENTS.md | Skills (`.agents/skills/`, `.cursor/skills/`). Restrict target files with `paths` | Skills (`.agents/skills/` etc.); Blueprint/Snapshot for environment setup | Turn procedures into templates and load them only when needed |
| Tool consolidation (3) | Shell scripts and available code execution | Shell scripts and available execution features | Shell scripts, etc. | Shell scripts on the VM | Composite tools, code execution, Programmatic Tool Calling on supported APIs |
| Planning (4) | plan mode | `/plan` | Plan Mode, `/plan` in the CLI | Ask for investigation and planning; Agent for execution | Separate planning from execution where needed |
| Staged verification (4) | Lightweight diagnostics via `PostToolUse` etc. Run heavy checks per unit of work | `PostToolUse` etc. Check the tools used and the return format | `postToolUse` etc. Return via `additional_context` and similar | Specify check scope and commands in requests or Skills | Decide the scope from the diff, dependencies, and changes since the last verification |
| Pre-completion verification (4) | Continuation control in `Stop` + CI | `decision: "block"` etc. in `Stop` + CI | `followup_message` in `stop` + CI | Specify completion criteria and verification procedures + CI; distinguish unrun checks | Check acceptance criteria. Cap the number of continuations too |
| Execution limits (4) | `--max-turns` and `--max-budget-usd` in print mode, `maxTurns` for subagents | `agents.max_concurrent_threads_per_session` limits concurrent child threads. A general setting that guarantees an overall budget was not confirmed in this review. Also use external monitoring | `loop_limit` caps automatic continuation by Stop-type hooks. Separate from the overall budget | Devin MCP supports an ACU limit at session creation; manage parent/child scope separately | Limit time, number of model calls, total cost, and repeated failures |
| Output style (5) | Instructions and `CLAUDE_CODE_MAX_OUTPUT_TOKENS` | AGENTS.md, `model_verbosity` on supported models | Rules, the request itself | Requests, AGENTS.md; distinct from API output caps | Per-API output limits and style. Handle early termination too |
| Caching (6) | Controlled by the product. Users measure the impact of unnecessary changes and restarts | Controlled by the product. You can't necessarily set arbitrary API parameters directly | Depends on the model in use and the product's own controls | Controlled by the product; don't apply API cache settings directly | Configure per the model- and API-specific specs of OpenAI, Anthropic, and Google |
| Model switching (7) | `/model`, `model` for subagents | Model selection, `--profile`, `model` for subagents | Model selection or Auto for the parent; `model` etc. for children. Plan or admin restrictions may mean a substitute model is used | Agent selector (Lite/Normal/Ultra/Fusion etc.); subject to admin settings | Route per request or per task |
| Reasoning effort (7) | Supported effort/thinking settings | `model_reasoning_effort` etc. | Follow the settings the selected model offers | Medium/High/Max in the SWE-2 research preview; don't generalize to other modes | Per-model reasoning/thinking settings |
| Measurement (8) | `/usage`, `/context`, logs, traces | `/status`, usage dashboard, and logs; distinguish allowances from cumulative tokens | Usage page, dashboards depending on your plan | Session Insights, plan-specific usage displays | Link usage, cost, success rate, and completion time |

For Claude Code, see [Memory](https://code.claude.com/docs/en/memory), [Permissions](https://code.claude.com/docs/en/permissions), [Hooks](https://code.claude.com/docs/en/hooks), [CLI reference](https://code.claude.com/docs/en/cli-reference), and [Subagents](https://code.claude.com/docs/en/sub-agents). For Codex, the corresponding pages are [AGENTS.md](https://developers.openai.com/codex/guides/agents-md), [Configuration Reference](https://developers.openai.com/codex/config-reference), [Hooks](https://developers.openai.com/codex/hooks), [Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents), and [Developer commands](https://developers.openai.com/codex/cli/slash-commands). For Cursor, see [Rules](https://cursor.com/docs/rules), [Ignore file](https://cursor.com/docs/reference/ignore-file), [Hooks](https://cursor.com/docs/hooks), [Subagents](https://cursor.com/docs/subagents), and [CLI Slash commands](https://cursor.com/docs/cli/reference/slash-commands).

For Devin Cloud, see [AGENTS.md](https://docs.devin.ai/onboard-devin/agents-md), [Skills](https://docs.devin.ai/product-guides/skills), [Memory](https://docs.devin.ai/product-guides/memory), [Environment](https://docs.devin.ai/onboard-devin/environment), [Managed Devins](https://docs.devin.ai/work-with-devin/advanced-capabilities#managed-devins), [Devin MCP](https://docs.devin.ai/work-with-devin/devin-mcp), [Your First Session](https://docs.devin.ai/get-started/first-run), and [Usage](https://docs.devin.ai/admin/billing/usage). This column covers Cloud, not the settings of Devin CLI or Desktop.

The eight principles also apply to products not in the table, such as Gemini CLI, but don't borrow setting names from other products. Also distinguish among three things: execution environments such as Cursor's Cloud Agents, subagents within a conversation, and the coordinator in Projects. The coordinator doesn't write code itself; it launches workers in an execution environment and receives only their results. [Cursor docs: Cloud Agents](https://cursor.com/docs/cloud-agent), [Cursor docs: Projects](https://cursor.com/docs/agent/projects)

### Don't confuse product usage allowances with API billing

For Codex, distinguish the included allowance and additional credits used with ChatGPT sign-in from API billing with an API key. **Codex credit billing has no separate cache-write charge.** Don't apply the OpenAI API cache-write rates in principle 6 to it: use API pricing for API-key usage and Codex rates for credit usage. Credit rates alone don't determine consumption of the included subscription allowance; check the product's actual remaining-limit display. [Codex docs: Pricing](https://learn.chatgpt.com/docs/pricing)

In Cursor, usage allowances and extra charges vary by plan. As of this review, The X Subscription includes a weekly shared allowance for Cursor models and credits for third-party models and the like, while existing legacy plans work differently. Don't convert estimated token costs directly into what you pay or how much allowance you have left. [Cursor docs: Models & Pricing](https://cursor.com/docs/models-and-pricing)

Whether you can use Projects (section 2-1) depends on your plan. As of this review, Cursor offers it on Premium and above, Teams, Enterprise, and legacy paid plans, but not on Free, Plus, or legacy Hobby, or with Privacy Mode (Legacy). Claude Code offers it as a public beta on Pro and Max; it isn't yet available on Team or Enterprise. Project usage comes out of the same allowance as regular sessions. In Claude Code, threads in a new Project run on Opus at high effort by default, and the official docs note that this uses up your allowance faster than a single session. You can change a thread's model and effort in Project settings. A thread that hits its limit waits for the next allowance reset and then continues automatically. The coordinator keeps the conversation going with automatic summarization in Cursor, and with recent messages, thread reports, and project memory in Claude Code. For handoffs, Claude Code uses project memory and project instructions, and Cursor uses the shared-context files that the Project syncs. [Cursor docs: Projects](https://cursor.com/docs/agent/projects), [Claude Code docs: Projects](https://code.claude.com/docs/en/claude-projects)

Devin's **Self-serve plans (Free, Pro, Max, Teams) use usage allowances plus on-demand credits, and Enterprise uses ACUs at contracted rates**. Consumption is based on factors such as the number and complexity of operations and VM time; it isn't simply billed by elapsed time. The docs explain that waiting for tests or responses isn't added to usage, apart from things like keeping the VM running. Cut unnecessary exploration, operations, and trial and error, and compare using actual usage. [Devin docs: Billing](https://docs.devin.ai/admin/billing), [Usage](https://docs.devin.ai/admin/billing/usage)

## A practical Codex CLI workflow

**Check authentication and usage, change only reasoning effort, and compare against the same completion criteria.**

This configuration example follows the official specifications checked on October 25, 2026. The profile format targets Codex CLI 0.134.0 and later. It is not a report of execution tests on a specific version or measured savings. Don't assume the same settings apply to desktop or cloud execution.

1. Record the CLI version and authentication mode with `codex --version` and `codex login status`. Decide whether you are evaluating the ChatGPT allowance, credits, or API charges, then collect a baseline using the displays and logs in section 8-2.
2. Hold the model, starting directory, target code, prompt, and completion criteria constant. Keep the AGENTS.md discovery scope and other settings consistent too.
3. Create the following two files and compare them in separate trials. Merge the relevant settings if files already exist. The model is inherited from the shared configuration, so verify that the actual model ID is the same and supports both `medium` and `high`.

```toml
# ~/.codex/standard.config.toml
model_reasoning_effort = "medium"
```

```toml
# ~/.codex/investigate.config.toml
model_reasoning_effort = "high"
```

```bash
codex --profile standard
# Start a separate comparison trial
codex --profile investigate
```

From 0.134.0 onward, use separate `~/.codex/name.config.toml` files, not `[profiles.name]` tables in `config.toml`. Project settings and command-line overrides can take precedence, so check the effective model and effort after starting. For supported models, `model_verbosity = "low"` is an option for shorter answers, but evaluate it separately from reasoning effort. [Codex docs: Profiles](https://learn.chatgpt.com/docs/config-file/config-advanced#profiles), [Configuration Reference](https://learn.chatgpt.com/docs/config-file/config-reference)

4. Record total consumption including failures, retries, and children, along with pass rate and completion time. Try higher effort if lower effort stalls on causal hypotheses or repeatedly misses requirements that verification catches. Permission failures and external outages need separate remedies, not higher reasoning effort.
5. Continue the conversation for follow-up work on the same fix; start fresh for independent tasks. Before compacting a long single task, save exact errors, changes, verification results, and the next action. Record compaction and cache conditions during comparison trials too.

This example starts with reasoning effort. Don't shorten instructions, trim tool output, and switch models at the same time; use principle 8 to evaluate one change at a time.

## Adoption priority checklist

First get the breakdown, then try one measure at a time, starting where costs are highest. The items below are candidates ordered by how easy they are to adopt; you aren't expected to adopt all of them.

### What you can do today (no configuration changes)

- [ ] Define success, and take baseline measurements of total tokens, total cost, pass rate, and completion time.
- [ ] Include the target files and functions, reproduction conditions, constraints, and verification method in requests.
- [ ] Move unrelated tasks to a separate session, and carry over only the necessary state.
- [ ] Shorten lists of passing tests and successful build output, while keeping failure diagnostics, the path to the full log, and the exit status.
- [ ] Choose exploration that fits the purpose: a partial Read for known files, search for strings, LSP for references, and so on.
- [ ] Don't repeat the full text or diff of directly edited files in every final response.
- [ ] Confirm a plan first only for ambiguous or high-impact changes, and go straight ahead with small fixes.

### Set once, and revisit when models are updated

- [ ] Check how much of your instruction files actually gets loaded, and move unnecessary always-on instructions into conditional or on-demand references.
- [ ] Audit fixed procedures written for older models and duplicated verification instructions.
- [ ] If you use `.claudeignore`, review its targets and purpose, and move them to a `Read` deny if needed. Also check that `.cursorignore` doesn't block files that some tasks need to read.
- [ ] Narrow routine exploration of generated artifacts and the like. Keep lockfiles accessible for tasks that need them.
- [ ] Clean up unused MCP servers and plugins, taking deferred loading into account.
- [ ] Decide what progress files and compaction should preserve: the goal, constraints, verification results, exact errors, and next action.
- [ ] Move fixes with a single deterministic outcome, such as formatting, into on-save actions, pre-commit, and CI.
- [ ] Run lightweight diagnostics right after edits, related tests per unit of work, and the necessary full verification before completion.
- [ ] Prevent Stop-type hooks from continuing endlessly, and verification from repeating when nothing has changed.
- [ ] Set limits on time, calls, and budget that cover non-interactive runs and subagent costs.

### Decide on tools by comparing them

- [ ] Try one output wrapper; check whether originals, exit codes, or diagnostics get lost, and compare total task cost.
- [ ] Measure whether LSP, code graphs, or semantic search reduce re-exploration compared with ordinary search.
- [ ] Try moving bulk fetching and aggregation into a program so intermediate results never reach the model.
- [ ] Compare tool consolidation including the cost of generating and running it, timeouts, and handling of partial failures.

### Decide through design and operations

- [ ] Decide whether to use subagents based on independence, how much information can be isolated, and the cost of startup and integration.
- [ ] Compare how much a fork inherits with how much re-exploration that inheritance avoids.
- [ ] When failures with the same cause keep happening, either update the hypothesis or hand the task off as incomplete.
- [ ] For long tasks, save state and compact deliberately rather than automatically waiting for the limit.
- [ ] Compare small models, adjusting a single model's reasoning effort, and delegation against the same acceptance criteria.
- [ ] Account for cache retention, write, and read pricing for each model and API.
- [ ] Decide on adoption by total cost per passed task, including parents and children, retries, and sent-back work.

### Practices to avoid

- Cutting the meaning of the spec or necessary error information, and treating short input as the only measure of success.
- Always delegating even simple searches, or always using a code graph.
- Running the full test suite after every edit, verifying unchanged targets over and over.
- Compacting extremely often, or avoiding compaction even when it's needed.
- Keeping unneeded history or long shared instructions just to raise the cache hit rate.
- Leaving the cost of a cheap model's failures, subagent consumption, or review time out of the totals.
- Treating token counts, estimated costs, remaining plan allowance, and actual payments as the same number.
- Introducing several cost-cutting measures at once, so you can't trace the cause of a quality drop.

## References

The main papers, the claims by LangWatch and Manus, and the key settings, billing details, and API specifications in the text were checked against primary sources. Sources and URLs are collected below. Practitioner and comparison articles are supplementary examples of adoption; their savings figures and settings are not adopted as general rules in this guide. For official URLs that have moved, the current pages referenced in the text are listed as well.

### Preprints and research (key experimental conditions verified)

- [Don't Break the Cache: An Evaluation of Prompt Caching for Long-Horizon Agentic Tasks](https://arxiv.org/abs/2601.06007)
- [Can your AI agent be cheaper? Investigating the effects of task specifications on token spend in agentic coding tasks](https://arxiv.org/abs/2608.25399)
- [alphaXiv page for the paper above](https://www.alphaxiv.org/abs/2608.25399) — supplementary link. Figures and conditions were checked against the original paper above.

### Company data (observations and operational insights)

- [The Context Tax: When to Compact Your Coding Agent](https://langwatch.ai/blog/context-tax-when-to-compact)
- [Context Engineering for AI Agents: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [How prompt caching cut our production AI agent input costs by 77%](https://derivai.substack.com/p/prompt-caching-production-ai-agent-costs)

### Practitioner reports and supplementary material

- [3 Simple Techniques to Reduce Token Consumption in Claude Code and Codex](https://tech.autoscout24.com/blog/posts/3-techniques-to-reduce-token-consumption-claude-code-codex/)
- [How We Cut AI Agent Costs 25% With Prompt Caching](https://draup.com/engineering/under-the-hood/draup-agent-costs)
- [11 ways to cut token usage in AI-assisted development](https://www.pluralsight.com/resources/blog/ai-and-data/cutting-ai-token-usage-development)
- [When our AI agents' monthly token usage passed several million](https://zenn.dev/dx_pm_product/articles/ai-token-cost-reduction-guide) (in Japanese)
- [How to save tokens in Codex](https://ai.giftx.co.jp/blog/codex-token-saving/) (in Japanese)
- [How to reduce token consumption in AI coding](https://note.com/sayu_creative/n/n7b35c6ec2de7) (in Japanese)
- [How to Cut Token Usage in AI Coding Agents](https://blog.openreplay.com/cut-token-usage-ai-coding-agents/)
- [23 Tips for Smart Claude Code Token Saving](https://www.analyticsvidhya.com/blog/2026/05/tips-for-claude-code-token-saving/)

### Official specifications and vendor material

- [Claude Code: Manage costs effectively](https://code.claude.com/docs/en/costs)
- Claude Code: [Hooks reference](https://code.claude.com/docs/en/hooks), [Hooks guide](https://code.claude.com/docs/en/hooks-guide), [Best practices](https://code.claude.com/docs/en/best-practices), [Permission modes](https://code.claude.com/docs/en/permission-modes), [CLI reference](https://code.claude.com/docs/en/cli-reference), [Subagents](https://code.claude.com/docs/en/sub-agents), [Agent SDK: agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop)
- [Codex: Hooks](https://developers.openai.com/codex/hooks), [Codex CLI: Slash commands](https://developers.openai.com/codex/cli/slash-commands)
- [Hooks](https://cursor.com/docs/hooks), [Plan Mode](https://cursor.com/docs/agent/plan-mode)
- [About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [How to Reduce AI Coding Token Cost: 7 Tactics That Actually Work in 2026](https://www.atlascloud.ai/blog/tips/reduce-ai-coding-token-cost)
- [Models & Pricing](https://cursor.com/docs/models-and-pricing)
- [Subagents](https://cursor.com/docs/subagents), [Cloud Agents](https://cursor.com/docs/cloud-agent), [CLI Slash commands](https://cursor.com/docs/cli/reference/slash-commands)

### Primary sources for API specs, configuration, and billing

- [OpenAI: Prompt caching](https://developers.openai.com/api/docs/guides/prompt-caching) — pricing by model generation, explicit breakpoints, usage.
- [OpenAI: Conversation state](https://developers.openai.com/api/docs/guides/conversation-state) — the difference between keeping state and being billed for input.
- [OpenAI: Reasoning models](https://developers.openai.com/api/docs/guides/reasoning) — reasoning billing, output limits, incomplete responses.
- [Codex: Configuration Reference](https://developers.openai.com/codex/config-reference), [AGENTS.md](https://developers.openai.com/codex/guides/agents-md), [Subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents) — configuration and inheritance. As of the review date, Codex pages at `developers.openai.com/codex/*` redirect to `learn.chatgpt.com/docs/*`. The same applies to the other Codex links in this guide.
- [Claude Code: Permissions](https://code.claude.com/docs/en/permissions), [Memory](https://code.claude.com/docs/en/memory), [Environment variables](https://code.claude.com/docs/en/env-vars) — the `.claudeignore` correction, instruction loading, units of output limits.
- [Anthropic: Programmatic tool calling](https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling) — tool consolidation, and the evaluation conditions behind the 24% reduction and the roughly 8% increase.
- [Anthropic: Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching), [Effort](https://platform.claude.com/docs/en/build-with-claude/effort) — API-specific behavior and example responses for usage and stop_reason.
- [Anthropic: Optimizing for cost and intelligence](https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence) — auditing prompts against current models.
- [Google: Context caching](https://ai.google.dev/gemini-api/docs/caching) — implicit and explicit methods, and support by model and API.
- [Cursor: Ignore file](https://cursor.com/docs/reference/ignore-file), [Rules](https://cursor.com/docs/rules) — the scope of exclusions and conditional instructions.
- [Devin: Billing](https://docs.devin.ai/admin/billing), [Usage](https://docs.devin.ai/admin/billing/usage) — Self-serve vs. Enterprise, and how operations and wait time are counted.
- [Devin: AGENTS.md](https://docs.devin.ai/onboard-devin/agents-md), [Skills](https://docs.devin.ai/product-guides/skills), [Memory](https://docs.devin.ai/product-guides/memory) — instruction loading limits, shared procedures, and personal memory.
- [Devin: Environment](https://docs.devin.ai/onboard-devin/environment), [Blueprints](https://docs.devin.ai/onboard-devin/environment/blueprints), [Blueprint reference](https://docs.devin.ai/onboard-devin/environment/blueprint-reference) — environment reuse and session boundaries.
- [Devin: Your First Session](https://docs.devin.ai/get-started/first-run), [Managed Devins](https://docs.devin.ai/work-with-devin/advanced-capabilities#managed-devins), [Devin MCP](https://docs.devin.ai/work-with-devin/devin-mcp), [Session Insights](https://docs.devin.ai/product-guides/session-insights) — planning and agent selection, child sessions and limits, usage and issue analysis.
- [RTK: How RTK Savings Work](https://github.com/rtk-ai/rtk/blob/develop/docs/guide/resources/savings-explained.md) — the difference between output byte reduction and your bill, estimated token counts.
- [Claude Code: MCP](https://code.claude.com/docs/en/mcp) — when tool search and deferred loading apply, and exceptions.
- [Anthropic: Create a Message](https://platform.claude.com/docs/en/api/messages/create) — API reference for `max_tokens`, responses, and usage.
- [Cursor: Search](https://cursor.com/docs/agent/tools/search), [Prompting agents](https://cursor.com/docs/agent/prompting) — current search, automatic summarization, and context display.
