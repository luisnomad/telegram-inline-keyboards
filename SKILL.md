---
name: telegram-inline-keyboards
description: >
  Generate Telegram inline keyboard JSON for interactive bot conversations — yes/no prompts,
  multiple choice, multi-question panels, and multi-step wizards with state tracking.
  Use this skill whenever the user wants to add buttons, quick replies, option menus, polls,
  questionnaires, surveys, onboarding flows, settings panels, or any interactive UI to a
  Telegram bot. Also trigger when the user mentions OpenClaw inline buttons, callback_data
  design, or wizard/interview flows over Telegram. Covers both generating keyboard markup
  and a suggested callback routing pattern for handling responses.
---

# Telegram Inline Keyboards

Generate framework-agnostic JSON for Telegram Bot API inline keyboards. The output is always
a valid `reply_markup` object (or an OpenClaw `buttons` array) that any bot framework or
HTTP call can use directly.

## Core Concepts

Telegram offers two keyboard types. This skill focuses on **inline keyboards** (buttons
attached to a message) because they don't send visible messages to the chat and support
`callback_data` for programmatic routing.

### Inline vs Reply keyboards at a glance

- **InlineKeyboardMarkup** → buttons below a bot message; user taps, bot receives a
  `callback_query` update. No message appears in chat.
- **ReplyKeyboardMarkup** → buttons replace the device keyboard; tapping sends a regular
  text message. Good for persistent menus, not for structured data collection.

This skill produces `InlineKeyboardMarkup` JSON unless the user explicitly asks for a
reply keyboard.

## When NOT To Use Inline Keyboards

Having this skill available does not mean every message should sprout buttons. Keyboards
are a tool, not the default. Prefer plain text when:

- **The answer is open-ended.** "What should I name my project?" is a terrible
  multiple-choice question. Just let the user type.
- **The question is simple and context is obvious.** If the bot already knows what to
  do next, just do it — don't ask for confirmation on every step.
- **There's only one realistic option.** A keyboard with a single button is visual
  noise. Send a message instead.
- **The conversation is flowing naturally.** Interrupting a freeform chat with a
  structured keyboard panel feels robotic. Match the energy of the conversation.
- **You're tempted to turn a 1-message answer into a 4-step wizard.** Resist.
  Wizards exist for genuinely multi-field tasks (onboarding, booking, config). A
  question like "Want me to summarize this?" is just a yes/no, not a wizard.

Think of keyboards as **accelerators for predictable choices**, not a replacement for
conversation. The test: would a human assistant ask this as a menu, or would they just
talk? If the latter, skip the keyboard.

## Hard Limits (Telegram Bot API)

These are enforced server-side — violating them causes `400 Bad Request`:

| Constraint              | Limit                    |
|-------------------------|--------------------------|
| `callback_data` length  | **64 bytes** (not chars!) |
| Buttons per row         | 8                        |
| Total buttons           | 100                      |
| Button text             | No hard limit, but truncates on small screens |

The 64-byte limit on `callback_data` is the most common pitfall. Because it's bytes not
characters, non-ASCII text eats the budget fast. Stick to short ASCII tokens.

## callback_data Design

A good `callback_data` schema is the foundation of every pattern below. Use a compact,
parseable format:

```
<prefix>:<action>:<payload>
```

**Examples:**
- `yn:yes` / `yn:no` — simple yes/no
- `mc:q1:a` / `mc:q1:b` — multiple choice, question 1, option a/b
- `wiz:2:lang:es` — wizard step 2, field "lang", value "es"
- `pg:next:5` — pagination, next page, offset 5
- `do:snz:task42:60` — action: snooze item "task42" for 60 minutes
- `do:watch:btc:58000` — action: alert when BTC drops below 58000

Rules of thumb:
- Keep values as short ASCII tokens — single letters or short slugs
- Use `:` as separator (easy to split, no URL-encoding issues)
- Prefix with a namespace so different features don't collide
- If you need to pass an ID, use a short hash or numeric index, not a UUID

### Encoding helper (optional)

For payloads that risk exceeding 64 bytes, store the real data server-side keyed by a
short token and put only the token in `callback_data`:

```
callback_data: "ref:a7f3"   →   server lookup: a7f3 → { step: 3, answers: {...} }
```

## Patterns

Read `references/patterns.md` for full JSON examples of every pattern. Below is a summary.

### Mandatory Escape Hatches

Every keyboard that presents choices **must** include escape routes so the user is
never trapped in a flow or forced to pick from a list that doesn't fit:

**"Other" / free-text option.** Any multiple-choice keyboard (single select, panel
question, wizard step with predefined choices) should include an "✏️ Other" button
that switches the bot to accept free-text input for that question. The user's actual
answer is almost never limited to the 3–5 options you thought of. Omitting "Other"
silently forces the user into a box.

```json
[{ "text": "✏️ Other", "callback_data": "mc:food:other" }]
```

