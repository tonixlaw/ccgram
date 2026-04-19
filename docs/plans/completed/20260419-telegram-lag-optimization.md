# Overview: Resolve Telegram message lag by throttling UI edits

## Issue Description
Telegram messages were lagging significantly behind actual events in the tmux window, sometimes by several minutes. This made the bot feel unresponsive and difficult to use for real-time monitoring of long-running commands.

## Problem Identified
The codebase polls tmux status every 1.5 seconds. For windows with passive shell output monitoring or status bubbles (spinners/timers), each poll often triggered an in-place message edit via Telegram's `editMessageText` API. 

Telegram imposes a rate limit of approximately 20 messages per minute in group chats. Our 1.5s poll cycle (40 edits/minute) was consistently hitting this limit. When the bot received a `RetryAfter` error, the message queue worker would sleep for the penalty duration (30-60 seconds), causing all subsequent messages (including AI responses) to buffer and lag.

## Action Taken
1. **Throttled Shell Relay**: Modified `src/ccgram/handlers/shell_capture.py` to record `last_relay_time`. Passive output updates are now skipped if less than 3.0 seconds have elapsed since the last update, unless the command has finished (has an exit code).
2. **Throttled Status Bubble**: Modified `src/ccgram/handlers/status_bubble.py` to store `last_edit_time` in the `_status_msg_info` registry. Status updates are now skipped if within the 3.0s window, unless it's a critical transition (e.g., moving to or from an 'idle' state).
3. **Optimized API Usage**: By reducing the frequency of intermediate UI edits, we stay well under Telegram's flood control limits, ensuring that the message queue remains unblocked and responsive for actual content delivery.

Created At: 2026-04-19
