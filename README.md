# Central Notifier for Home Assistant

Two blueprints:

- `blueprints/script/central_notifier.yaml`: the single notification script that every automation calls
- `blueprints/automation/alarmo_to_notifier.yaml`: sends Alarmo events through that script

## One-click import
[![Import Central Notifier](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FBanham22%2Fha-central-notifier%2Fblob%2Fmain%2Fblueprints%2Fscript%2Fcentral_notifier.yaml)
[![Import Alarmo link](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2FBanham22%2Fha-central-notifier%2Fblob%2Fmain%2Fblueprints%2Fautomation%2Falarmo_to_notifier.yaml)

Or in HA: **Settings → Automations & scenes → Blueprints → Import blueprint** and paste:
- `https://github.com/Banham22/ha-central-notifier/blob/main/blueprints/script/central_notifier.yaml`
- `https://github.com/Banham22/ha-central-notifier/blob/main/blueprints/automation/alarmo_to_notifier.yaml`

To update later, use **Re-import blueprint** from the blueprint's menu.

## Requirements
- Home Assistant 2024.10 or later
- HA Companion app on each phone (iOS and Android both work)
- **Alexa Media Player** from HACS, only if you use Echo speakers
- Alarmo

## Install
1. Copy the files into your HA config folder, using File Editor, Studio Code Server or Samba:
   - `central_notifier.yaml` → `/config/blueprints/script/andrew/`
   - `alarmo_to_notifier.yaml` → `/config/blueprints/automation/andrew/`
2. Optional helpers. Create these under **Settings → Devices & services → Helpers**:
   - Toggle `Notify silent mode`. When it's on, everything except critical is only logged.
   - Text `Notify cooldown`. Set max length to **255**. This turns on per-tag rate limiting.
3. Create the script:
   1. Go to **Settings → Automations & scenes → Blueprints → Central Notifier → Create script**.
   2. Name it **Notify** (the entity becomes `script.notify`).
   3. Fill in the sections: Phones, Speakers, Behaviour, Confirm.
   4. Save.
4. Create the Alarmo automation:
   1. Go to **Blueprints → Alarmo -> Central Notifier → Create automation**.
   2. Pick the Alarmo panel and `script.notify`.
   3. Save.
5. Open Alarmo → **Actions** and **turn off its built-in notifications**. Otherwise you get every alert twice.

## Test
In **Developer tools → Actions**, switch to YAML mode and run this:
```yaml
action: script.notify
data:
  title: Test
  message: Front door left open
  severity: warning
  channel: security
  confirm: snooze
  escalate_after: 2
```

## Using it in automations
In an automation, add the action **Run script → Notify**. Every field appears as a dropdown or text box.

To wait for the answer, call the script as an action with `response_variable`:
```yaml
- action: script.notify
  data:
    message: Garage still open. Close it?
    severity: warning
    channel: house
    confirm: acknowledge
  response_variable: reply
- if: "{{ reply.status == 'acknowledge' }}"
  then:
    - action: cover.close_cover
      target: {entity_id: cover.garage}
```
To send and carry on without waiting, use `script.turn_on` with `variables:` instead.

`reply.status` returns one of these values:
- `sent`
- `acknowledge`
- `snooze`
- `disarm`
- `silence` (the Silence button was pressed)
- `timeout`
- `escalated`
- `silenced` (silent mode was on)
- `cooldown`

## How it behaves
| Level | iOS | Android | Speakers |
|---|---|---|---|
| Info | active | channel `Info` | if set |
| Warning | time-sensitive | channel `Warnings`, high priority | if set |
| Critical | critical sound, bypasses silent/DND | `alarm_stream` channel | always (Echo DND overridden) |
| Quiet hours (info and warning) | passive (silent) | channel `Quiet`, low priority | none |

- **Phones:** these are resolved as `notify.mobile_app_<device name>`. If you rename the phone in the Companion app, re-register it.
- **Disarm button:**
  - On iPhones it only works when the phone is unlocked. You can turn that requirement off in the blueprint.
  - It can be hidden on Android.
- **Answering:** when one person taps a button, the alert clears from every phone and the logbook records who answered.
- **Snooze:** resends after the snooze time.
- **Escalation:** resends as critical, to the critical phones or to everyone, and always speaks it.
- **Speaker volume:** the script saves each speaker's volume, sets the alert volume, speaks, then restores the old volume. The fallback volume is used if the old one is unknown.

## Known limits
- **Restarts:** if HA restarts while the script is waiting for a reply, that wait is lost.
- **Alexa volume:** the volume Alexa Media reports can be stale, so the restore may not be exact.
- **Alexa DND switch:** the script assumes it's named `switch.<echo>_do_not_disturb_switch`, which is Alexa Media's default naming.
- **Cooldown:** it only applies when the call sets both a `tag` and `cooldown`. Critical alerts always get through.
- **Android channels:** a channel's sound and importance are fixed the first time it's created. Change them later in the phone's Companion app notification settings.
