# Changelog

## 0.3.0 — 2026-10-18

- Added an optional, configurable morning heads-up with its own time, title and text before the existing evening reminder.
- The morning notification lists matching waste types and can go to both configured phones without affecting the evening Yes/No state, repeat count, or helper.
- Documented the separate morning notification and its behavior if Home Assistant is offline at the scheduled time.

## 0.2.3 — 2026-10-18

- Changed the default waste-sensor date format to `%d-%m-%Y` and updated the setup documentation.
- Kept the internally stored pickup date in ISO format for scheduling and reply IDs.
- Added a Dutch README, documented confirmation and same-day retesting, and corrected installation examples.

## 0.2.2 — 2026-10-18

- Send and clear notifications for two phones in independent parallel branches, so a failure on one device does not prevent an attempt on the other.
- Validate the persistent text helper's length settings before writing reminder state.
- Stop scheduled processing with a clear error when the helper contains invalid JSON instead of silently replacing its state.
- Clarified that a failed notification consumes one send attempt because the state is checkpointed before delivery.

## 0.2.1 — 2026-10-18

- Added an optional second mobile notify service, configured separately from the first phone.
- Both phones receive the same Yes/No choices; Yes on either device confirms the shared cycle and clears both notifications.
- Supports dual-device delivery in both simple and confirmation modes without requiring a notification group or extra helper.
- Fixed the README import button to point to the actual blueprint file.

## 0.2.0 — 2026-10-18

- Renamed project/blueprint to **Trash Notify** (`trash_notify.yaml`) to match the GitHub repository.
- Replaced blocking `wait_for_trigger` and delays with a minute-based scheduler and mobile reply event triggers.
- Added one persistent `input_text` helper to store the due date, matched waste types, next push time, count, and Yes/No state across Core restarts and automation reloads.
- Added checkpoint writes via `homeassistant.save_persistent_states`.
- Added recovery for the initial reminder if Core was offline at the selected time and starts later the same day.
- Kept simple one-shot mode independent of the helper.
- Added clearer setup instructions and documented caveats.

## 0.1.0 — prototype

- Extended the original date-based garbage reminder with optional actionable notifications and timed repeats.
- Known issue: running reminder loops ended during Home Assistant Core restarts or automation reloads.
