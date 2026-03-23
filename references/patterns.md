# Patterns Reference

Complete JSON examples for every inline keyboard pattern. Each example includes the
message text, the `reply_markup` JSON, and the equivalent OpenClaw `buttons` format.

---

## 1. Yes / No Confirmation

**Use case:** "Do you want to proceed?", "Confirm deletion?", accept/reject flows.

For consequential actions, use emoji + short text so the meaning is unambiguous. For
low-stakes confirmations, emoji-only is fine.

### Consequential (emoji + text)

**Message text:**
```
🗑 Delete all completed tasks?
This action cannot be undone.
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "✅ Yes, delete", "callback_data": "yn:yes" },
      { "text": "❌ Cancel", "callback_data": "yn:no" }
    ]
  ]
}
```

### Low-stakes (emoji-only)

**Message text:**
```
Save this to your reading list?
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "✅", "callback_data": "yn:yes" },
      { "text": "❌", "callback_data": "yn:no" }
    ]
  ]
}
```

Emoji-only buttons are small, fast to scan, and don't clutter the chat. Use them for
yes/no, like/dislike (👍/👎), allow/deny (✅/🚫), or any binary choice where the
message text already explains what's being decided.

**OpenClaw buttons:**
```json
{
  "action": "send",
  "channel": "telegram",
  "to": "<chat_id>",
  "message": "🗑 Delete all completed tasks?\nThis action cannot be undone.",
  "buttons": [
    [
      { "text": "✅ Yes, delete", "callback_data": "yn:yes" },
      { "text": "❌ Cancel", "callback_data": "yn:no" }
    ]
  ]
}
```

**callback_data map:**

| callback_data | Meaning          |
|---------------|------------------|
| `yn:yes`      | User confirmed   |
| `yn:no`       | User cancelled   |

---

## 2. Multiple Choice — Single Select

**Use case:** Picking one option from a list (food, language, priority, category).

### 2a. Vertical layout (long labels)

Use one button per row when labels are longer than ~12 characters. This prevents
truncation on small screens and keeps each option easy to read and tap.

**Message text:**
```
🍽 What would you like for lunch?
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [{ "text": "🍕 Margherita Pizza", "callback_data": "mc:lunch:pizza" }],
    [{ "text": "🍔 Classic Burger", "callback_data": "mc:lunch:burger" }],
    [{ "text": "🥗 Caesar Salad", "callback_data": "mc:lunch:salad" }],
    [{ "text": "🍜 Ramen Bowl", "callback_data": "mc:lunch:ramen" }],
    [
      { "text": "✏️ Other", "callback_data": "mc:lunch:other" },
      { "text": "❌ Cancel", "callback_data": "mc:lunch:cancel" }
    ]
  ]
}
```

When the user taps "✏️ Other", the bot replies with "Type what you'd like instead"
and awaits a free-text message. When they tap "❌ Cancel", the keyboard is removed
and the bot continues without an answer.

### 2b. Grid layout (short labels)

Use rows of 2–3 buttons when labels are very short (2–6 characters, typically
emoji + abbreviation). This keeps the keyboard compact without sacrificing readability.

**Message text:**
```
🌐 Select your language:
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🇬🇧 EN", "callback_data": "mc:lang:en" },
      { "text": "🇪🇸 ES", "callback_data": "mc:lang:es" },
      { "text": "🇫🇷 FR", "callback_data": "mc:lang:fr" }
    ],
    [
      { "text": "🇩🇪 DE", "callback_data": "mc:lang:de" },
      { "text": "🇮🇹 IT", "callback_data": "mc:lang:it" },
      { "text": "🇵🇹 PT", "callback_data": "mc:lang:pt" }
    ],
    [
      { "text": "✏️ Other", "callback_data": "mc:lang:other" },
      { "text": "❌ Cancel", "callback_data": "mc:lang:cancel" }
    ]
  ]
}
```

**callback_data map:**

| callback_data    | Meaning                                    |
|------------------|--------------------------------------------|
| `mc:lang:en`     | English                                    |
| `mc:lang:es`     | Spanish                                    |
| `mc:lang:other`  | User wants to type a language not listed    |
| `mc:lang:cancel` | User aborts the question                   |