When "Other" is tapped: `answerCallbackQuery` with "Type your answer below", then
set the session to await a text message (same pattern as wizard free-text steps —
see `references/routing.md`).

**"Cancel" button.** Any keyboard that starts a flow (wizard, multi-question panel)
or asks for a consequential decision must include a cancel/abort button. For
single-shot questions (like a yes/no) the "No" option often serves this purpose,
but for wizards and panels, add an explicit cancel:

```json
[{ "text": "❌ Cancel", "callback_data": "wiz:cancel" }]
```

When cancelled: dismiss the keyboard, send a short confirmation ("Cancelled."), and
clean up any in-progress session state.

These are not optional — they're part of respecting the user's autonomy. A keyboard
without an exit is a cage.

### Silent Collection + Confirm to Submit

When a keyboard collects multiple answers (panels, multi-select, or any flow with
more than one tap), the bot **must not react to each individual tap**. Each tap
should silently record the selection (update the button visually via
`editMessageReplyMarkup`, show a brief toast via `answerCallbackQuery` at most) and
wait for the user to finish. The bot only processes all answers together when the
user explicitly taps a **"✅ Confirm"** or **"💾 Submit"** button.

Why: if the AI generates a full response after every single button press, the user
gets interrupted mid-flow, the chat fills with noise, and they're left wondering
whether the bot is still waiting for their other answers or has already acted.

The rule is simple: **collect silently, submit explicitly.**

### No Meta-Commentary

The bot must **never explain how the keyboard works**. No "I will collect your
answers silently", no "tap Submit when you're done", no "here are three questions
for you to answer". The questions and buttons speak for themselves — if they need
an instruction manual, the design is wrong.

Specifically, the bot must NOT:
- Announce how many questions are coming
- Explain that it will wait for all answers before responding
- Describe the confirm/cancel mechanics
- Narrate its own behavior ("I shall process everything together")
- Add pleasantries or padding around the keyboard ("Please proceed at your leisure")

The user sees a question with buttons. They tap. The next question appears. When
they confirm, the bot acts. That's it. Any additional text is noise.

### Buttons Next to Their Question

When sending multiple questions with buttons, each question's options must appear
**immediately below that question** — not lumped together at the end. Telegram
supports only one `reply_markup` per message, so the correct approach for multi-
question flows is:

- **One message per question**, each with its own inline keyboard, or
- **A single message** where the questions and options are interleaved in the text
  and the keyboard rows are visually grouped (using the panel pattern with clear
  label rows — see pattern 4 in `references/patterns.md`)

Never send a wall of text listing all questions followed by a detached grid of
buttons. The user can't tell which buttons belong to which question.

### 1. Yes / No confirmation

Single message, two buttons on one row. The simplest pattern.

```json
{
  "inline_keyboard": [
    [
      { "text": "✅ Yes", "callback_data": "yn:yes" },
      { "text": "❌ No", "callback_data": "yn:no" }
    ]
  ]
}
```

### 2. Multiple choice (single select)

One question, 2–6 options. Stack vertically for clarity when labels are long, or use
rows of 2–3 for short labels.

```json
{
  "inline_keyboard": [
    [{ "text": "🍕 Pizza", "callback_data": "mc:food:pizza" }],
    [{ "text": "🍔 Burger", "callback_data": "mc:food:burger" }],
    [{ "text": "🥗 Salad", "callback_data": "mc:food:salad" }],
    [{ "text": "✏️ Other", "callback_data": "mc:food:other" }]
  ]
}
```

### 3. Multi-question panel (combined layout)

When collecting 2–4 short answers, send **one message per question** — each with its
own inline keyboard directly below. Taps are recorded silently. After the last question,
send a summary message with a "✅ Confirm" button. Alternatively, for very compact
choices (2–3 options with short labels), use a single-message panel with clearly
labeled keyboard rows — but never a blob of text followed by a blob of buttons.

### 4. Multi-step wizard

A sequential flow where each step is a new message (or an edit of the same message).
The bot maintains state between steps — either in-memory, in a database, or encoded
in the `callback_data` prefix.

### 5. Rating / scale

A row of numbered buttons (1–5 stars, 1–10 scale, etc.).

### 6. Pagination

"Previous / Next" row appended below content buttons, with offset encoded in
`callback_data`.

### 7. Action buttons (contextual operations)

Buttons attached to informational messages that let the user *act* on what they just
read — without composing a follow-up. Think about what the user will want to do next:

| Button               | What it does                                        |
|----------------------|-----------------------------------------------------|
| 💤 Snooze            | Re-deliver the notification later                   |
| 🔔 Notify on change  | Watch a data source and alert when it changes        |
| 💾 Remember this     | Persist the info in the agent's memory              |
| ✅ Mark done          | Complete a task or reminder                          |
| 🔇 Mute              | Suppress future notifications on this topic          |
| 🔄 Refresh           | Re-fetch and update the data in-place                |

Pick 1–3 that fit the message context. These are especially powerful with OpenClaw,
which can handle snoozing, watching, and remembering internally without user effort.

