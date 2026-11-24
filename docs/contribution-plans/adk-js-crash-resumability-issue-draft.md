# Issue template — ADK JS crash resumability

状态：待用户手动提交。此前创建 Issue 的尝试被 GitHub 集成以 403 拒绝；尚无上游 Issue 编号。检索到相关 HITL 问题，但没有找到完全相同的普通崩溃恢复请求。建议使用 [Feature request](https://github.com/google/adk-js/issues/new?template=feature_request.md)。下面的英文标题和正文可直接复制；不包含私有仓库地址、用户名、真实业务数据。无需提出 PR 或承诺实现。

## Title

Workflow crash recovery: resume from persisted node outputs after unexpected process termination

## Issue body (copy below)

### Describe the feature

Please provide a supported way to resume an unfinished Workflow after unexpected process termination, using already persisted node outputs and routing decisions instead of executing completed nodes again.

With `@google/adk@2.5.0`, interrupt-based recovery using RequestInput works across processes in our tests, but an abrupt termination without an interrupt appears to be treated as the end of the previous run. A subsequent runAsync call on the same Session executes the completed nodes again.

### Use case

Long-running workflows may run LLM calls, local workers, or remote jobs for hours. A host can be killed, lose power, or crash without an opportunity to emit RequestInput. Session events are durable, yet restarting from the beginning can repeat expensive LLM calls and completed work. We would like to reuse the durable execution history.

### Environment

- Package: @google/adk@2.5.0
- Source commit (npm gitHead): b1b94c0d501c3d1eb028aa413d783d6573f6c785
- Node.js: 24.22.1
- Persistent DatabaseSessionService with SQLite, @mikro-orm/core and @mikro-orm/sqlite 7.2.11
- Workflow API, deterministic function nodes (no real LLM needed)

### Steps to reproduce

1. Define a Workflow with sequential function nodes A and B and a persistent SessionService. A appends a marker to a file and returns a result. B waits at a barrier.
2. Create a Session with a fixed session ID and run the Workflow. Wait until A's output has been persisted and yielded by Runner, before B completes.
3. Send SIGKILL to the host process.
4. In a fresh process, reopen the same database, construct the same Workflow and Runner, and call runAsync using the same session ID and original input.
5. Inspect the persisted events and execution markers.

### Observed behavior

A's completed output is present in the database, but A executes again. B then executes in the new invocation. In a parallel-branch variant, A completed and B was still running at the time of termination; both branches were executed again after restart.

A separate control test used RequestInput and a matching structured response in a new process, and successfully resumed the waiting Workflow. This suggests that the problem is specifically recovery without a recorded interrupt, rather than persistence in general.

The following are simplified traces of the observed node execution order (not raw console output):

```text
Sequential:
  before SIGKILL: A
  after restart:  A, B, deliver
Parallel:
  before SIGKILL: A, B(started)
  after restart:  A, B, deliver
```

### Relevant implementation

- [Runner.runAsync](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/src/runner/runner.ts#L300-L379) generates a fresh invocation ID for each call. Its public parameters do not include a previous invocation ID to resume.
- [eventsForCurrentRun](https://github.com/google/adk-js/blob/b1b94c0d501c3d1eb028aa413d783d6573f6c785/core/src/workflow/utils/rehydration_utils.ts#L38-L100) identifies paused historical invocations based on whether they raised interrupts. A historical invocation with node events but no interrupt becomes a run boundary; an abruptly terminated invocation can therefore be excluded from rehydration even if it has durable node outputs.

### Expected behavior

An explicit recovery API or documented recovery path that:

- distinguishes a completed run, an intentional wait, cancellation, and unexpected termination;
- reuses durably completed node outputs and routing decisions, including parallel and dynamic nodes where supported;
- continues unfinished work without replaying already committed LLM/node outputs;
- preserves the ability to start a genuinely new run in the same Session.

We understand that in-flight side effects cannot generally be recovered exactly once, and that external operations need application-level idempotency and reconciliation. We are not requesting arbitrary restoration of process memory or automatic replay of uncertain external side effects.

Is this already supported through a different public API in the TypeScript Workflow runtime, or is crash resumability planned?

---

## Internal notes — do not paste

- Prior test cases: R08 control; R10 sequential crash; R11 external side-effect window; R12 parallel crash. Actual temporary probe scripts were not retained in the repo; do not claim an attached runnable reproducer. If maintainers request one, reproduce on an upstream-only minimal test and attach real code/output.
- Related but distinct: [#733](https://github.com/google/adk-js/issues/733), [#734](https://github.com/google/adk-js/issues/734), [#730](https://github.com/google/adk-js/issues/730), [#715](https://github.com/google/adk-js/issues/715). Recheck open/closed Issue/PR search and actual upstream version before posting.
- Project decision: no local ADK recovery patch for first release; host restart reconciles and safely reruns with possible repeated LLM calls. Upstream issue is non-blocking. S3 contribution is separate.
