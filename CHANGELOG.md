# Changelog

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
