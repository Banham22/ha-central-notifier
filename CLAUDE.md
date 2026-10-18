# Home Assistant notification rules

All notifications in this Home Assistant go through **one script: `script.notify`**, created from the *Central Notifier* blueprint. Follow these rules whenever you create or edit an automation or script.

## Hard rules
- **Never** call `notify.mobile_app_*`, `notify.alexa_media*`, `tts.*`, `media_player.volume_set` or `persistent_notification.create` directly to alert a person. Always use `script.notify`.
- **Never** put phone names, speaker names or volumes in an automation. Those live in the script's blueprint settings.
- When editing an existing automation that notifies directly, **replace** that action with `script.notify`. Keep the same message wording unless asked to change it.
- Don't edit the Central Notifier script or its blueprint unless explicitly asked.
- Alarmo alerts are already handled by the *Alarmo -> Central Notifier* automation. Don't add Alarmo notifications anywhere else. There's one automation per Alarmo area, e.g. house and garage.

## Fields
| Field | Required | Values | Notes |
|---|---|---|---|
| `message` | yes | text | Short and specific. Include the entity or area. |
| `title` | no | text | Defaults to the channel name. |
| `severity` | yes | `info` / `warning` / `critical` | See the guide below. |
| `channel` | yes | `security` / `house` / `energy` / `fire` / `system` | Decides the route. `fire` is always sent as critical. |
| `confirm` | no | `none` / `acknowledge` / `snooze` / `disarm` / `silence` | Adds buttons. Default `none`. |
| `escalate_after` | no | minutes (0 = never) | Only used with `confirm`. Resends as critical if unanswered. |
| `escalate_to` | no | `critical` / `everyone` | Default `critical`. |
| `speak` | no | `auto` / `always` / `never` | Default `auto`: speaks only if speakers are set for that severity and it isn't quiet hours. |
| `tag` | no | text | Use for anything that can repeat. A new alert with the same tag replaces the old one. Use a short, stable, snake_case tag, e.g. `garage_door`. |
| `cooldown` | no | seconds | Drops repeats with the same `tag` within this time. Needs `tag`. Critical alerts ignore it. |
| `image` | no | URL | e.g. `/api/camera_proxy/camera.front_door` |
| `link` | no | URL | Opened on tap, e.g. `/lovelace/security` |
| `target_phones` | no | mobile_app device IDs | Only when the alert is for one specific person. Avoid otherwise. |
| `alarm_entity` | no | alarm_control_panel entity | Which alarm the Disarm button disarms. Always set it when `confirm: disarm` is used and there's more than one alarm, e.g. house and garage. |
| `silence_entity` | no | input_boolean / switch / siren | What the Silence button controls. Defaults to the one set in the blueprint. |
| `internal` | - | - | Reserved for the script. Never set it. |

## Severity guide
- **info**: nice to know, no action needed (laundry done, someone arrived home, a backup finished). Silent in quiet hours.
- **warning**: needs action soon (door left open, low battery, freezer warming, a service down). Silent push in quiet hours.
- **critical**: needs action now, or it's a safety issue (water leak, smoke, CO, the alarm triggered). Bypasses do-not-disturb and quiet hours, and speaks on the speakers. Use it sparingly.

## Channel guide
- **security**: alarm, doors and locks, cameras, intrusion
- **house**: appliances, climate, doors left open, people coming and going
- **fire**: smoke, heat and CO alarms. Always use `confirm: silence` and an `escalate_after`.
- **energy**: heat pump, UPS, solar and grid, tariffs
- **system**: HA itself, add-ons, updates, network, backups, devices offline

## Calling patterns
**Fire and forget**. This is the default and doesn't block the automation:
```yaml
- action: script.turn_on
  target:
    entity_id: script.notify
  data:
    variables:
      title: Laundry
      message: Washing machine finished
      severity: info
      channel: house
      tag: laundry
```

**Simple direct call.** The automation waits only if `confirm` is set:
```yaml
- action: script.notify
  data:
    message: Front door has been open for 10 minutes
    severity: warning
    channel: house
    tag: front_door_open
    cooldown: 600
```

**Wait for a reply and act on it:**
```yaml
- action: script.notify
  data:
    title: Garage
    message: Garage door still open. Acknowledge to close it.
    severity: warning
    channel: house
    confirm: acknowledge
    escalate_after: 10
    tag: garage_door
  response_variable: reply
- if:
    - condition: template
      value_template: "{{ reply.status == 'acknowledge' }}"
  then:
    - action: cover.close_cover
      target:
        entity_id: cover.garage_door
```

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

**Clearing an alert when the condition resolves.** Send to the same tag with a short info message, or leave it if it doesn't matter.

## Converting an old automation
1. Find every direct notify, TTS, Alexa or persistent-notification action that alerts a person.
2. Replace them all with one `script.notify` action. Don't use a separate call per phone or speaker.
3. Pick the `severity` and `channel` from the guides above.
4. Add a `tag` if the trigger can fire repeatedly, and a `cooldown` if it could spam.
5. Remove any volume set/restore, delays or Alexa do-not-disturb handling that only existed for the old notification. The script now handles these.
6. Keep everything else in the automation unchanged.
7. In your reply, briefly list what you changed and the severity and channel you chose.

## Fire example
```yaml
- action: script.notify
  data:
    title: Fire
    message: Smoke detected in the kitchen
    channel: fire
    confirm: silence
    escalate_after: 2
    escalate_to: everyone
    tag: fire_kitchen
```

## Before and after example
```yaml
# Before
- action: notify.mobile_app_andrew_iphone
  data:
    message: Water leak under kitchen sink!
- action: media_player.volume_set
  target: {entity_id: media_player.kitchen_echo}
  data: {volume_level: 0.9}
- action: notify.alexa_media_kitchen_echo
  data:
    message: Water leak detected in the kitchen
    data: {type: announce}

# After
- action: script.notify
  data:
    title: Water leak
    message: Water leak detected under the kitchen sink
    severity: critical
    channel: house
    confirm: acknowledge
    escalate_after: 5
    tag: leak_kitchen
```
