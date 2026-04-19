# Overview: Multi-User Topic Autobinding Fix

## Issue Description
When an authorized user sends a message in a Telegram group topic where another user already has an active session, the bot fails to recognize the existing shared session. Instead of automatically binding the new user to the existing tmux window, the bot asks the new user to select a directory, leading to redundant session creation and duplicate tmux windows for the same topic.

## Problem Identified
When multiple authorized users are in the same Telegram group chat and one has already started a session in a topic (mounted to a tmux window), the system correctly handled bindings for that first user. However, when a second user sent their first message in that same topic, the `get_window_for_thread` lookup queried the `thread_bindings` solely by their unique user ID. Because they hadn't explicitly been bound yet, the query failed, returning an "unbound topic" state. This resulted in the second user being prompted to select a directory, which subsequently created a duplicate session and a duplicate tmux window.

## Action Taken
1. Updated `ThreadRouter.get_window_for_thread()` in `src/ccgram/thread_router.py`.
2. Implemented a fallback `group_chat_ids` lookup: when a user issues a command or sends a message, their `chat_id` and `thread_id` context is used to discover if another authorized user in the *same* group chat has already bound a tmux window session to that exact topic.
3. If a shared session is found, the system automatically and silently performs `bind_thread` for the new user against the existing window.
4. Both users now elegantly share the same Telegram topic and same underlying tmux window, allowing simultaneous participation in the conversation without duplicate sessions being spawned.

Created At: 2026-04-19
