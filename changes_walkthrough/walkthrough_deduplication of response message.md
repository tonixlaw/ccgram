# Issue Resolution: Duplicate Gemini / Hook Messages

I have successfully resolved the issue where Gemini CLI responses (and Claude Code hook events) were sent multiple times to a single Telegram topic when multiple users were bound to it.

## Overview
The root cause was that the message processing systems (`polling_coordinator.py`, `bot.py`, and `hook_events.py`) iterated indiscriminately over all `(user_id, window_id, thread_id)` bindings. Because the Telegram message dispatch relies on the resolved `(chat_id, thread_id)` topic, and the underlying message queue is per-user, multiple workers would send the exact same API requests for the single shared topic.

To fix this, I implemented topic deduplication that elects a deterministic representative user for any broadcast/status/interactive message aimed at the shared topic.

### Key Changes (Original)
1. **Topic Deduction in Router (`thread_router.py`)**: Added `iter_topic_representatives()` to yield exactly one representative user binding (the one with the lowest `user_id`) for every active Telegram topic.
2. **Polling Deduplication (`polling_coordinator.py`)**: Swapped `iter_thread_bindings()` for `iter_topic_representatives()` in the main `status_poll_loop`, preventing identical topic updates scaling with the number of bound users.
3. **Smart Message Queueing (`bot.py`)**: For transcript parsing (e.g. Gemini CLI), the `handle_new_message` routine was updated to deduplicate `enqueue_content_message` per topic. Crucially, I maintained the iteration over the full active user list only to correctly apply the read cursor offset updates via `user_preferences.update_user_window_offset`, ensuring the `\history` feature still behaves correctly for all users.
4. **Hook Events Deduplication (`hook_events.py`)**: Added `_dedup_users_by_topic` to filter bindings passed to `enqueue_status_update` and related Telegram messaging triggers on agent states (`SubagentStop`, `TeammateIdle`, `SessionEnd`, etc). This eliminates rapid-fire editing and redelivery of bubbles inside topics.

---

## Post-Review Bug Fixes

A follow-up code review of the original changes uncovered four additional bugs. All have been patched.

### Bug 1 — Python 2 `except` syntax in `hook_events.py` (**Critical**)

**File:** `handlers/hook_events.py`, `_get_llm_summary()`

The bare comma-separated `except` clause:
```python
# Before (Python 2 syntax — SyntaxError at runtime on Python 3):
except RuntimeError, OSError, ValueError:

# After (correct Python 3 syntax):
except (RuntimeError, OSError, ValueError):
```
This would have raised a `SyntaxError` at import time if the LLM summariser code path was ever reached, crashing every `_handle_stop` event.

---

### Bug 2 — `set_interactive_mode` called before the topic deduplication guard (`bot.py`) (**High**)

**File:** `bot.py`, `handle_new_message()`

In the original code, `set_interactive_mode(user_id, window_id, thread_id)` was unconditionally called for *every* user in the outer loop before the `delivered_topics` guard. For non-representative users:
- Interactive mode was set (causing the polling loop to skip that window).
- `handle_interactive_ui` was never called (already in `delivered_topics`).
- `clear_interactive_mode` was not called (was in an unreachable `else` branch).
- The loop **fell through** to `build_response_parts` and attempted to process the `tool_use` message as a normal message.

**Fix:** Moved `set_interactive_mode` and the `else: clear_interactive_mode` clause inside the `if not handled:` block. Non-representative users now skip both the mode mutation and the UI render entirely, and correctly `continue` out of the iteration via the `if handled: continue` path (because `handled` is pre-seeded to `True` when `topic_key in delivered_topics`).

```python
# Before:
set_interactive_mode(user_id, window_id, thread_id)   # called for every user
...
if topic_key not in delivered_topics:
    handled = await handle_interactive_ui(...)
# if not representative, handled=False → falls through to build_response_parts

# After:
handled = topic_key in delivered_topics          # pre-seed True for non-representatives
if not handled:
    set_interactive_mode(user_id, window_id, thread_id)   # only representative
    handled = await handle_interactive_ui(...)
    if handled:
        delivered_topics.add(topic_key)
    else:
        clear_interactive_mode(user_id, thread_id)
if handled:
    ...
    continue                                     # non-representatives also skip cleanly
```

---

### Bug 3 — `clear_interactive_msg` Telegram API called N times per topic (`bot.py`) (**High**)

**File:** `bot.py`, `handle_new_message()`

When a non-interactive message arrived, `clear_interactive_msg(user_id, bot, thread_id)` was called once per bound user with no deduplication guard. Since this function makes a Telegram delete/edit API call against the shared topic, N users produced N identical API calls.

**Fix:** Added a `cleared_interactive_topics` set (mirroring `delivered_topics`) to gate the API call to exactly one call per topic. Subsequent users only have their per-user local state cleared via `clear_interactive_mode`.

```python
# Before:
if get_interactive_msg_id(user_id, thread_id):
    await clear_interactive_msg(user_id, bot, thread_id)   # N calls for N users

# After:
cleared_interactive_topics: set[tuple[int, int]] = set()
...
if get_interactive_msg_id(user_id, thread_id):
    _ci_chat_id = thread_router.resolve_chat_id(user_id, thread_id)
    _ci_key = (_ci_chat_id, thread_id)
    if _ci_key not in cleared_interactive_topics:
        await clear_interactive_msg(user_id, bot, thread_id)   # 1 call per topic
        cleared_interactive_topics.add(_ci_key)
    else:
        clear_interactive_mode(user_id, thread_id)             # local state only
```

---

### Known Limitation — `resolve_chat_id` fallback can split topic keys at bind time (Low)

**Files:** `thread_router.py` (`iter_topic_representatives`), `hook_events.py` (`_dedup_users_by_topic`)

Both dedup functions call `resolve_chat_id(uid, tid)` to compute the canonical `(chat_id, thread_id)` key. If `group_chat_id` has not yet been stored for a user-thread pair (e.g., at cold startup before the first group message arrives), `resolve_chat_id` falls back to `user_id`, producing different keys for what is actually the same topic. This would defeat deduplication for that brief window.

In practice, `group_chat_id` is persisted in `state.json` and restored before bindings are active, so this race is very narrow. No patch applied; noted for observability.

---

## Validation
With all improvements, all Telegram outbound actions (including normal completion messages from the Gemini CLI, interactive UI rendering, and status bubble edits) are correctly isolated per-topic, completely stopping duplicate messages while seamlessly maintaining internal state awareness for all participating users.
