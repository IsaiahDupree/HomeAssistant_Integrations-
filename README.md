# Home Assistant Chromecast Caster (YouTube + SoundCloud)

Cast YouTube videos and SoundCloud tracks to a Google Chromecast from Home Assistant. This repo provides:

- Scripts for YouTube and SoundCloud casting with clear status notifications
- Helpers (text + dropdown) to paste links and choose the target device
- A ready-to-paste Lovelace card
- YAML snippets to include in `configuration.yaml`

Works via:
- YouTube: `media_player.play_media` with the YouTube app
- SoundCloud: `media_extractor.play_media` to resolve the stream

---

## Requirements

- Home Assistant with the Google Cast integration (to discover your Chromecast devices)
- For SoundCloud: enable `media_extractor:` in `configuration.yaml` (see below). `ffmpeg:` is recommended and included in HA OS.

---

## Option 1 — UI-only setup (no file editing)

1) Create Helpers (UI)
- Text helper: Settings → Devices & services → Helpers → Create helper → Text
  - Name: "YouTube URL to Cast"
  - Entity: `input_text.youtube_url_to_cast`
  - Max length: ≤ 255
- Text helper (optional, for SoundCloud): Name: "SoundCloud URL to Cast"
  - Entity: `input_text.soundcloud_url_to_cast`
- Dropdown helper: Create helper → Dropdown
  - Name: "Cast Target"
  - Options: add your Chromecast `media_player.*` entity_ids (e.g., `media_player.living_room_chromecast`)
  - Entity: `input_select.cast_target`

2) Add Scripts (UI)
- Settings → Automations & scenes → Scripts → + Create script → ⋮ → Edit in YAML
- Create these two scripts (paste the body only, no wrapper key at the top):

YouTube script body:
```yaml
alias: Cast YouTube from input
mode: restart
sequence:
  - variables:
      url: "{{ states('input_text.youtube_url_to_cast') | trim }}"
      # Safe extractor: get first 11-char ID from common YouTube URL shapes
      video_id: >-
        {{ (url
            | regex_findall('(?:youtu\\.be/|[?&]v=|/shorts/|/live/|/embed/)([A-Za-z0-9_-]{11})')
            | first
            | default('', true))
           or (url if url|length == 11 else '') }}
      target_entity: "{{ states('input_select.cast_target') }}"
      valid_target: "{{ target_entity is string and target_entity.startswith('media_player.') }}"
      valid_id: "{{ video_id|length == 11 }}"

  - service: persistent_notification.create
    data:
      notification_id: youtube_cast_status
      title: "YouTube Cast: starting…"
      message: >-
        URL: {{ url }}
        Parsed ID: {{ video_id if valid_id else '—' }}
        Target: {{ target_entity if valid_target else '—' }}

  - choose:
      - conditions: "{{ valid_id and valid_target }}"
        sequence:
          - service: media_player.play_media
            target:
              entity_id: "{{ target_entity }}"
            data:
              media_content_type: cast
              media_content_id: >
                {{ {'app_name':'youtube','media_id': video_id} | tojson }}
          - service: persistent_notification.create
            data:
              notification_id: youtube_cast_status
              title: "YouTube Cast: sent ✅"
              message: "Sent {{ video_id }} to {{ state_attr(target_entity, 'friendly_name') or target_entity }}."
          - service: logbook.log
            data:
              name: YouTube Caster
              message: "Casting {{ video_id }} to {{ state_attr(target_entity, 'friendly_name') or target_entity }}"
              entity_id: "{{ target_entity }}"

      - conditions: "{{ not valid_id }}"
        sequence:
          - service: persistent_notification.create
            data:
              notification_id: youtube_cast_status
              title: "YouTube Cast: failed ❌"
              message: >-
                Could not parse a YouTube video ID from:
                {{ url }}
                Tip: paste a normal YouTube link or just the 11-char ID (e.g., dQw4w9WgXcQ).

      - conditions: "{{ not valid_target }}"
        sequence:
          - service: persistent_notification.create
            data:
              notification_id: youtube_cast_status
              title: "YouTube Cast: failed ❌"
              message: >-
                Cast Target "{{ target_entity }}" is not a valid media_player.
                Edit the Cast Target dropdown so each option is a media_player.* entity_id
                (e.g., media_player.living_room_chromecast).
```

