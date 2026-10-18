# Trash Notify

[English](README.md) · [Nederlands](README.nl.md)

**Trash Notify** is a Home Assistant automation blueprint for waste-collection reminders with optional **Yes / No** mobile actions and configurable repeat notifications.

A **Core restart or automation reload does not reset an active reminder**: the next reminder time, collection date, reminder count and confirmation are stored in one persistent `input_text` helper. A minute-based trigger resumes the cycle after startup. No extra waste sensors, template sensors, AppDaemon scripts, or custom integrations are needed.

> **Version:** 0.3.0 — candidate for testing on Home Assistant OS 2026.10 or later. One No → Yes cycle has been observed on a real installation. Core restart recovery and two-device delivery still need live verification before labeling this stable.
>
> **Important:** enabling Yes/No requires **one `input_text` helper per Trash Notify automation**. Simple one-shot mode doesn't require the helper.

## Blueprint

[Import blueprint into Home Assistant](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fdkwolf1%2FTrash-notify%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Ftrash_notify.yaml) · [View YAML](blueprints/automation/trash_notify.yaml)

## Features

- Supports up to four waste sensors; their **state** must equal a collection date (default format `%d-%m-%Y`).
- Configure an optional morning heads-up and the evening reminder separately. The morning message has its own time, title and text, and has no Yes/No buttons. Enable it under **Ochtendwaarschuwing** and choose a time before the evening reminder.
- Configure the evening reminder time and number of days before pickup (`1` = tomorrow). The date format applies to sensor states; an existing automation with a saved date-format input keeps its selected value until you change it.
- Optional actionable iOS/Android notifications: **Yes** confirms the bins are out and clears the notification; **No** postpones the next reminder from the time you tap it.
- No response: the next reminder arrives after the configured interval (e.g. 10, 60 or 120 minutes), up to a configurable maximum.
- Supports **two independently configured Companion App phones** without creating a notification group (second phone optional). A mobile notification group remains possible. **Yes on either phone** confirms the cycle and clears both notifications.
- The latest reminder is replaced using a stable notification tag.
- Resumes after Core restart, blueprint reload and automation reload (within approximately a minute of the next scheduled check).
- If Core was offline at the original reminder time, the cycle can start later the **same calendar day**.
- Stale replies from a different pickup date are ignored, using collection-date-specific action IDs.
- No extra sensor or template entities required.

## Installation

