---
title: Agent-to-agent communication across machines, not just within one runner
state: OPEN
labels: [design, enhancement, multi-agent]
priority: medium
---

# Agent-to-agent communication across machines

**Status: rough idea.** Captured as stated, with a survey of what stands in the
way. No architecture is chosen here, and none should be until there is a
concrete use case to shape it — see "Decide before building".

## The idea

> Is there a way that Hugin agent sessions could communicate with each other —
> between computers, for example? So not sessions within the same runner, but
> between machines.

Today every form of coordination Hugin has assumes one process. Two agents can
talk only if both are live Python objects in the same `Session`.

## What exists today, and why it is process-local

Four mechanisms, all of which pass a live object reference:

| Mechanism | Where | What it passes |
|---|---|---|
| Direct messaging | `agent/agent.py:361-368` — `message_agent()` calls `stack.insert_external_input()`. Example: `examples/agent_messaging` | The caller must already hold the target `Agent` object |
| Sub-agents | `interaction/agent_call.py` creates the child via `parent_agent.session.create_agent_from_task(...)`; the child's `TaskResult.step()` resumes the parent by pushing onto `task_def.caller.stack` (`interaction/task_result.py:135-139`) | A live `caller` reference on both legs |
| Stepping | `Session.step()` (`agent/session.py:158-179`) iterates `self.agents` | A list of live agents in one interpreter |
| Shared state | `Environment.env_vars`, e.g. the documented `{"worlds": {"world_1": shared_world_object}}` (`CLAUDE.md`, "Shared State"); `examples/shared_state` | Arbitrary live Python objects — not serialisable in general |

So "cross-machine" is not a transport problem bolted onto an existing seam. Each
of these would need a remote answer, and the shared-state one may not have a
good answer at all in its current form.

## What is already networked (prior art to reuse, not rebuild)

- **Sandbox SSH backend** — `sandbox/ssh.py`, registered at `sandbox/sandbox.py:157`,
  alongside `docker.py` and the egress proxy in `egress_proxy.py`. Hugin can
  already run an agent's *commands* on another host. Note the boundary: this
  distributes **tool execution**, not agent sessions. The agent stays here.
- **Monitor server** — `cli/monitor_agents.py`, the only HTTP server in the
  package. It reads a storage directory; it does not coordinate anything.
- **Router correlation** — `llm/router_correlation.py` and `llm/router_outcome.py`
  already stamp a `session.id` across sub-agent LLM calls and report an
  edition's outcome to an external endpoint. Evidence that a session id is
  already a meaningful cross-process handle.

## The likely seam

`Storage` is an ABC (`storage/storage.py:21`) with `LocalStorage`
(`storage/local.py:52`) as its only implementation, persisting sessions, agents,
interactions and artifacts. If two machines shared a storage backend they would
already share the durable half of a session. That is worth investigating first,
because it is the one place the codebase is already abstract.

It is not sufficient on its own. Storage records what happened; it does not
step an agent, deliver a message to a *running* stack, or tell machine B that
machine A wants something. At minimum it leaves open: who steps which agent,
how a waiting agent learns it can resume, and what happens when two machines
write the same session.

## Decide before building

Following the precedent of tasks 029 and 032 — collect the evidence before
paying for the mechanism.

1. **What is the actual use case?** These want very different systems:
   - fan-out to more compute (agents that do not talk, only report results);
   - agents on machines with different *capabilities* (a GPU box, a machine with
     access to a private network or dataset);
   - long-lived agents on different machines genuinely conversing;
   - handing a session off between machines (start on a laptop, continue on a
     server).
   The first is largely task 004's problem. Only the third really needs this.
2. **Does it need to be live, or is durable hand-off enough?** A shared storage
   backend plus "whoever picks it up next steps it" is dramatically cheaper than
   a live message bus, and may cover most of the value.
3. **What is the trust boundary?** Cross-machine messaging means one machine can
   insert an `ExternalInput` into another machine's agent stack — that is prompt
   injection with a network interface. The sandbox work already established that
   the runtime is the containment boundary; this needs the equivalent answer.
4. **What breaks in `env_vars`?** Live-object shared state cannot cross a
   machine. Either it stays explicitly local, or shared state needs a
   serialisable subset with its own contract.

## Relationship to existing tasks

- **004 — Define portable local and cloud agent-run orchestration.** Adjacent and
  should be settled first. 004 is about running *a* run somewhere else, with a
  backend-neutral run contract, stable run ids and a storage capability
  statement. Much of what 004 must define (run id, storage that is not a local
  filesystem, resume policy) is a prerequisite here. This task is the different
  question of two runs *talking* while both are alive.
- **026 — Bash sandbox SSH/remote backend (merged).** Prior art for reaching
  another machine, at the tool layer.
- **006 — Support batched tool calls end-to-end.** Unrelated, but touches the
  same stepping loop; worth checking for interactions if both land.

## Tasks

- [ ] Write down the concrete use case that motivated this, and which of the
      four shapes above it is.
- [ ] Spike the cheap version: a shared `Storage` implementation, two machines,
      one session, hand-off rather than live messaging. Report what breaks.
- [ ] From that, decide whether live cross-machine messaging is needed at all.
- [ ] If it is: design the remote equivalents of `message_agent`, the
      `AgentCall`/`Waiting`/`TaskResult` resume path, and session stepping —
      including who owns stepping an agent, and what a partitioned network does
      to a waiting parent.
- [ ] State the trust boundary for a message arriving from another machine.

## Success Criteria

To be defined once the use case is. As a placeholder for the spike:

- [ ] A session's agents can be stepped from a second machine against shared
      storage, with a stated policy for concurrent writes.
- [ ] A written recommendation on whether live cross-machine messaging is worth
      building, with the evidence behind it.
