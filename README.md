# Light Smart Control

A Home Assistant **blueprint** that automatically tunes the **color
temperature** (and optionally brightness) of your lights throughout the
day, based on the **type of room** they live in.

It only ever adjusts lights that are **already on** — it never turns
lights on or off. So it composes cleanly with motion lighting, manual
switches, scenes, and any other automation you already have.

## Why

Cool, bluish light (4000–6500 K) helps you wake up and stay focused.
Warm light (1800–2700 K) signals "wind down" and protects melatonin
production in the evening. The right curve depends on what the room is
*for*: a kitchen wants bright, neutral task light during the day; a
bedroom wants warm and dim from sunset onwards; a bathroom wants cool
crisp light in the morning for grooming and warm light at night so it
doesn't blast you awake.

This blueprint encodes those recommendations as **room-type presets**
and applies them on a six-slot daily schedule.

## Features

- **Room-type presets** — Bedroom, Bathroom, Living/Family room,
  Kitchen, Dining, Office/Study, Hallway. Each comes with sensible
  Kelvin + brightness values for every slot of the day.
- **Six daily slots** — `pre_dawn`, `morning`, `midday`, `afternoon`,
  `evening`, `night`, with configurable transition times. Overnight
  wrap is handled.
- **Custom mode** — pick `custom` and drive every Kelvin and brightness
  value yourself.
- **Brightness optional** — only adjust Kelvin if you'd rather control
  brightness elsewhere (e.g. via scenes or motion).
- **Manual-override safe** — skips a light if it was changed in the
  last *N* minutes (default 15) so a manual dim isn't immediately
  overwritten.
- **Hardware-aware** — clamps the target Kelvin to each light's
  reported `min_color_temp_kelvin` / `max_color_temp_kelvin`.
- **Flicker-free** — skips updates whose Kelvin delta is below a
  configurable tolerance.
- **Multi-entity** — runs the same logic over a list of lights.
  Instantiate the blueprint once per room (or per group of rooms with
  the same role) so each gets its own preset and schedule.

## Recommended values (built-in presets)

Kelvin per slot, by room. Brightness defaults are also tuned per slot.

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Bedroom            | 1900     | 3000    | 3500   | 3000      | 2200    | 1800  |
| Bathroom           | 2200     | 5000    | 5000   | 4000      | 2400    | 1800  |
| Living / Family    | 2000     | 3500    | 3500   | 3000      | 2400    | 2000  |
| Kitchen            | 2400     | 4000    | 4500   | 3500      | 2700    | 2000  |
| Dining             | 2200     | 2700    | 2700   | 2700      | 2400    | 2000  |
| Office / Study     | 3000     | 4500    | 5000   | 4000      | 3000    | 2200  |
| Hallway / Landing  | 1800     | 3000    | 3500   | 3000      | 2400    | 1800  |

Pick `Custom` from the room-type selector to override every value.

## Requirements

- Lights that support color temperature (`light.turn_on` with
  `kelvin`). Most modern smart bulbs (Hue, Wiz, IKEA Tradfri tunable
  white, Shelly RGBW, Tuya CCT, Zigbee2MQTT-paired tunable bulbs, etc.)
  qualify.
- One `input_boolean` to act as the master enable toggle.

## Installation

### Import via My Home Assistant

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fandre-sam%2Flight-smart-control%2Fblob%2Fmain%2Flight-smart-control.yaml)

### Or manually

1. Copy [light-smart-control.yaml](light-smart-control.yaml) into
   `config/blueprints/automation/light-smart-control/` on your Home
   Assistant instance.
2. **Settings → Automations & Scenes → Blueprints** — the blueprint
   should appear as *"Light Smart Control (Adaptive Color
   Temperature)"*.
3. Click **Create Automation** from the blueprint and fill in the
   inputs.

## Usage tips

- **One instance per room.** Different room types want different
  curves, so create one automation per room (or per group of rooms
  sharing a role). Each gets its own light list, room-type preset and
  schedule.
- **Schedule defaults are conservative.** Tweak `morning_start` to
  match local sunrise and `evening_start` to match local sunset for
  the most natural feel. (You can also set them seasonally via
  helpers and edit the blueprint to use them — out of scope here.)
- **Don't fight your dimmer.** If you regularly grab a physical dimmer
  to brighten/dim a light, set *Skip if recently changed* to ~30 min
  so the automation respects you. Set it to `0` to always re-assert
  the slot value.
- **Bulbs with narrow Kelvin range.** The blueprint clamps to each
  bulb's reported range, so an 2700–6500 K bulb will simply sit at
  2700 K during the warmest slots — it won't error.
- **Brightness off.** Untick *Also adjust brightness* if you only want
  Kelvin changes. Useful when brightness is already controlled by
  motion or scenes.

## How it works (brief)

Every 5 minutes, on each slot boundary, and whenever a light turns
on, the automation:

1. Works out the current slot from the clock.
2. Looks up the target Kelvin + brightness (preset or custom).
3. For each light in the list that's currently `on`:
   - skip if its `last_changed` is within the manual-override window;
   - skip if the Kelvin delta is below tolerance;
   - clamp to the bulb's supported range;
   - call `light.turn_on` with `kelvin` (and optionally
     `brightness_pct`) and a soft transition.

That's it — no state machine, no flag entities required.

## License

MIT.