SoundCloud script body:
```yaml
alias: Cast SoundCloud from input
mode: restart
sequence:
  - variables:
      url: "{{ states('input_text.soundcloud_url_to_cast') | default(states('input_text.youtube_url_to_cast')) | trim }}"
      target_entity: "{{ states('input_select.cast_target') }}"
      valid_target: "{{ target_entity is string and target_entity.startswith('media_player.') }}"
      valid_sc: "{{ url|lower is search('soundcloud\\.com|on\\.soundcloud\\.com') }}"

  - service: persistent_notification.create
    data:
      notification_id: sc_cast_status
      title: "SoundCloud Cast: starting…"
      message: >-
        URL: {{ url if url else '—' }}
        Target: {{ target_entity if valid_target else '—' }}

  - choose:
      - conditions: "{{ valid_target and valid_sc and url|length > 0 }}"
        sequence:
          - service: media_extractor.play_media
            target:
              entity_id: "{{ target_entity }}"
            data:
              media_content_id: "{{ url }}"
              media_content_type: music
          - service: persistent_notification.create
            data:
              notification_id: sc_cast_status
              title: "SoundCloud Cast: sent ✅"
              message: "Sent to {{ state_attr(target_entity, 'friendly_name') or target_entity }}."
          - service: logbook.log
            data:
              name: SoundCloud Caster
              message: "Casting {{ url }}"
              entity_id: "{{ target_entity }}"

      - conditions: "{{ not valid_sc }}"
        sequence:
          - service: persistent_notification.create
            data:
              notification_id: sc_cast_status
              title: "SoundCloud Cast: failed ❌"
              message: >-
                That doesn't look like a SoundCloud link.
                Paste a SoundCloud track/playlist URL (e.g., https://soundcloud.com/artist/track).

      - conditions: "{{ not valid_target }}"
        sequence:
          - service: persistent_notification.create
            data:
              notification_id: sc_cast_status
              title: "SoundCloud Cast: failed ❌"
              message: >-
                Cast Target "{{ target_entity }}" is not a media_player.* entity_id.
                Fix the dropdown options (e.g., media_player.living_room_chromecast).
```

3) Add Lovelace card (UI)
- Dashboard → Edit → Add card → Entities (or Manual) → paste this YAML:
```yaml
type: entities
title: Cast to TV
entities:
  - entity: input_text.youtube_url_to_cast
    name: Paste YouTube URL or ID
  - entity: input_text.soundcloud_url_to_cast
    name: Paste SoundCloud URL
  - entity: input_select.cast_target
    name: Choose Chromecast
  - entity: script.cast_youtube_from_input
    name: "▶️ Play YouTube"
  - entity: script.cast_soundcloud_from_input
    name: "▶️ Play SoundCloud"
```

---

## Option 2 — YAML files (copy from this repo)

1) Add to `configuration.yaml`:
```yaml
input_text:  !include input_text.yaml
input_select: !include input_select.yaml
script:       !include scripts.yaml
media_extractor:
ffmpeg:
```

2) Copy these files from this repo into your HA config directory (`/config/`):
- `input_text.yaml` → from `homeassistant/helpers/input_text.yaml`
- `input_select.yaml` → from `homeassistant/helpers/input_select.yaml`
- `scripts.yaml` → from `homeassistant/scripts/scripts.yaml`

3) Add the Lovelace card:
- Dashboard → Edit → Add card → Manual → paste the YAML shown above.

4) Developer Tools → YAML → Check configuration → Restart.

---

## Files in this repo

- `homeassistant/helpers/input_text.yaml` — text helpers (YouTube + SoundCloud)
- `homeassistant/helpers/input_select.yaml` — dropdown to choose Chromecast target
- `homeassistant/scripts/scripts.yaml` — both scripts (YouTube + SoundCloud)
- `homeassistant/lovelace/cast_entities_card.yaml` — ready-to-paste card
- `homeassistant/snippets/configuration.yaml` — example includes for your `configuration.yaml`

---

## Troubleshooting

- Ensure your Chromecast device(s) appear under Settings → Devices & services → Google Cast.
- The dropdown options must be real `media_player.*` entity_ids (not friendly names).
- YouTube: paste a standard link or the 11-char ID. The script extracts from `watch?v=`, `youtu.be/`, `/shorts/`, `/live/`, `/embed/` and ignores extra params like `&t=`.
- SoundCloud: enable `media_extractor:`. If short `on.soundcloud.com` links fail, open once and copy the canonical `soundcloud.com/artist/track` URL.
- Always check the Notifications panel; the scripts create a status notification and write to Logbook.
- Use the script TRACE (Scripts → your script → Last run → TRACE) to review variables and the execution path.

---

## License

Choose a license (MIT recommended for simplicity). If you tell me your preferred license, I’ll add the appropriate `LICENSE` file.

---

## Publishing to GitHub

After we finalize the filenames and license, run:
```powershell
# Initialize repo and push (replace <your-username> and <repo-name>)
git init
git add .
git commit -m "Add HA Chromecast casting (YouTube + SoundCloud) with helpers and Lovelace"
git branch -M main
git remote add origin https://github.com/<your-username>/<repo-name>.git
git push -u origin main
```

If you use GitHub CLI:
```powershell
gh repo create <repo-name> --public --source . --remote origin --push
```