---

## 3. Multiple Choice — Multi Select (Toggle)

**Use case:** Selecting multiple items (toppings, tags, features). Buttons act as toggles.
Re-render the keyboard after each tap to show selected state.

**Message text (initial):**
```
🍕 Choose your toppings (tap to toggle):

⬜ Mozzarella
⬜ Pepperoni
⬜ Mushrooms
⬜ Olives

Tap "Done" when finished.
```

**reply_markup (initial):**
```json
{
  "inline_keyboard": [
    [{ "text": "⬜ Mozzarella", "callback_data": "ms:top:mozz" }],
    [{ "text": "⬜ Pepperoni", "callback_data": "ms:top:pepp" }],
    [{ "text": "⬜ Mushrooms", "callback_data": "ms:top:mush" }],
    [{ "text": "⬜ Olives", "callback_data": "ms:top:oliv" }],
    [{ "text": "✏️ Add custom", "callback_data": "ms:top:other" }],
    [
      { "text": "✅ Done", "callback_data": "ms:top:done" },
      { "text": "❌ Cancel", "callback_data": "ms:top:cancel" }
    ]
  ]
}
```

"Add custom" lets the user type a topping not in the list. The bot adds it to
their selection and re-renders the keyboard with the custom item included.

**After user taps Mozzarella and Mushrooms — editMessageReplyMarkup:**
```json
{
  "inline_keyboard": [
    [{ "text": "✅ Mozzarella", "callback_data": "ms:top:mozz" }],
    [{ "text": "⬜ Pepperoni", "callback_data": "ms:top:pepp" }],
    [{ "text": "✅ Mushrooms", "callback_data": "ms:top:mush" }],
    [{ "text": "⬜ Olives", "callback_data": "ms:top:oliv" }],
    [{ "text": "✅ Done", "callback_data": "ms:top:done" }]
  ]
}
```

The toggle state (which items are selected) lives server-side. The `callback_data` stays
the same — the handler looks up current state, flips the toggled item, and re-renders.

---

## 4. Multi-Question Panel (Collecting Multiple Answers)

**Use case:** Collecting answers to 2–4 short questions in sequence. Good for
quick surveys, settings, or preference collection.

### The golden rule: collect silently, submit explicitly

When a panel has multiple taps, the bot **must not generate a response after each
tap**. Each button press silently records the answer (at most a brief toast via
`answerCallbackQuery` like "Dark ✅"). The bot only acts on the collected answers
when the user taps a final **"✅ Confirm"** button.

### No preamble, no explanation

The bot sends the questions with their buttons and nothing else. It does NOT:
- Announce what's coming ("Three questions await your selections")
- Explain how the keyboard works ("I shall collect your answers silently")
- Describe the submit/cancel mechanics ("Tap Submit when you're done")
- Add conversational padding ("Please proceed at your leisure")
- List the questions in advance before showing them

The questions speak for themselves. If the user sees a question with buttons,
they know what to do. Any surrounding text is noise that pushes the buttons
further down the screen.

**Wrong:**
```
Here are three questions for your preferences. I will collect your answers
silently and process them when you tap Submit.

1. Theme (text vs. dark mode)
2. Notifications (frequency)
3. Language

Let's begin!
```

**Right:** Just send the first question directly:
```
🎨 Theme:
```
(with buttons attached)

### Approach A: One message per question (recommended)

Send each question as a separate message with its own inline keyboard. Buttons
always appear directly below their question. No preamble before the first question.

**Message 1:**
```
📋 Quick setup (1/3)

🎨 Theme:
```

**reply_markup for message 1:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🌙 Dark", "callback_data": "pnl:theme:dark" },
      { "text": "☀️ Light", "callback_data": "pnl:theme:light" },
      { "text": "🔄 Auto", "callback_data": "pnl:theme:auto" }
    ]
  ]
}
```

When tapped: silently record the answer, `answerCallbackQuery("Dark ✅")`,
then edit the message to show the selection and remove the keyboard:

**Message 1 after tap (editMessageText + remove reply_markup):**
```
📋 Quick setup (1/3)

