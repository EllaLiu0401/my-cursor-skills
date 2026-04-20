---
name: slack-status-update
description: Draft and send Slack status update messages for engineering work. Use when the user asks to write a Slack message, draft a status update, send a message to a teammate or channel about bugs, issues, investigation results, PRs, or asks for support/help via Slack.
---

# Slack Status Update

Compose concise, professional Slack messages for engineering status updates, investigation reports, and support requests. Uses Slack MCP to search users/channels and create drafts.

## Message Style

- English, professional but friendly
- Concise and scannable — use bold headers, bullet points, short paragraphs
- Polite and collaborative tone (not demanding)
- Structure: what you need first, then context, then what you'll do

## Slack mrkdwn Formatting

Slack uses its own markdown variant, NOT standard markdown:

- Bold: `*text*` (single asterisk)
- Italic: `_text_` (underscore)
- Code: `` `code` `` (backtick)
- Strikethrough: `~text~`
- Blockquote: `>text`
- Bullet list: `•` character or `- ` (NOT `*` for lists)
- Numbered lists: `1.` works
- No headings (`#` does NOT work) — use `*Bold Text*` as section headers
- Line breaks: `\n` in the message string
- Emoji: `:emoji_name:` (e.g. `:white_check_mark:`, `:warning:`, `:heart_hands:`)

## Message Template

```
[Greeting + one-line summary of what you need]

*[Section Header]*

[Key question or decision needed — put this FIRST if applicable]
• Option A — brief description
• Option B — brief description

*[What I Found / Context]*

*1. Issue title (severity):* Description of the issue.

*2. Issue title:* Description.

*3. Issue title (severity):* Description.

[One line about who IS / ISN'T affected]

*[What I'll Do]*

• Action item 1
• Action item 2

Thanks!
```

## Workflow

1. User describes the situation and who to send to
2. Compose message following the template and style above
3. Show the draft to user for review as a plain text block
4. When approved, use Slack MCP:
   - `slack_search_users` to find the recipient's user ID
   - `slack_search_channels` to find channel ID (if sending to a channel)
   - `slack_send_message_draft` to create the draft (user sends manually)
   - Only use `slack_send_message` if user explicitly says to send directly

## Key Rules

- Always create a **draft** unless user explicitly asks to send directly
- If draft already exists for that channel, tell user to delete it first
- One draft per channel — Slack limitation
- Use user ID as `channel_id` for DMs
- Keep messages under 4000 characters (Slack limit)
