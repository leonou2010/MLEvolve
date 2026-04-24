# MLEvolve Bug Consultant — Change Log

## Fix 7: Conditional Parent Bug Rule

**Problem:** When a child node crashes due to a latent bug in its valid parent's code (not from what
the child itself added), the global BANNED rule was too generic. The child saw "BANNED: no duplicate
features" but did not know that the specific parent code contained `assemble_full_features()` that
produces duplicates — so it could not fix it.

**Fix:** When a child crashes AND its parent was valid (no `exc_type`), analyze the parent code,
child diff, and error to generate a **parent-specific conditional rule**. Store it keyed by parent
node ID. When any future node improves from that same parent, inject the conditional rule
prominently into the prompt — before the global Bug Prevention Alert — so the LLM knows exactly
what pattern to fix.

**Files changed:**

- `agents/consultant/bug_consultant.py`
  - Added `learn_conditional_rule_spec` FunctionSpec.
  - Added `self.parent_conditional_rules: dict` to `__init__`.
  - Added `learn_conditional_rule(parent_id, parent_code, child_diff, error_type, error_msg)` —
    background-thread LLM call, gated by `_write_lock`, produces `"When the existing code has
    [PATTERN]: BANNED: [TRIGGER] (causes [ERROR_TYPE])"`.
  - Added `get_conditional_guidance(parent_id) -> str` — returns stored rule or `""`.

- `engine/agent_search.py`
  - After each `learn_from_bug` call (in `_run_single_step` and `execute_deferred_node`), spawns a
    daemon thread calling `learn_conditional_rule` when the result node is buggy and its parent was
    valid.

- `agents/improve_agent.py`
  - Injects `conditional_section` (parent-specific warning) immediately before the global Bug
    Prevention Alert in both full-rewrite and diff-mode paths.

- `agents/evolution_agent.py`
  - Same injection as `improve_agent.py`.

- `agents/fusion_agent.py`
  - Same injection in both `fuse_two_nodes` and `_fuse_with_multiple_references`.

**Constraints honoured:**
- All new code gated by `if getattr(agent, 'bug_consultant', None)`.
- No FIX/USE line in the LLM prompt for `learn_conditional_rule`.
- LLM call runs in a background daemon thread (non-blocking).
- `None` parent handled gracefully throughout.
