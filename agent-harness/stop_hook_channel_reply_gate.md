# Agent Harness — Stop-Hook Channel-Reply Gate

Operational pattern for agents that talk to a human over an **out-of-band channel** (Telegram, Slack,
SMS, email) whose messages egress through a *tool call*, not through the agent's ordinary text output.
This is the library's first *agent-harness* pattern — it governs the harness the agent runs in, not a
product UI.

## The failure it prevents

**Incident (Pantheon / head-orchestrator, 2026-08-18, tony):** The orchestrator answered the operator
twice by writing the reply as **plain assistant text** instead of calling the Telegram reply tool. Plain
text lands only in the session transcript; the operator saw silence while the agent believed it had
responded. The operator's words: *"you seem to think you have responded to me. you are wrong … this is
the second time … THIS IS UNTENABLE. THIS NEEDS TO BE REMEDIATED."*

**Ruling:** Remediate structurally, not by good intentions. Add a **`Stop` lifecycle hook** that refuses
to let the agent end its turn while the operator's most recent inbound message sits unanswered by an
actual send through the channel tool.

## Why the `Stop` lifecycle event is the right hook

The relevant lifecycle events an agent harness exposes, and why `Stop` is the one that fits:

- **`PreToolUse`** fires only when a tool *is* invoked — it cannot catch the bug, whose whole nature is
  that **no** send tool was invoked.
- **`PostToolUse`** likewise presupposes a tool ran.
- **`Stop`** fires exactly when the agent yields the turn back to the human. That is the one moment where
  "did a reply actually go out?" is both answerable and still actionable — the hook can **block** the
  stop and force the send before control returns.

A `Stop` hook is the harness's last checkpoint before silence. Put the delivery guarantee there.

## Generalized principle (harness-, channel-, and vendor-agnostic)

A `Stop` hook that enforces channel-reply delivery should have these properties:

1. **Anchor on the last inbound, not the last turn.** Scan the transcript for the most recent genuine
   *human inbound on the channel* (exclude tool-result records and the agent's own outbound echoes).
   Then ask: *was the channel send-tool invoked anywhere after that inbound?* Anchoring on "since the
   last inbound" — rather than "in the final turn" — means an early acknowledgement ("on it") satisfies
   the gate across later tool-working turns, while a **fresh** inbound re-arms it. A per-turn check would
   nag on every working turn of a long task.

2. **Block, don't warn.** On an unanswered inbound, emit the harness's *block* decision with a reason
   that names the mechanism ("plain text goes only to the transcript; call the send tool with the
   inbound's reply address"). A warning the model can ignore is how the bug recurs.

3. **FAIL-OPEN, always.** Any error — unreadable transcript, missing field, schema drift, unparseable
   line — must fall through to *allow the stop*. A delivery guard that can wedge the agent's ability to
   finish a turn is worse than the bug it prevents. Wrap everything; on exception, allow.

4. **Loop-safe.** Harnesses expose a "hook already fired this stop-cycle" signal (e.g.
   `stop_hook_active`). If it is set and a reply is *still* absent, **allow** rather than hard-loop — the
   block is a forcing reminder, not a cage. Correct delivery detection already makes the second stop pass
   (the reply now exists after the inbound); the loop-guard is the belt-and-braces backstop.

5. **Identify the channel by its inbound signature, not by content heuristics.** Match the structural
   marker the harness stamps on channel inbounds (a `source=`/channel tag), so ordinary local turns and
   tool results never trip the gate.

6. **Register it like any other security-relevant hook.** Deploy the script, make it executable, wire it
   into the harness config under the `Stop` event, add it to whatever hook **allowlist / integrity**
   mechanism the fleet runs (so a monitor doesn't read the new wiring as tampering), and keep a
   version-controlled canonical for byte-integrity checks.

**One-line form:** *If a human's message came in over a tool-delivered channel and no send-tool fired
since, a `Stop` hook must block the turn from ending — because "I replied" in the transcript is not
"they received it."*

## Reference implementation

Pantheon: `tools/hooks/telegram-reply-gate.py` (deployed at `~/.claude/hooks/`), wired as a `Stop` hook
in the agent's `settings.json`, allowlisted in `context_fill_monitor.py`'s `HOOK_ALLOWLIST`. Unit-tested
against synthetic transcripts for: unanswered→block, answered→allow, early-ack-then-work→allow,
re-armed-by-fresh-inbound→block, no-inbound→allow, loop-guard→allow, missing-transcript→fail-open.