🎨 Theme: 🌙 Dark ✅
```

Then immediately send message 2:

**Message 2:**
```
📋 Quick setup (2/3)

🔔 Notifications:
```

**reply_markup for message 2:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🔔 All", "callback_data": "pnl:notif:all" },
      { "text": "🔕 Muted", "callback_data": "pnl:notif:mute" },
      { "text": "⚡ Urgent", "callback_data": "pnl:notif:urg" }
    ]
  ]
}
```

**Message 3 (after message 2 is answered):**
```
📋 Quick setup (3/3)

🌐 Language:
```

**reply_markup for message 3:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🇬🇧 EN", "callback_data": "pnl:lang:en" },
      { "text": "🇪🇸 ES", "callback_data": "pnl:lang:es" },
      { "text": "🇩🇪 DE", "callback_data": "pnl:lang:de" }
    ],
    [
      { "text": "✏️ Other", "callback_data": "pnl:lang:other" }
    ]
  ]
}
```

**After all 3 answered — final confirmation message:**
```
📋 Your preferences:

🎨 Theme: 🌙 Dark
🔔 Notifications: ⚡ Urgent only
🌐 Language: 🇪🇸 ES

All correct?
```

**reply_markup for confirmation:**
```json
{
  "inline_keyboard": [
    [
      { "text": "✅ Confirm", "callback_data": "pnl:confirm" },
      { "text": "🔄 Start over", "callback_data": "pnl:restart" },
      { "text": "❌ Cancel", "callback_data": "pnl:cancel" }
    ]
  ]
}
```

Only when the user taps "✅ Confirm" does the bot process and apply the settings.
This is the **collect silently, submit explicitly** pattern.

### Approach B: Single-message compact panel

For very simple cases (2–3 questions, 2–3 short-label options each), a single
message can work if the keyboard rows are clearly separated and each row's purpose
is obvious from its position in the message text.

**Message text:**
```
📋 Quick preferences:

1️⃣ Theme → pick from row 1
2️⃣ Notifications → pick from row 2
3️⃣ Language → pick from row 3
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🌙 Dark", "callback_data": "pnl:theme:dark" },
      { "text": "☀️ Light", "callback_data": "pnl:theme:light" },
      { "text": "🔄 Auto", "callback_data": "pnl:theme:auto" }
    ],
    [
      { "text": "🔔 All", "callback_data": "pnl:notif:all" },
      { "text": "🔕 Muted", "callback_data": "pnl:notif:mute" },
      { "text": "⚡ Urgent", "callback_data": "pnl:notif:urg" }
    ],
    [
      { "text": "🇬🇧 EN", "callback_data": "pnl:lang:en" },
      { "text": "🇪🇸 ES", "callback_data": "pnl:lang:es" },
      { "text": "🇩🇪 DE", "callback_data": "pnl:lang:de" }
    ],
    [
      { "text": "✅ Confirm", "callback_data": "pnl:confirm" },
      { "text": "❌ Cancel", "callback_data": "pnl:cancel" }
    ]
  ]
}
```

Each tap silently marks the selection (edit the reply_markup to add a ✅ prefix
to the chosen button, e.g. "✅ Dark" replacing "🌙 Dark") and shows a brief toast.
The bot only processes the answers when "✅ Confirm" is tapped.

**Important:** This only works when the numbered list in the text clearly maps to
the keyboard rows below. If there's any ambiguity about which buttons belong to
which question, use Approach A (one message per question) instead.

For panels where an option row has limited choices (like language), consider
adding "✏️ Other" as an extra button in that row to allow free-text input.

---

## 5. Multi-Step Wizard

**Use case:** Onboarding, registration, booking, multi-field forms. Each step is either
a new message or an edit of the existing one.

Wizards are the right tool when a single keyboard would be overwhelming. Each step
should present one question with 2–5 options — small enough to take in at a glance.
If a step still feels crowded, it probably should be two steps.

### Step 1 of 3 — Name

**Message text:**
```
🧙 Setup (1/3)