1. Click **Import blueprint into Home Assistant** above, or go to **Settings → Automations & scenes → Blueprints → Import Blueprint** and paste the [blueprint URL](https://github.com/dkwolf1/Trash-notify/blob/main/blueprints/automation/trash_notify.yaml). For manual installation, download the YAML and place it at `/config/blueprints/automation/dkwolf1/trash_notify.yaml`.
2. For **Yes/No mode**, create a Text helper in **Settings → Devices & services → Helpers → Create helper → Text**:
   - Name: `Trash Notify State`
   - Entity ID (suggested): `input_text.trash_notify_state`
   - **Minimum length: 0 characters**.
   - **Maximum length: 255 characters** (the stored JSON is longer than the default limit of 100).
   - Leave any validation pattern empty.
   - Do **not** specify an `initial` value if configuring it via YAML, so the previous state is restored automatically.
3. Alternatively create the helper in `configuration.yaml`:

   ```yaml
   input_text:
     trash_notify_state:
       name: Trash Notify State
       min: 0
       max: 255
       # Important: no `initial:` property
   ```

   Add it to your existing `input_text:` block if one already exists, rather than creating a duplicate top-level key. Reload the Input Text integration or restart Home Assistant if needed.
4. Import the blueprint or make an automation based on it. Choose the waste sensors, evening reminder time and **Notificatieapparaat 1** (`notify.mobile_app_*`). Optionally configure **Notificatieapparaat 2** with a different mobile notify service (no group necessary).
5. Optionally enable **Stuur een waarschuwing in de ochtend** under **Ochtendwaarschuwing**. Set the morning time, title and text. The morning message lists the matching waste types and needs no extra helper.
6. Enable **Vraag om Ja/Nee-bevestiging**, select the `input_text` entity under **Statusopslag voor bevestigingen**, and set the repeat interval and notification maximum.
7. Disable your old reminder automation before activating the new one to prevent duplicate pushes.

After repository updates, re-import the blueprint in Home Assistant and reload automations. Check the date-format input on existing automations: an explicitly saved value does not change when the blueprint default changes.

For multiple Trash Notify automations, create **one separate `input_text` helper per automation** to prevent their state and button IDs from colliding.

## Morning and evening notifications

With the default one-day offset, the morning notification warns that the matched bins must go out **that evening** for collection the next day. It uses the same sensor date match as the evening reminder. Both messages go to the configured notify services; the morning message has no actions and is sent once at its configured time. It does not change the persistent helper, the evening send count, or the evening Yes/No cycle. An evening **Yes** confirms that collection cycle, including if a morning warning was sent earlier.

The morning warning is a separate notification and can remain visible after the evening notification is cleared. If Home Assistant is offline at the morning time, that warning is missed; the existing evening confirmation cycle can still start later that day. For same-day retesting, adjusting only the evening time does not reset a confirmed cycle; see below.

## How restart-safe reminders work

- The blueprint stores compact JSON in the helper, for example:

  ```json
  {"day":"2026-10-18","due":"2026-10-19","mask":1,"status":"pending","sent":1,"next":1792339200}
  ```

- The `day` field identifies the reminder day, `due` the pickup day, `mask` the selected waste sensors, `status` stores `pending` or `confirmed`, `sent` counts send attempts, and `next` stores the Unix timestamp at which the next push is due. The stored `day` and `due` values use ISO dates even when the sensors use `%d-%m-%Y`.
- Each `input_text.set_value` write is followed by `homeassistant.save_persistent_states` to checkpoint the state.
- Instead of a long-running `delay` / `wait_for_trigger`, each minute or on startup the automation checks whether a pending reminder is due. After a restart it therefore uses the **previously stored reminder time**, rather than starting another full interval.
- A **Yes** reply is stored as `confirmed`, so restarting Core does **not** restart that day's notifications. The notification is also cleared by its tag.
- A **No** reply resets `next` to the reply time plus the configured interval; no reply keeps the countdown based on the last push.
- After the maximum number of sends, no more automatic reminders are pushed for that pickup. The last notification retains its **Yes** action when still present.

### Example

At 18:00 you receive a push. You tap **No** at 18:05 and choose a 60-minute interval. The next push is scheduled for **19:05**. If Core restarts at 18:30, the stored `next` timestamp is retained and the notification is still sent around 19:05. If you tap **Yes** at 18:35, the confirmation is saved and there are no more pushes for that collection.

## After Yes and the next pickup

Tapping **Yes** keeps the helper on `confirmed` for that reminder day. Changing only the automation time on the same day does not start another notification. The helper is not cleared automatically: otherwise the minute-based check could start the same pickup again.

On a later day, once the configured time has passed and a sensor state matches the next target pickup date, the blueprint replaces the old state with a new `pending` cycle. If no sensor matches, the old `confirmed` state may remain visible; it does not block a future matching pickup.

To repeat a test **on the same day**, run this once in **Developer Tools → Actions**:

```yaml
action: input_text.set_value
target:
  entity_id: input_text.trash_notify_state
data:
  value: "{}"
```

Then make sure the sensor date matches the configured offset and date format. If the reminder time has already passed, the next minute check can start the cycle; otherwise it starts at the configured time. Use your own helper entity ID if it differs from the example.

## Notification action / mobile limitations

The Yes/No buttons require the official Home Assistant Companion App on iOS or Android. Use a `notify.mobile_app_*` service or a mobile notification group. Using `notify.notify` only works if its recipients actually support actionable notifications.

Both devices receive the same action IDs and notification tag. Tapping **Yes** on either device saves one shared confirmation and sends `clear_notification` to both devices. Tapping **No** on either device postpones the shared reminder cycle. Delivery and notification clearance depend on mobile OS notification permissions and Companion App behavior.

## Current limitations

- A **single `input_text` entity stores one active collection cycle**. If pickups on consecutive days overlap before the earlier reminder is confirmed, a new daily reminder can supersede the earlier cycle. Use separate automations/helpers if you need independent overlapping cycles.
- Grouping several waste types for the same pickup day results in a **single Yes/No** decision for all those bins.
- State restoration relies on a valid, available `input_text` helper with `max: 255`; when missing, confirmation mode deliberately stops with an error instead of silently losing its state.
- The helper must also have `min: 0` and no validation pattern. Invalid or incomplete JSON in the helper stops scheduled processing with an error so the saved cycle is not silently replaced.
- Checks run at one-minute resolution. Following a restart or automated time check, a reminder can arrive up to approximately one minute after the due time (plus notification delivery delay).
- With unexpected power loss at the exact moment a notification is sent, it is impossible to guarantee exactly-once delivery: state is saved **before** sending to favor avoiding duplicate notifications. A send interrupted by a crash, or a failed notification service, consumes one send attempt; the next attempt occurs at the configured interval if the maximum has not been reached.
- Notifications and notification clearing run in independent parallel branches for the two phones. If one service fails, Home Assistant still attempts the other.
- Reminder cycles stop sending after the end of the collection day, even if the maximum number has not been reached.
- Manually resetting the `input_text` state to `{}` can start a new cycle on the same day if the time and sensor date still match. Do this only when intentionally repeating a test.
- The first one-shot notification (confirmation disabled) retains the original trigger-at-time behavior. It is not replayed following a missed trigger.

## Credits

Inspired by the original [Garbage reminder Home Assistant Community blueprint](https://community.home-assistant.io/t/garbage-reminder/284213). This project extends the original date-based matching idea with reply persistence and automatic retries.
