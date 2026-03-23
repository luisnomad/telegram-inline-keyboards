# Callback Routing Reference

Framework-agnostic pseudocode for handling inline keyboard callbacks.

## Basic Router

Parse `callback_data` on `:` and dispatch by prefix. This works in any language
or framework — just adapt the syntax.

```
on callback_query(query):
    data = query.callback_data
    parts = data.split(":")
    prefix = parts[0]

    switch prefix:
        case "yn"  → handle_yes_no(query, parts)
        case "mc"  → handle_multiple_choice(query, parts)
        case "ms"  → handle_multi_select(query, parts)
        case "pnl" → handle_panel(query, parts)
        case "wiz" → handle_wizard(query, parts)
        case "rate" → handle_rating(query, parts)
        case "pg"  → handle_pagination(query, parts)
        case "act" → handle_action(query, parts)
        case "do"  → handle_do_action(query, parts)
        default    → answer_callback(query.id, "Unknown action")
```

## Yes/No Handler

```
handle_yes_no(query, parts):
    answer = parts[1]   // "yes" or "no"

    answer_callback(query.id)

    if answer == "yes":
        // Execute the confirmed action
        edit_message_text(query.message, "✅ Done!")
    else:
        edit_message_text(query.message, "❌ Cancelled.")

    // Remove the keyboard after acting
    edit_message_reply_markup(query.message, null)
```

## Multiple Choice Handler

```
handle_multiple_choice(query, parts):
    question_id = parts[1]   // e.g. "food", "lang"
    selected    = parts[2]   // e.g. "pizza", "en", "other", "cancel"

    if selected == "cancel":
        answer_callback(query.id, "Cancelled")
        edit_message_text(query.message, "Skipped.")
        edit_message_reply_markup(query.message, null)
        return

    if selected == "other":
        answer_callback(query.id, "Type your answer below")
        edit_message_reply_markup(query.message, null)
        // Set session flag so the next text message is captured as the answer
        set_awaiting_text(query.from.id, question_id)
        return

    answer_callback(query.id, "Selected: " + selected)

    // Store the answer
    save_answer(query.from.id, question_id, selected)

    // Update the message to reflect the choice
    edit_message_text(query.message, "You chose: " + selected)
    edit_message_reply_markup(query.message, null)
```

## Multi-Select (Toggle) Handler

```
handle_multi_select(query, parts):
    group   = parts[1]   // e.g. "top" (toppings)
    item    = parts[2]   // e.g. "mozz", "pepp", "done", "other", "cancel"

    if item == "cancel":
        answer_callback(query.id, "Cancelled")
        clear_selections(query.from.id, group)
        edit_message_text(query.message, "Selection cancelled.")
        edit_message_reply_markup(query.message, null)
        return

    if item == "other":
        answer_callback(query.id, "Type your custom item below")
        set_awaiting_text(query.from.id, group + ":custom")
        return    // Keep the keyboard visible so user can still tap Done after

    if item == "done":
        answer_callback(query.id, "Selection confirmed!")
        selections = get_selections(query.from.id, group)
        edit_message_text(query.message, "You selected: " + join(selections))
        edit_message_reply_markup(query.message, null)
        return

    // Toggle the item
    toggle_selection(query.from.id, group, item)
    answer_callback(query.id)

    // Re-render keyboard with updated checkmarks
    selections = get_selections(query.from.id, group)
    new_keyboard = build_multi_select_keyboard(group, all_items, selections)
    edit_message_reply_markup(query.message, new_keyboard)
```

## Panel Handler

The panel handler supports two approaches. Both follow the same rule:
**collect silently, submit explicitly** — never trigger a full AI response on
an individual button tap.

### Approach A: One message per question

Each question is its own message. Taps record silently, then advance to the
next question. Only the final "Confirm" triggers processing.