What should I call you?
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "Use my Telegram name", "callback_data": "wiz:1:name:tg" }
    ],
    [
      { "text": "❌ Cancel", "callback_data": "wiz:cancel" }
    ]
  ]
}
```

For free-text input on a wizard step, the bot waits for a regular text message
(not a callback). The keyboard can offer quick-pick options while still accepting
typed input. When the user sends text, the bot advances to the next step.

### Step 2 of 3 — Role

**Message text (after editMessageText):**
```
🧙 Setup (2/3)

👤 Name: Luis ✅

Your role?
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "💻 Developer", "callback_data": "wiz:2:role:dev" },
      { "text": "🎨 Designer", "callback_data": "wiz:2:role:des" }
    ],
    [
      { "text": "📊 Manager", "callback_data": "wiz:2:role:mgr" },
      { "text": "📝 Other", "callback_data": "wiz:2:role:oth" }
    ],
    [
      { "text": "◀️ Back", "callback_data": "wiz:back:1" },
      { "text": "❌ Cancel", "callback_data": "wiz:cancel" }
    ]
  ]
}
```

### Step 3 of 3 — Confirm

**Message text:**
```
🧙 Setup (3/3)

👤 Name: Luis ✅
💼 Role: Developer ✅

All correct?
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "✅ Confirm", "callback_data": "wiz:3:confirm" },
      { "text": "🔄 Start over", "callback_data": "wiz:restart" }
    ],
    [
      { "text": "◀️ Back", "callback_data": "wiz:back:2" },
      { "text": "❌ Cancel", "callback_data": "wiz:cancel" }
    ]
  ]
}
```

### Wizard state management

State can be tracked in three ways (from simplest to most robust):

1. **In callback_data** — Encode accumulated answers in the payload. Works for 2–3
   short answers but hits the 64-byte limit fast.
   ```
   wiz:3:confirm:name=Luis&role=dev    (41 bytes — OK)
   ```

2. **Server-side session keyed by chat_id** — Store a `{ step, answers }` object
   in memory or a database. `callback_data` only carries the current step's answer.
   This is the recommended approach.

3. **Hybrid** — Use a short session token in `callback_data` that maps to server
   state: `wiz:s:a7f3:role:dev` → look up session `a7f3`.

---

## 6. Rating / Scale

**Use case:** 1–5 star rating, NPS score, satisfaction survey.

**Message text:**
```
⭐ How would you rate your experience?
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "1 ⭐", "callback_data": "rate:1" },
      { "text": "2 ⭐", "callback_data": "rate:2" },
      { "text": "3 ⭐", "callback_data": "rate:3" },
      { "text": "4 ⭐", "callback_data": "rate:4" },
      { "text": "5 ⭐", "callback_data": "rate:5" }
    ]
  ]
}
```

---

## 7. Pagination

**Use case:** Browse a list of items (search results, catalog, history) with prev/next.

**Message text:**
```
📚 Results (page 2 of 5):

