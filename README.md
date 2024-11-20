# Home Assistant Media Player Volume Fade

This script fades the volume of a `target_player` media player, starting at its current `volume_level`, to a user-defined `target_volume` over the user-defined `duration` in seconds. It also applies one of three `curve` algorithms to shape the fade, defaulting to `logarithmic`, which is often considered the most natural sounding fade.

I struggled to find any comprehensive solutions on the Home Assistant forums for fading media player volumes with common attenuation curves. I really don't like abrupt changes to audio volume, so I put this script together.

For those interested, the script is fully commented, and I've put together a quick explanation of the script's working below.

## Input Parameters
- **target_player**: media_player entity
- **target_volume**: float 0 - 1 --- volume to end at
- **duration**: float 0.1 - 60 --- duration of fade in seconds 
- **curve**: selector [logarithmic, bezier, linear]

## Working

The script works by calculating the elapsed time and progress of the fade based on the user-defined `duration`. It determines the difference between the media player's current volume and the user-defined `target_volume`. It applies the difference value as a factor to the shaped fade amount, adds it to the original volume, then pushes the new volume to the media player entity `volume_level` for each step in a `while` loop.

## Algorithms:
Where `x` is the normalized time value based on the `duration`, starting at `0` and ending at `1`.
- (red) Linear: `f(x) = x` 
- (blue) Bezier: `f(x) = x / (1 + (1 - x ))`
- (green) Logarithmic: `f(x) = x * x * (3 - 2x)`

<img src="fade_curves.png" alt="drawing" width="300"/>

## Script code

Add this to your Home Assistant `script` config `.yaml`, and use it anywhere that allows a service call (such as automations):

```yaml
fade_volume:
  alias: Fade the volume of a media player
  mode: restart
  fields:
    target_player:
      name: Target media player
      description: Target media player of volume fade.
      required: true
      example: media_player.lounge_sonos
      selector:
        entity:
          domain: media_player
    target_volume:
      name: Target volume
      description: Volume the media play will be at the end of the fade duration.
      required: true
      default: 0.5
      example: "0.5"
      selector:
        number:
          max: 1
          min: 0
          step: 0.01
          mode: slider
    duration:
      name: Fade duration
      description: Length of time in seconds the fade should take.
      required: true
      default: 5
      example: "5"
      selector:
        number:
          mode: box
          min: 0
          max: 100000
          unit_of_measurement: s
    curve:
      name: Fade curve algorithm
      description: Shape of the fade curve to apply.
      required: true
      default: logarithmic
      example: logarithmic
      selector:
        select:
          options:
          - logarithmic
          - bezier
          - linear
  variables:
    start_volume: "{{ state_attr(target_player, 'volume_level') | float(0) }}"
    start_diff: "{{ target_volume - start_volume }}"
    start_time: "{{ now().timestamp() }}"
  sequence:
    - repeat:
        while:
          - condition: template
            value_template: "{{ (now().timestamp() - start_time) < duration }}"
        sequence:
          - variables:
              elapsed_time: "{{ now().timestamp() - start_time }}"
              progress: "{{ (elapsed_time / duration) | round(4) }}"
          - action: media_player.volume_set
            target:
              entity_id: "{{ target_player }}"
            data:
              volume_level: |-
                {% if curve == 'logarithmic' %}
                  {{ start_volume + (progress / (1 + (1 - progress))) * start_diff }}
                {% elif curve == 'bezier' %}
                  {{ start_volume + (progress * progress * (3 - 2 * progress)) * start_diff }}
                {% else %}
                  {{ start_volume + progress * start_diff }}
                {% endif %}
    # Ensure we reach the target volume
    - action: media_player.volume_set
      target:
        entity_id: "{{ target_player }}"
      data:
        volume_level: "{{ target_volume }}"
  icon: mdi:tune-vertical
```
