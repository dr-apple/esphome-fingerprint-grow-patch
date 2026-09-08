# Helpers required by the automations/dashboard

Create these in Home Assistant under **Settings → Devices & Services → Helpers**
(or via the `input_text`/`input_boolean` config-flow API) before importing the
automations and dashboard view. One pair per enrolled finger slot (5 shown here,
matching the 5 enroll buttons in the ESPHome config — add more pairs if you
enroll more fingers).

| Entity ID | Type | Purpose | Default |
|---|---|---|---|
| `input_text.fingerabdruck_finger_1_person` | Text | Name of the person finger slot 1 belongs to | empty |
| `input_boolean.fingerabdruck_finger_1_aufschliessen_aktiv` | Toggle | Whether finger slot 1 is allowed to unlock the door | off |
| `input_text.fingerabdruck_finger_2_person` | Text | " | empty |
| `input_boolean.fingerabdruck_finger_2_aufschliessen_aktiv` | Toggle | " | off |
| `input_text.fingerabdruck_finger_3_person` | Text | " | empty |
| `input_boolean.fingerabdruck_finger_3_aufschliessen_aktiv` | Toggle | " | off |
| `input_text.fingerabdruck_finger_4_person` | Text | " | empty |
| `input_boolean.fingerabdruck_finger_4_aufschliessen_aktiv` | Toggle | " | off |
| `input_text.fingerabdruck_finger_5_person` | Text | " | empty |
| `input_boolean.fingerabdruck_finger_5_aufschliessen_aktiv` | Toggle | " | off |

The unlock automation looks these up dynamically from the `finger_id` in the
MQTT payload (`input_text.fingerabdruck_finger_<N>_person`,
`input_boolean.fingerabdruck_finger_<N>_aufschliessen_aktiv`), so the entity
IDs must match this exact pattern — a template helper created with a
different name won't be found.

**Don't forget to actually turn the toggle on** for each finger you enroll —
they default to `off`, so a freshly enrolled finger won't unlock the door
until you flip its switch on the dashboard.
