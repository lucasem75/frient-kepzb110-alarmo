# Frient KEPZB-110 keypad ↔ Alarmo (Zigbee2MQTT)

A Home Assistant blueprint that turns the Frient / Develco **KEPZB-110** keypad into a
proper terminal for **Alarmo**, over **Zigbee2MQTT**.

The keypad decides nothing. Alarmo is the single source of truth for alarm state; the
keypad sends requests and displays what it is told. That separation is what makes the
system behave like a commercial panel instead of a pile of automations.

> [![Import blueprint](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https://raw.githubusercontent.com/YOUR_USER/YOUR_REPO/main/frient_keypad_alarmo.yaml)

---

## Why another one

There are already several blueprints for this keypad, nearly all descending from
[AndrejDelany's 2021 work](https://community.home-assistant.io/t/zigbee2mqtt-sync-keypad-and-with-alarm-control-panel-states/345311).
This one exists because none of them covered a **duress code**, and because the ones I
tested carried bugs I needed fixed for my own installation.

### Fixed compared to the upstream blueprint

| Issue | Effect | Fix |
|---|---|---|
| `armed_home` state trigger missing | Arming in Home mode never reached the keypad; it kept showing the previous state | Trigger added |
| PIN list split without `trim` | `1234; 5678` produced `" 5678"`; that code never matched | `map('trim')` on the list |
| Special codes compared with `in` | Substring match: typing `99` fired the action bound to `9911` | Strict equality |
| Invalid-code feedback gated on a numeric test | Unknown RFID tags were silently ignored | Feedback covers tags too |
| IAS ACE transaction acknowledged only on failure | Keypad waited for a reply that never came on valid commands, then timed out | Arm Response sent on every valid command |

### Added

- **Duress code** — disarms the panel for real, returns keypad feedback that is
  byte-for-byte identical to a normal disarm, then runs a silent action of your choice.
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
problem properly at the quirk level.

---

## Installation

1. Import the blueprint with the button above, or drop the `.yaml` into
   `config/blueprints/automation/<your_folder>/` and reload blueprints from
   **Developer tools → YAML**.
2. Create an automation from it.
3. Fill in the inputs below.

---

## Configuration

### Keypad

| Input | Example |
|---|---|
| State topic | `zigbee2mqtt/Entrance_Keypad` |
| Set topic | `zigbee2mqtt/Entrance_Keypad/set` |
| Accepted PIN codes and RFID tags | `1234; 5678; +ACF5678B` |

Codes are separated by semicolons; surrounding spaces are tolerated. RFID tags are
declared with a leading `+`. This field is ignored in delegation mode.

### Panel

| Input | Notes |
|---|---|
| Panel entity | Your Alarmo entity, e.g. `alarm_control_panel.alarmo` |
| Code sent to the panel | A code that Alarmo accepts. **Required even in delegation mode** — it is what the duress branch uses to disarm |
| Delegate code validation to Alarmo | See below |

### Delegation mode

**Off** — the blueprint checks the entered code against its own list, then sends a
single shared code to Alarmo. Instant orange feedback on a bad code, but Alarmo never
sees a failure: no lockout, no idea who disarmed.

**On** — the entered code goes to Alarmo untouched. You get per-user codes,
`changed_by`, and Alarmo's failed-attempt lockout. Orange feedback is preserved, but it
is now inferred: the blueprint waits a short grace period and checks the state the panel
actually reached. Two consequences worth knowing:

- If Alarmo refuses to arm because a sensor is open, the keypad shows *invalid code*.
  The diagnosis is wrong, but "it didn't work" is still true.
- On a loaded host, two seconds may be too short and valid codes will flash orange.
  Raise the grace period under **Advanced**.

---

## Duress code

The keypad must behave exactly as it does for a normal disarm. No distinct beep, no
different LED, no delay — nothing an attacker standing next to you could notice.

Set a **Duress code** that is **not** in the accepted-codes list, then put your silent
action in **Silent action**. Example:

```yaml
- action: camera.snapshot
  target:
    entity_id: camera.front_door_sub
  data:
    filename: "/media/duress/{{ now().strftime('%Y%m%d_%H%M%S') }}.jpg"
- parallel:
    - action: notify.telegram_alerts
      data:
        title: "⚠️ DURESS"
        message: "Duress disarm at {{ now().strftime('%H:%M:%S') }}"
    - action: notify.mobile_app_your_phone
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

Two things to be aware of:

- Test it once for real, siren unplugged, then remember it exists. A duress code nobody
  remembers is worthless.
- Special codes are evaluated before the normal ones. A code that appears in both lists
  will only ever arm or disarm, so keep the lists disjoint, and avoid codes that are
  near-variants of each other.

---

## Code priority

First match wins:

```
SOS  >  duress  >  special codes 1/2/3  >  normal codes  >  invalid code
```

---

## Advanced

| Input | Default | Notes |
|---|---|---|
| Keypad buzzer on invalid code | off | Only enable if the **Exposes** tab in Z2M actually shows a `warning` property. Some converter versions expose `arm_mode` only, and the command will just fill your Z2M log with errors |
| Grace period (delegation mode) | 2 s | Time given to Alarmo before judging whether the command succeeded |
| Restart resync delay | 45 s | Time given to Z2M and the broker to come up before pushing state to the keypad |

---

## Known limitations

- **Not a certified alarm.** No radio jamming detection, no monitoring centre, no backed-up
  link. Your insurer will not recognise it.
- Exit and entry countdown beeping is produced by the keypad's own firmware, from the
  panel status it receives. The blueprint does not send a duration, so the keypad counts
  for its internal default. Keep your Alarmo delays aligned with it or you will get
  silence at the end of the countdown.
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