```
handle_panel(query, parts):
    field = parts[1]   // "theme", "notif", "lang", "confirm", "restart", "cancel"
    value = parts[2]   // "dark", "mute", "en", etc. (absent for confirm/restart/cancel)

    if field == "cancel":
        answer_callback(query.id, "Cancelled")
        clear_panel_state(query.from.id)
        edit_message_text(query.message, "❌ Cancelled.")
        edit_message_reply_markup(query.message, null)
        return

    if field == "restart":
        clear_panel_state(query.from.id)
        answer_callback(query.id, "Starting over")
        send_panel_question(query.message.chat.id, step=1)
        return

    if field == "confirm":
        // NOW the bot processes all collected answers at once
        answers = get_panel_state(query.from.id)
        answer_callback(query.id, "✅ Saved!")
        save_settings(query.from.id, answers)
        edit_message_text(query.message, "✅ Preferences saved.")
        edit_message_reply_markup(query.message, null)
        return

    if field == "other":
        // Free-text escape for the current question
        question_id = value   // reuse value slot for which question
        answer_callback(query.id, "Type your answer below")
        set_awaiting_text(query.from.id, "pnl:" + question_id)
        return

    // --- Silent collection: record answer, show toast, advance ---
    update_panel_state(query.from.id, field, value)
    answer_callback(query.id, value + " ✅")    // brief toast only, no AI response

    // Mark the answered question as done (edit current message)
    edit_message_text(query.message,
        get_question_label(field) + ": " + format_answer(value) + " ✅")
    edit_message_reply_markup(query.message, null)   // remove keyboard from this message

    // Check if all questions answered
    state = get_panel_state(query.from.id)
    if all_questions_answered(state):
        // Send confirmation summary
        send_panel_confirmation(query.message.chat.id, state)
    else:
        // Send the next unanswered question as a new message
        next_step = get_next_unanswered(state)
        send_panel_question(query.message.chat.id, next_step)
```

### Approach B: Single-message compact panel

For the compact single-message variant, taps only update the keyboard visually
(add ✅ prefix to the selected button). No text is sent, no AI response is
triggered. Only "Confirm" triggers processing.

```
handle_panel_compact(query, parts):
    field = parts[1]
    value = parts[2]

    if field == "confirm":
        answers = get_panel_state(query.from.id)
        answer_callback(query.id, "✅ Saved!")
        save_settings(query.from.id, answers)
        edit_message_text(query.message, "✅ Preferences saved.")
        edit_message_reply_markup(query.message, null)
        return

    if field == "cancel":
        // ... same as above
        return

    // Silent collection: just record + re-render keyboard
    update_panel_state(query.from.id, field, value)
    answer_callback(query.id, value + " ✅")    // brief toast, that's it

    // Re-render keyboard with checkmark on selected option
    state = get_panel_state(query.from.id)
    new_keyboard = build_compact_panel_keyboard(state)  // ✅ prefix on chosen buttons
    edit_message_reply_markup(query.message, new_keyboard)
    // Do NOT edit message text. Do NOT send a new message. Do NOT trigger AI.
```

## Wizard Handler (State Machine)

The wizard is the most complex pattern. It manages a linear (or branching) sequence
of steps with back/cancel support.

### State structure

```
wizard_session = {
    step: 1,              // Current step number
    total_steps: 3,       // Total steps in this wizard
    answers: {},          // Accumulated answers: { "name": "Luis", "role": "dev" }
    message_id: null      // The message being edited (set on first send)
}
```

### Handler

```
handle_wizard(query, parts):
    action = parts[1]

    switch action:
        case "cancel":
            answer_callback(query.id, "Wizard cancelled")
            delete_wizard_session(query.from.id)
            edit_message_text(query.message, "❌ Setup cancelled.")
            edit_message_reply_markup(query.message, null)
            return

        case "back":
            target_step = int(parts[2])
            session = get_wizard_session(query.from.id)
            session.step = target_step
            save_wizard_session(query.from.id, session)
            answer_callback(query.id)
            render_wizard_step(query.message, session)
            return

        case "restart":
            session = create_wizard_session(query.from.id, total_steps=3)
            answer_callback(query.id, "Starting over")
            render_wizard_step(query.message, session)
            return

        default:
            // Regular step answer: parts = ["wiz", step, field, value]
            step  = int(parts[1])
            field = parts[2]
            value = parts[3]

            session = get_wizard_session(query.from.id)
            session.answers[field] = value
            session.step = step + 1
            save_wizard_session(query.from.id, session)

            answer_callback(query.id)

            if session.step > session.total_steps:
                // Wizard complete
                finalize_wizard(query.message, session)
            else:
                render_wizard_step(query.message, session)
```

