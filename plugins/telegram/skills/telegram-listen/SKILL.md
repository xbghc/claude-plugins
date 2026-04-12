---
name: telegram-listen
description: >
  启动 Telegram 消息监听循环。当用户说"监听 Telegram"、"开始 Telegram"、
  "listen telegram"、"启动 bot" 等时触发。
  调用 start_listening → 进入 wait → 处理任务 → reply 回复 → 继续等待 的循环。
---

# Telegram Listen — 消息监听循环

通过 Telegram Bot 接收用户消息，执行任务，回复结果。

## 前置条件

启动监听前，提醒用户将权限模式切换为全部允许（如 `--dangerously-skip-permissions`），因为用户不在电脑前无法审批工具调用。如果用户确认已开启或明确表示不需要提醒，直接进入监听。

## 流程

1. 调用 `mcp__plugin_telegram_telegram__start_listening` 开始接收消息
2. 收到消息后，理解用户意图并执行对应任务（可自由使用所有可用工具）
3. 调用 `mcp__plugin_telegram_telegram__reply` 将结果回复到对应 `chat_id`
4. 继续等待下一条消息

## 规则

- 此模式服务于用户离开电脑的场景，必须自主完成任务，不要在终端等待用户确认
- 每次收到消息后，先完整执行任务，再回复结果
- 回复时使用消息中携带的 `chat_id`
- 如果任务执行失败，回复错误原因
- 如果消息内容不明确，通过 Telegram 回复询问用户意图，然后继续等待下一条消息
- 保持循环运行，直到用户在 Telegram 中发送"停止"或"stop"，或在终端中断
- 退出时调用 `mcp__plugin_telegram_telegram__stop_listening` 停止监听
