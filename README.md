# Frient KEPZB-110 keypad ↔ Alarmo (Zigbee2MQTT)

A Home Assistant blueprint that turns the Frient / Develco **KEPZB-110** keypad into a
proper terminal for **Alarmo**, over **Zigbee2MQTT**.

The keypad decides nothing. Alarmo is the single source of truth for alarm state; the
keypad sends requests and displays what it is told. That separation is what makes the
system behave like a commercial panel instead of a pile of automations.

[![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/lucasem75/frient-kepzb110-alarmo/main/frient_keypad_alarmo.yaml)

Running in production on a real installation: SLZB-06U coordinator, Zigbee2MQTT,
Mosquitto, Alarmo, KEPZB-110 keypad.

---

## Why another one

Several blueprints already exist for this keypad, nearly all descending from
[AndrejDelany's 2021 work](https://community.home-assistant.io/t/zigbee2mqtt-sync-keypad-and-with-alarm-control-panel-states/345311).
This one exists because none of them covered a **duress code**, and because the ones I
tested carried bugs I needed fixed for my own installation.

### Fixed compared to the upstream blueprint

| Issue | Effect | Fix |
|---|---|---|
| Numeric codes compared across types | Home Assistant's template engine casts `"1900"` back to the integer `1900`, so it never equalled the string it was compared against. **Duress and special codes never fired.** Only RFID tags, which start with `+`, kept working | `\| string \| trim` on both sides of every code comparison |
| `armed_home` state trigger missing | Arming in Home mode never reached the keypad; it kept showing the previous state | Trigger added |
| PIN list split without `trim` | `1234; 5678` produced `" 5678"`; that code never matched | `map('trim')` on the list |
| Special codes compared with `in` | Substring match: typing `99` fired the action bound to `9911` | Strict equality |
| Invalid-code feedback gated on a numeric test | Unknown RFID tags were silently ignored | Feedback covers tags too |
| IAS ACE transaction acknowledged only on failure | Keypad waited for a reply that never came on valid commands, then timed out | Arm Response sent on every valid command |

The first row is worth knowing about if you fork anything in this family: the bug is
invisible in the upstream layout because its delegation path compares no codes at all,
and RFID tags mask it everywhere else.

### Added

- **Duress code** — disarms the panel for real, returns keypad feedback that is
  indistinguishable from a normal disarm, then runs a silent action of your choice.
- **Delegation mode** — forward the entered code straight to Alarmo so it stays the
  only code authority: per-user PINs, `changed_by`, lockout after N failed attempts.
- **Restart resync** — pushes the current panel state back to the keypad after a Home
  Assistant restart, instead of leaving it frozen on a stale display.
- **Reinforced error feedback** — three orange flashes, optional buzzer.

---

## Requirements

- Home Assistant **2024.10.0** or later
- [Alarmo](https://github.com/nielsfaber/alarmo) installed via HACS and configured
- Zigbee2MQTT with the KEPZB-110 paired, and an MQTT broker
- The keypad's state topic and its `/set` counterpart

A ZHA variant is **not** provided. If you are on ZHA, look at
[HG943/Frient-Alarm-Sync](https://github.com/HG943/Frient-Alarm-Sync), which solves the
problem at the quirk level rather than working around it.

---

## Installation

1. Import the blueprint with the button above, or drop the `.yaml` into
   `config/blueprints/automation/<your_folder>/` and reload blueprints from
   **Developer tools → YAML**.
2. Create an automation from it.
3. Fill in the inputs below.

Get Alarmo working on its own first — arm and disarm from its Lovelace card, with your
sensors assigned to each mode. If arming fails there, the keypad will not fix it, and
you will spend your time debugging the wrong layer.

---

## Configuration

### Keypad

| Input | Example |
|---|---|
| State topic | `zigbee2mqtt/keypad` |
| Set topic | `zigbee2mqtt/keypad/set` |
| Accepted PIN codes and RFID tags | `1234; 5678; +ACF5678B` |

Codes are separated by semicolons; surrounding spaces are tolerated. RFID tags are
declared with a leading `+`. **Leave this field empty in delegation mode** — it is
ignored there.

### Panel

| Input | Notes |
|---|---|
| Panel entity | Your Alarmo entity. It is named after your Alarmo **area**, so it is often `alarm_control_panel.<area>` rather than `alarm_control_panel.alarmo` |
| Code sent to the panel | A code Alarmo accepts. **Required even in delegation mode** — the duress branch uses it to disarm |
| Delegate code validation to Alarmo | See below. Recommended: **on** |

For *Code sent to the panel*, create a dedicated Alarmo user — call it `System` — with a
long random code nobody ever types, and grant it **disarm only**. A duress disarm then
shows up under that name in `changed_by`, giving you a trace distinct from your normal
disarms. Do not enable Alarmo's *security code* option on it: despite the name, that is
the force-arm flag and has nothing to do with duress.

### Delegation mode

**On (recommended)** — the entered code goes to Alarmo untouched. You get per-user
codes, `changed_by`, and Alarmo's failed-attempt lockout. Nothing sensitive lives in the
automation config. Orange feedback is preserved, but it is inferred: the blueprint waits
a grace period and checks the state the panel actually reached. Two consequences:

- If Alarmo refuses to arm because a sensor is open, the keypad shows *invalid code*.
  The diagnosis is wrong, but "it didn't work" is still true.
- On a loaded host, two seconds may be too short and valid codes will flash orange.
  Raise the grace period under **Advanced**.

**Off** — the blueprint checks the entered code against its own list, then sends a
single shared code to Alarmo. Instant, accurate orange feedback on a bad code, but
Alarmo never sees a failure: no lockout, and no idea who disarmed. Inherited from the
upstream blueprint; keep it only if you are not using Alarmo's user management.

---

## Duress code

The keypad must behave exactly as it does for a normal disarm. No distinct beep, no
different LED, no delay — nothing an attacker standing next to you could notice.

Set a **Duress code** that exists **nowhere in Alarmo** — it is not a user, and the
blueprint alone recognises it, before anything is forwarded. Then put your silent action
in **Silent action**:

```yaml
- variables:
    stamp: "{{ now().strftime('%Y%m%d_%H%M%S') }}"
    shot: "/media/duress/{{ now().strftime('%Y%m%d_%H%M%S') }}.jpg"

- action: camera.snapshot
  continue_on_error: true
  target:
    entity_id: camera.front_door_sub
  data:
    filename: "{{ shot }}"

- delay:
    seconds: 3

- parallel:
    - action: telegram_bot.send_photo
      continue_on_error: true
      data:
        file: "{{ shot }}"
        caption: "⚠️ DURESS — forced disarm at {{ stamp }}"
    - action: notify.mobile_app_your_phone
      continue_on_error: true
      data:
        title: "⚠️ DURESS"
        message: "Duress disarm"
        data:
          push:
            sound:
              critical: 1
              volume: 1.0
          ttl: 0
          priority: high
```

Put **nothing audible or visible** in that action. No siren, no light, no TTS.

Practical notes from running this for real:

- Freeze the timestamp in a variable at the top, as above. Each `now()` is evaluated
  when it runs, so if the second ticks over between the snapshot and the send, the
  upload points at a file that does not exist.
- `camera.snapshot` returns before the file is written, and it does not create the
  directory. Create it first, allow it in `allowlist_external_dirs`, and leave a couple
  of seconds before sending.
- Once the sequence grows past a few actions, move it into a **script** and call that
  instead. You can then test the whole alert chain from *Developer tools → Actions*
  without arming the alarm, and reuse it for the SOS button.
- Test it once for real, siren unplugged, then remember it exists. A duress code nobody
  remembers is worthless.
- Keep the duress code well away from your normal ones. `1900` next to `1901` is an
  error waiting to happen under stress.

---

## Code priority

First match wins:

```
SOS  >  duress  >  special codes 1/2/3  >  normal codes  >  invalid code
```

A code is either an Alarmo user or a blueprint code — never both. Special codes and the
duress code are evaluated before the normal path, so declaring one as an Alarmo user
creates a second route with different behaviour.

---

## Advanced

| Input | Default | Notes |
|---|---|---|
| Keypad buzzer on invalid code | off | **Leave it off on a KEPZB-110.** See limitations |
| Grace period (delegation mode) | 2 s | Time given to Alarmo before judging whether the command succeeded |
| Restart resync delay | 45 s | Time given to Z2M and the broker to come up before pushing state to the keypad |

---

## Known limitations

- **Not a certified alarm.** No radio jamming detection, no monitoring centre, no
  backed-up link. Your insurer will not recognise it.
- **No buzzer on this keypad.** Zigbee2MQTT documents only `arm_mode` for the
  [KEYZB-110](https://www.zigbee2mqtt.io/devices/KEYZB-110.html); `squawk` and `warning`
  belong to the Develco sirens, not the keypad. The buzzer option exists for compatible
  hardware — on a KEPZB-110 it only fills the Z2M log with errors.
- **No confirmation beep on arm or disarm.** Sound comes from the keypad firmware in
  response to the panel status it receives, and the steady states are silent. Only the
  entry and exit countdowns beep.
- **Countdown duration comes from the keypad, not from Alarmo.** The blueprint sends no
  duration, so the keypad counts for its internal default. Keep your Alarmo delays
  aligned with it or you will get silence at the end of the countdown.
- **No `not_ready` feedback** when Alarmo refuses to arm because a sensor is open. The
  IAS ACE status exists and the keypad supports it; wiring it up means hooking Alarmo's
  failure event, which is not done yet.
- `armed_vacation` and `armed_custom_bypass` are not mapped to keypad modes — the keypad
  only has three physical arm buttons.

---

## Credits

- [AndrejDelany](https://community.home-assistant.io/t/zigbee2mqtt-sync-keypad-and-with-alarm-control-panel-states/345311)
  — the original keypad/panel sync blueprint that everything here descends from.
- [Bygood91/frient_keypad_alarmo](https://github.com/Bygood91/frient_keypad_alarmo)
  — the direct ancestor of this fork.
- [michaeln64](https://github.com/michaeln64/KEPZB-110-BluePrint-Z2M) — IAS ACE
  transaction acknowledgement.
- [teodesign](https://github.com/teodesign/Blueprint-Home-Assistant-Alarmo-Keypad-Frient-KEPZB-110-and-Zigbee2Mqtt)
  — the repeated invalid-code feedback pattern.
- [nielsfaber/alarmo](https://github.com/nielsfaber/alarmo) — the alarm engine this is
  built around.

## License

MIT. See [LICENSE](LICENSE).