1. Item Alpha
2. Item Beta
3. Item Charlie
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [{ "text": "Item Alpha", "callback_data": "pg:sel:alpha" }],
    [{ "text": "Item Beta", "callback_data": "pg:sel:beta" }],
    [{ "text": "Item Charlie", "callback_data": "pg:sel:charlie" }],
    [
      { "text": "◀️ Prev", "callback_data": "pg:prev:1" },
      { "text": "2/5", "callback_data": "pg:noop" },
      { "text": "Next ▶️", "callback_data": "pg:next:3" }
    ]
  ]
}
```

On the first page, omit or disable the "Prev" button. On the last page, omit "Next".
The page number in `callback_data` tells the handler which offset to load.

---

## 8. URL + Callback Mixed

Inline keyboards can mix `callback_data` buttons with `url` buttons in the same layout:

```json
{
  "inline_keyboard": [
    [
      { "text": "📖 Read more", "url": "https://example.com/article" },
      { "text": "🔖 Save", "callback_data": "act:save:42" }
    ],
    [
      { "text": "👍", "callback_data": "act:like:42" },
      { "text": "👎", "callback_data": "act:dislike:42" }
    ]
  ]
}
```

URL buttons show a small arrow icon and open the link in the user's browser. They
don't generate a `callback_query`.

---

## 9. Action Buttons (contextual operations)

**Use case:** Attaching quick operations to informational messages — notifications,
status updates, search results, summaries. Instead of just presenting information,
the bot offers to *do something* with it right there. The user acts with a single
tap rather than composing a follow-up message.

Action buttons turn the bot from a reporter into an assistant. The key principle is:
**think about what the user is likely to want to do next**, and offer it as a button.

### Common action buttons

| Button               | callback_data         | What the bot does                                  |
|----------------------|-----------------------|----------------------------------------------------|
| 💤 Snooze            | `do:snz:<id>:<mins>`  | Reschedule the notification and re-deliver later   |
| 🔔 Notify on change  | `do:watch:<id>`       | Set up a watch/monitor and alert when something changes |
| 💾 Remember this     | `do:save:<id>`        | Persist the info in the agent's memory / records   |
| 📌 Pin               | `do:pin:<id>`         | Pin or bookmark for quick retrieval later          |
| ✅ Mark done          | `do:done:<id>`        | Mark a task or reminder as completed               |
| 🔇 Mute              | `do:mute:<id>`        | Suppress future notifications about this topic     |
| 📤 Forward           | `do:fwd:<id>`         | Forward the message to another chat or channel     |
| 🔄 Refresh           | `do:refresh:<id>`     | Re-fetch and update the data in the message        |

These are not exclusive — think of them as a vocabulary. Pick the 1–3 actions that
make sense for the specific message, don't slap all of them on every notification.

### Example: flight status notification

**Message text:**
```
✈️ Flight IB3214 — Madrid → Valencia
Status: Delayed (new departure 18:45, was 17:30)
Gate: B22
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🔔 Notify on next update", "callback_data": "do:watch:ib3214" },
      { "text": "💾 Remember", "callback_data": "do:save:ib3214" }
    ],
    [
      { "text": "💤 Snooze 30m", "callback_data": "do:snz:ib3214:30" },
      { "text": "🔇 Mute", "callback_data": "do:mute:ib3214" }
    ]
  ]
}
```

### Example: task reminder

**Message text:**
```
🔔 Reminder: Review PR #482 (due today)
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "✅ Done", "callback_data": "do:done:pr482" },
      { "text": "💤 Snooze 1h", "callback_data": "do:snz:pr482:60" }
    ],
    [
      { "text": "💤 Snooze tomorrow", "callback_data": "do:snz:pr482:1d" }
    ]
  ]
}
```

### Example: price/stock alert

**Message text:**
```
📉 BTC dropped below $60,000 (currently $59,842)
```

**reply_markup:**
```json
{
  "inline_keyboard": [
    [
      { "text": "🔔 Alert if < $58k", "callback_data": "do:watch:btc:58000" },
      { "text": "💾 Log price", "callback_data": "do:save:btc:59842" }
    ],
    [
      { "text": "🔇 Stop alerts", "callback_data": "do:mute:btc" },
      { "text": "🔄 Refresh", "callback_data": "do:refresh:btc" }
    ]
  ]
}
```

### Snooze variations

Snooze is especially useful because it respects the user's attention. Instead of
dismissing a notification and forgetting it, or having to deal with it *right now*,
the user can push it to a better moment:

| Button text          | callback_data           | Duration              |
|----------------------|-------------------------|-----------------------|
| 💤 15m               | `do:snz:<id>:15`        | 15 minutes            |
| 💤 1h                | `do:snz:<id>:60`        | 1 hour                |
| 💤 Tomorrow          | `do:snz:<id>:1d`        | Next day, 9:00 AM     |
| 💤 Monday            | `do:snz:<id>:mon`       | Next Monday, 9:00 AM  |
| 💤 Custom            | `do:snz:<id>:pick`      | Ask user for time     |

For the "Custom" option, the bot can follow up with a second keyboard offering
time slots, or switch to free-text input for a natural language time expression
(e.g. "in 3 hours", "Friday afternoon").

### Design principles for action buttons

- **1–3 actions per message.** More than that and the user has to think about
  which button to press, which defeats the purpose of quick actions.
- **Context-specific, not generic.** Don't add "💾 Remember" to every message —
  add it when the message contains information worth persisting (a booking
  confirmation, a decision, a price point).
- **The action should be self-explanatory from the button + message.** The user
  should never wonder "what will this button do?" If the action needs explanation,
  the label is too vague.
- **Confirm destructive actions.** "🔇 Mute" on a critical alert should ask for
  confirmation ("Mute all BTC alerts? ✅/❌") before actually muting.
- **After acting, update the message.** When the user taps "💤 Snooze 1h",
  edit the message to show "💤 Snoozed until 15:30" and remove the keyboard.
  The user should see that their tap had an effect.

### OpenClaw integration for actions

OpenClaw's agent capabilities map naturally to these actions:

- **Snooze** → Schedule a delayed re-delivery of the message using OpenClaw's
  heartbeat or scheduling system.
- **Notify on change** → Set up a monitoring task. The agent periodically checks
  the data source and sends a new notification when something changes.
- **Remember this** → Persist the key information into the agent's memory via
  the workspace or memory system.
- **Mark done** → Update the task/reminder status in whatever system tracks it
  (OpenClaw reminders, a connected calendar, a project management tool).
- **Mute** → Add a suppression rule to the agent's notification preferences.

The bot handles all of this internally after the user's single tap — that's the
whole point. The user doesn't need to know *how* it's done, just that tapping
"💤 Snooze 1h" means "remind me again in an hour."

---

## Anti-patterns

❌ **Don't explain how the keyboard works.** The bot must never say "I will
   collect your answers silently", "tap Submit when you're done", or "here
   are three questions". The questions and buttons are the interface — they
   don't need a user manual. Any preamble or meta-commentary is noise that
   pushes the actual buttons further down the screen.

❌ **Don't trigger AI responses on every button tap.** This is the single most
   common mistake. When collecting multiple answers (panel, multi-select, wizard),
   each tap should silently record the selection — a brief toast at most. The AI
   only processes and responds when the user taps "✅ Confirm" or "💾 Submit".
   Otherwise the user drowns in mid-flow responses and the bot looks broken.

❌ **Don't separate questions from their buttons.** Each question's options must
   appear immediately below that question. If you're sending multiple questions,
   use one message per question (each with its own keyboard). Never send a wall
   of text listing all questions followed by a detached blob of buttons — the
   user can't tell which buttons belong to which question.

❌ **Don't use keyboards for everything.** A question with an obvious answer, an
   open-ended prompt, or a simple confirmation in a flowing conversation doesn't
   need buttons. Keyboards are accelerators for predictable choices, not the
   default communication mode.

❌ **Don't omit "Other" / free-text escape.** If your list of options is anything
   less than exhaustive (and it almost always is), the user needs a way to type
   their own answer. Forcing a closed set when the real answer isn't listed is
   one of the worst UX failures in bot design.

❌ **Don't trap the user in a flow.** Every multi-step or multi-question keyboard
   must have a Cancel button. The user should be able to abandon a wizard at any
   step without side effects.

❌ **Don't make the user scroll to see all buttons.** If a keyboard exceeds ~5 rows,
   break it into steps or use pagination. A keyboard that pushes the message text
   off-screen is a keyboard nobody reads.

❌ **Don't use long labels in grid layouts.** A row of 3 buttons with 15-word labels
   truncates into gibberish on mobile. If labels are longer than ~12 characters,
   stack vertically (one per row). If they're short, grid is fine.

❌ **Don't write "Yes" / "No" when ✅ / ❌ will do.** For binary choices where the
   message text already explains the decision, emoji-only buttons are cleaner and
   faster. Save text labels for actions with consequences where ambiguity matters.

❌ **Don't put more than 8 buttons per row** — Telegram rejects it.

❌ **Don't use UUIDs in callback_data** — A single UUID is 36 bytes, leaving only
   28 bytes for everything else.

❌ **Don't mix `callback_data` and `url` on the same button** — Each button must
   have exactly one of these fields.

❌ **Don't forget `answerCallbackQuery`** — The spinner will hang forever.

❌ **Don't send a new message per wizard step if you can `editMessageText`** — It
   clutters the chat.

❌ **Don't rely on button order for meaning** — Use explicit `callback_data` values.
