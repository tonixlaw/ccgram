# Overview: Restore Multi-user Group Session Binding

## Problem Identified
Your analysis is exact. The multi-user deduction strategy and manual group mapping were introduced in older commits (`400d9a2` and `70de860`) on the `feat/multiusers` branch. However, upon investigating this exact codebase branch state:

1. `thread_router.py` correctly establishes group chat bindings and `iter_topic_representatives`.
2. `hook_events.py` correctly uses `_dedup_users_by_topic` to enforce representative iteration.
3. **Regression found**: The recent modularity refactoring on `main` deleted the main `bot.py` handlers and transposed them into `src/ccgram/handlers/message_routing.py`. The inline `handle_new_message` loop iterating over topic representation sets was lost, executing the LLM response loop separately for independent grouped IDs and firing duplicate `enqueue_content_message` hits!

## Action Plan

Since the structural commits are already natively part of `feat/multiusers`, we do not need to execute any `git merge origin/dev`. We just need to implement the lost deduplication state directly onto `message_routing.py`. 

- Re-add internal deduplication tracking sets (`cleared_interactive_topics`, `rendered_interactive_topics`, `delivered_topics`) inside `handle_new_message()`.
- Wrap the topic UI state updates so that interactive components and response fragments are exclusively sent once per representative group identity, without stripping the `user_preferences.update_user_window_offset` global offset update for the silent observers.

## User Review Required
> [!IMPORTANT] 
> No branch merges will be performed. I'll execute the `message_routing.py` deduplication code update entirely within our currently active context, effectively finishing the true stabilization of multi-user group behavior requested on `feat/multiusers`.