For complete JSON and message text examples of all patterns, read:
→ `references/patterns.md`

## OpenClaw Integration

OpenClaw's Telegram channel supports inline buttons natively via the `buttons` field
in message actions. The `buttons` array maps directly to `inline_keyboard` rows:

```json
{
  "action": "send",
  "channel": "telegram",
  "to": "<chat_id>",
  "message": "Choose an option:",
  "buttons": [
    [
      { "text": "✅ Yes", "callback_data": "yn:yes" },
      { "text": "❌ No", "callback_data": "yn:no" }
    ]
  ]
}
```

Make sure `capabilities.inlineButtons` is enabled in your OpenClaw Telegram config
(scopes: `off`, `dm`, `group`, `all`, `allowlist`).

## Callback Routing Pattern

When the bot receives a `callback_query`, route it by parsing the prefix:

```
callback_data received → split on ":"
  ├─ prefix "yn"  → yes/no handler
  ├─ prefix "mc"  → multiple choice handler (read question ID + selected value)
  ├─ prefix "wiz" → wizard handler (read step + field + value, advance state)
  ├─ prefix "pg"  → pagination handler
  ├─ prefix "do"  → action handler (snooze, watch, save, done, mute, refresh)
  └─ unknown      → answer with "Unknown action" alert
```

**Always call `answerCallbackQuery`** after handling — even with an empty response.
Telegram shows a loading spinner on the button until you do.

For wizard flows, after processing a step:
1. Call `answerCallbackQuery` to dismiss the spinner
2. Either `editMessageText` + `editMessageReplyMarkup` to update the current message
   with the next step, or `sendMessage` with a new keyboard for the next step
3. On the final step, send a confirmation message and optionally remove the keyboard

For the multi-question panel, after each answer:
1. `answerCallbackQuery` with a short confirmation (e.g. "Pizza selected ✅")
2. `editMessageReplyMarkup` to visually mark the selected option (e.g. add a ✔ prefix)
3. When all questions are answered, edit the message to show the summary

See `references/routing.md` for a framework-agnostic pseudocode router and state
machine example.

## Output Format

When asked to generate a keyboard, produce:

1. **The message text** — what the user sees above the buttons
2. **The `reply_markup` JSON** — the `InlineKeyboardMarkup` object
3. **callback_data map** — a short table mapping each `callback_data` value to its meaning

If the user is working with OpenClaw, use the `buttons` array format instead of wrapping
in `reply_markup`.

## Readability & Layout

A keyboard the user has to squint at or scroll through is worse than no keyboard at all.
Every layout decision should serve scannability on a phone screen.

**Keep keyboards compact — never force scrolling.** A keyboard that pushes the message
text off-screen is too big. Aim for 3–5 rows max in a single message. If you have more
options than that, break them into a multi-step flow or use pagination. A 12-option
flat list is an eyesore; three steps of 4 options each is comfortable.

**Use multi-step flows to reduce complexity.** When a single keyboard would need 8+
buttons, split it into sequential steps. Each step should present one clear question
with a small set of choices. This is easier to read, easier to tap, and easier to
change your mind about (with Back).

**Avoid long text in grid/column layouts.** A row of 3 buttons works great when labels
are 2–4 words ("🇬🇧 EN", "🇪🇸 ES"). But a row of 3 buttons with 15-word labels
becomes an unreadable wall of truncated text. Rule of thumb: if labels are longer than
~12 characters, stack vertically (one button per row) instead of side by side.

**Use emoji as the primary label for binary/simple choices.** For yes/no, like/dislike,
allow/deny, approve/reject — emoji-first buttons are faster to scan and tap than text:

| Instead of this                | Use this          |
|-------------------------------|--------------------|
| `"Yes"` / `"No"`             | `"✅"` / `"❌"`    |
| `"Like"` / `"Dislike"`       | `"👍"` / `"👎"`    |
| `"Allow"` / `"Deny"`         | `"✅ Allow"` / `"🚫 Deny"` |
| `"Approve"` / `"Reject"`     | `"✅ Approve"` / `"❌ Reject"` |

For binary actions, emoji-only (no text) is fine — the meaning is universally clear.
For actions with consequences (approve/deny an exec request), pair the emoji with
a short word so there's zero ambiguity.

**Balance information density.** The message text above the keyboard should carry the
context (the question, progress info, previous answers). The buttons should carry only
the choices. Don't repeat the question inside button labels.

## Tips

- Keep button text under ~20 chars for mobile friendliness
- For wizards, show a progress indicator in the message text (e.g. "Step 2 of 4")
- For multi-question panels, show which questions are answered (✅) and pending (⬜)
- When editing a message to show the next wizard step, preserve the previous answers
  in the message text so the user can see their progress
- Prefer `editMessageText` over sending new messages to keep the chat clean
- For cancel/back buttons in wizards, use a consistent prefix like `wiz:back` or `wiz:cancel`