### Rendering a wizard step

```
render_wizard_step(message, session):
    // Build progress header — short, no explanatory text
    header = "🧙 Setup (" + session.step + "/" + session.total_steps + ")\n\n"

    // Show previously answered fields
    for field, value in session.answers:
        header += field_emoji(field) + " " + field + ": " + value + " ✅\n"

    // Add current question — just the question, nothing else
    // NO: "Please select one of the following options:"
    // YES: "Your role?"
    step_config = get_step_config(session.step)
    text = header + "\n" + step_config.question

    // Build keyboard for this step
    keyboard = step_config.build_keyboard(session.step)

    // Add navigation row
    nav_row = []
    if session.step > 1:
        nav_row.append({ text: "◀️ Back", callback_data: "wiz:back:" + (session.step - 1) })
    nav_row.append({ text: "❌ Cancel", callback_data: "wiz:cancel" })
    keyboard.append(nav_row)

    edit_message_text(message, text)
    edit_message_reply_markup(message, { inline_keyboard: keyboard })
```

### Handling free-text input in wizard steps

Some wizard steps need typed input (e.g. "What's your name?"). For these:

1. Send/edit the message with a keyboard that has optional quick-pick buttons
   but also says "or type your answer"
2. Set a flag in the session: `session.awaiting_text = true`
3. In your main message handler (not the callback handler), check if the user
   has an active wizard session with `awaiting_text = true`
4. If yes, treat the text message as the answer, advance the wizard

```
on message(msg):
    session = get_wizard_session(msg.from.id)
    if session and session.awaiting_text:
        field = get_step_config(session.step).field
        session.answers[field] = msg.text
        session.awaiting_text = false
        session.step += 1
        save_wizard_session(msg.from.id, session)

        if session.step > session.total_steps:
            finalize_wizard_via_new_message(msg.chat.id, session)
        else:
            send_wizard_step(msg.chat.id, session)  // New message since we can't edit user's msg
        return

    // ... normal message handling
```

## Action Handler (do: prefix)

Action buttons use the `do:` prefix and represent contextual operations the bot
performs internally — snooze, watch, save, mute, etc. The user taps once and the
bot handles everything behind the scenes.

### Router

```
handle_do_action(query, parts):
    action = parts[1]     // "snz", "watch", "save", "done", "mute", "fwd", "refresh"
    item_id = parts[2]    // identifier for the subject (flight, task, ticker, etc.)
    extra = parts[3..]    // optional additional params (duration, threshold, etc.)

    switch action:
        case "snz"     → handle_snooze(query, item_id, extra)
        case "watch"   → handle_watch(query, item_id, extra)
        case "save"    → handle_save(query, item_id, extra)
        case "done"    → handle_done(query, item_id)
        case "mute"    → handle_mute(query, item_id)
        case "fwd"     → handle_forward(query, item_id)
        case "refresh" → handle_refresh(query, item_id)
        default        → answer_callback(query.id, "Unknown action")
```

### Snooze handler

```
handle_snooze(query, item_id, extra):
    duration = extra[0]    // "15", "60", "1d", "mon", "pick"

    if duration == "pick":
        // Ask user for a custom time — switch to free-text or offer a time picker
        answer_callback(query.id, "When should I remind you? Type a time below.")
        set_awaiting_text(query.from.id, "snooze_time:" + item_id)
        return

    // Parse duration into a concrete delivery time
    snooze_until = compute_snooze_time(duration)

    // Schedule re-delivery
    schedule_notification(query.from.id, item_id, snooze_until)

    answer_callback(query.id, "💤 Snoozed until " + format_time(snooze_until))

    // Update the original message to reflect the snooze
    edit_message_text(query.message,
        query.message.text + "\n\n💤 Snoozed until " + format_time(snooze_until))
    edit_message_reply_markup(query.message, null)
```

### Watch handler

```
handle_watch(query, item_id, extra):
    threshold = extra[0] if extra else null    // optional: "58000" for price alerts

    // Register a monitoring task
    create_watch(query.from.id, item_id, threshold)

    label = "Watching for changes"
    if threshold:
        label = "Alert set for " + threshold

    answer_callback(query.id, "🔔 " + label)

    edit_message_text(query.message,
        query.message.text + "\n\n🔔 " + label)
    edit_message_reply_markup(query.message, null)
```

### Save / Remember handler

```
handle_save(query, item_id, extra):
    // Extract key information from the original message
    info = extract_info(query.message.text)

    // Persist into agent memory / workspace
    save_to_memory(query.from.id, item_id, info)

    answer_callback(query.id, "💾 Saved to memory")

    edit_message_text(query.message,
        query.message.text + "\n\n💾 Saved to records")
    edit_message_reply_markup(query.message, null)
```

### Mark done handler

```
handle_done(query, item_id):
    // Update task/reminder status
    mark_complete(query.from.id, item_id)

    answer_callback(query.id, "✅ Marked as done")

    edit_message_text(query.message,
        query.message.text + "\n\n✅ Done")
    edit_message_reply_markup(query.message, null)
```

### Mute handler

```
handle_mute(query, item_id):
    // Destructive-ish action — confirm first if it's a critical alert category
    if is_critical_alert(item_id):
        // Replace keyboard with a confirmation
        edit_message_reply_markup(query.message, {
            inline_keyboard: [
                [
                    { text: "✅ Yes, mute", callback_data: "do:mute_confirm:" + item_id },
                    { text: "❌ Keep alerts", callback_data: "do:mute_cancel:" + item_id }
                ]
            ]
        })
        answer_callback(query.id)
        return

    suppress_notifications(query.from.id, item_id)
    answer_callback(query.id, "🔇 Muted")
    edit_message_text(query.message,
        query.message.text + "\n\n🔇 Notifications muted")
    edit_message_reply_markup(query.message, null)
```

### Refresh handler

```
handle_refresh(query, item_id):
    answer_callback(query.id, "🔄 Refreshing...")

    // Re-fetch the data
    new_data = fetch_latest(item_id)
    new_text = format_notification(new_data)

    // Rebuild action buttons (same as original)
    new_keyboard = build_action_keyboard(item_id, new_data)

    edit_message_text(query.message, new_text)
    edit_message_reply_markup(query.message, new_keyboard)
```

### Post-action pattern

All action handlers follow the same UX pattern:

1. **answerCallbackQuery** with a short confirmation toast
2. **editMessageText** to append a status line to the original message
3. **editMessageReplyMarkup(null)** to remove the buttons (action taken, no more choices)

The user sees their tap reflected immediately in the message itself — no ambiguity
about whether the action worked.

## Session Storage

For any stateful pattern (multi-select, panel, wizard), you need a way to store
state keyed by `chat_id` or `user_id`. Options from simplest to most robust:

| Approach         | Pros                        | Cons                         |
|------------------|-----------------------------|------------------------------|
| In-memory Map    | Zero setup, fast            | Lost on restart              |
| File (JSON)      | Survives restart            | No concurrency safety        |
| SQLite           | Robust, queryable           | Needs a library              |
| Redis            | Fast, TTL support, shared   | External dependency          |
| Encoded in data  | Stateless server            | 64-byte limit, complex       |

For OpenClaw skills, the simplest approach is to use the agent's workspace files
or memory system to track wizard state per user.

## answerCallbackQuery Quick Reference

Always call this. Options:

| Parameter    | Type    | Description                                    |
|--------------|---------|------------------------------------------------|
| callback_query_id | String | Required. The query ID from the update    |
| text         | String  | Optional. Toast notification (up to 200 chars) |
| show_alert   | Boolean | Optional. Show as modal alert instead of toast |
| cache_time   | Integer | Optional. Seconds to cache result client-side  |

```
// Toast notification (appears briefly at top of chat)
answer_callback(query.id, "Pizza selected ✅")

// Modal alert (user must dismiss)
answer_callback(query.id, "⚠️ This will delete all data!", show_alert=true)

// Silent (just dismiss the spinner)
answer_callback(query.id)
```
