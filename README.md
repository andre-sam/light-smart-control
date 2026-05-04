# Light Smart Control

A Home Assistant **blueprint** that automatically tunes the **color
temperature** (and optionally brightness) of every light in your home
throughout the day, based on the **Area** each light is assigned to in
Home Assistant.

It only ever adjusts lights that are **already on** - it never turns
lights on or off. So it composes cleanly with motion lighting, manual
switches, scenes, and any other automation you already have.

## Why

Cool, bluish light (4000–6500 K) helps you wake up and stay focused.
Warm light (1800–2700 K) signals "wind down" and protects melatonin
production in the evening. The right curve depends on what the room is
*for*: a kitchen wants bright, neutral task light during the day; a
bedroom wants warm and dim from sunset onwards; a bathroom wants cool
crisp light in the morning for grooming and warm at night so it
doesn't blast you awake.

This blueprint encodes those recommendations as **room-type presets**
and applies them on a six-slot daily schedule.

## How room type is decided

You assign each Home Assistant **Area** to a room *type* once, in the
blueprint inputs:

- *Bedroom areas* → `Master Bedroom`, `Kids Bedroom`, …
- *Bathroom areas* → `Main Bathroom`, `Ensuite`, …
- *Living areas* → `Living Room`, `Family Room`, …
- *Kitchen areas* → `Kitchen`, …
- *Dining areas* → `Dining Room`, …
- *Office areas* → `Study`, `Office`, …
- *Hallway areas* → `Hallway`, `Landing`, `Stairs`, …

Every `light.*` placed in those areas is automatically picked up and
tuned with the right curve - including lights you add to those areas
later. **No per-light wiring.**

Lights without color-temperature support are ignored. You can also
exclude specific lights (e.g. RGB scene lamps) via the *Excluded
lights* input.

## Features

- **Area-driven** - one automation tunes every tunable-white light in
  the home; new lights are picked up automatically.
- **Room-type presets** - Bedroom, Bathroom, Living/Family, Kitchen,
  Dining, Office/Study, Hallway. Each comes with sensible Kelvin +
  brightness values for every slot.
- **Six daily anchors** - `pre_dawn`, `morning`, `midday`, `afternoon`,
  `evening`, `night`. Kelvin and brightness are **linearly
  interpolated** between anchors, so changes are smooth rather than
  stepped at each boundary. Overnight wrap is handled.
- **Sun-driven schedule** - all six anchors derive from `sun.sun`:
  `pre_dawn` from `next_dawn`, `morning` from `next_rising`,
  `evening` from `next_setting`, `night` from `next_dusk` (each plus
  a per-anchor offset). `midday` and `afternoon` are anchored
  symmetrically around solar noon (`next_noon ± peak_half_width_hours`,
  default ±2.5h). The whole schedule shifts naturally with the
  seasons and DST.
- **Decoupled Kelvin & brightness scheduling** -
  **Kelvin tracks the raw sun position** (best-practice circadian
  response: warm light arrives with the actual sunset, regardless of
  clock), while **brightness uses configurable clock floors/ceilings**
  on the `evening` and `night` anchors. This solves the high-latitude
  winter problem where Sydney sunset at ~17:00 in June would otherwise
  start dimming the house mid-afternoon - lights still warm up at
  sunset, but stay near full brightness until people are actually
  winding down. Defaults: brightness `evening` clamped to 18:00–20:00,
  brightness `night` clamped to 21:30–23:00.
- **Smooth ramps** - by default, every minute each on-light glides
  toward the next anchor with a 60s transition, so changes are
  imperceptible.
- **Two-layer lighting (downlights + lamps)** - tag each light with an
  HA **Label** (`downlight` or `secondary`). Downlights use a flat
  curve (90–100% all day, 50% at night), so they stay useful for
  task light. Secondary lamps follow the gentler circadian wind-down.
  Both share the same Kelvin curve. Lights without a label fall back
  to a configurable default class.
- **Brightness optional** - only adjust Kelvin if you'd rather control
  brightness elsewhere (e.g. via scenes or motion).
- **Manual-override safe** - skips a light if it was changed in the
  last *N* minutes (default 15) so a manual dim isn't immediately
  overwritten.
- **Hardware-aware** - clamps the target Kelvin to each light's own
  reported `min_color_temp_kelvin` / `max_color_temp_kelvin`, so mixing
  bulbs with different ranges (2000–6666 K, 2202–6535 K, 2900–7000 K,
  …) just works.
- **Flicker-free** - skips updates whose Kelvin delta is below a
  configurable tolerance.
- **Excludes** - exclude specific lights from auto-tuning.

## Recommended values (built-in presets)

Kelvin per anchor, by room. Brightness defaults are also tuned per
anchor. Values *between* anchors are linearly interpolated.

**Sleeping & bathing**

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Bedroom            |     1900 |    3000 |   3500 |      3000 |    2200 |  1800 |
| Bathroom / ensuite |     2200 |    5000 |   5000 |      4000 |    2400 |  1800 |

**Living & dining**

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Living / Family    |     2000 |    3500 |   3500 |      3000 |    2400 |  2000 |
| Dining             |     2200 |    2700 |   2700 |      2700 |    2400 |  2000 |

**Work & utility**

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Kitchen            |     2400 |    4000 |   4500 |      3500 |    2700 |  2000 |
| Office / Study     |     3000 |    4500 |   5000 |      4000 |    3000 |  2200 |
| Laundry            |     2200 |    4000 |   4500 |      4000 |    2700 |  2200 |
| Garage             |     2200 |    4000 |   5000 |      4000 |    3000 |  2200 |

**Circulation & transition**

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Staircase          |     1800 |    3000 |   3500 |      3000 |    2400 |  1800 |
| Entry / Foyer      |     2000 |    3500 |   4000 |      3500 |    2700 |  2000 |

**Outdoor - private**

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Front veranda      |     2000 |    3500 |   4000 |      3500 |    2700 |  2200 |
| Back veranda       |     2000 |    3000 |   3500 |      3000 |    2400 |  2000 |
| Alfresco / Deck    |     2000 |    3000 |   3500 |      3000 |    2400 |  2000 |
| Front yard         |     1800 |    3500 |   4000 |      3500 |    2700 |  2200 |
| Backyard           |     1800 |    3000 |   3500 |      3000 |    2400 |  2000 |
| Pool area          |     2000 |    3500 |   4000 |      3500 |    2700 |  2200 |

**Outdoor - security & utility**

| Room               | pre-dawn | morning | midday | afternoon | evening | night |
|--------------------|---------:|--------:|-------:|----------:|--------:|------:|
| Driveway           |     1800 |    3500 |   4000 |      4000 |    3000 |  2700 |
| Shed / Workshop    |     2200 |    4000 |   5000 |      4500 |    3500 |  2700 |

## Two-layer lighting (downlights + secondary)

If you have **primary downlights** (ceiling/recessed, doing the heavy
lifting) plus **secondary lamps** (table/floor/accent for ambience),
tag each light with an HA **Label**:

- `downlight` - flat brightness curve, stays bright through the day,
  drops only late evening / night. Kelvin still goes warm at night.
- `secondary` - the gentler ambience curve (the per-room values in
  the table above). Kelvin matches the downlights.

Assign labels in HA: **Settings → Areas, Labels & Zones → Labels**,
create `downlight` and `secondary`, then tag each light. New lights
you later tag are picked up automatically.

Lights without either label fall back to *Default fixture class*
(default: `downlight`).

### Downlight brightness curve

| Anchor    | Brightness |
|-----------|-----------:|
| pre_dawn  | 30%        |
| morning   | 90%        |
| midday    | 100%       |
| afternoon | 100%       |
| evening   | 85%        |
| night     | 50%        |

## Requirements

- Lights with color-temperature support (`light.turn_on` with
  `color_temp_kelvin`). Most modern tunable-white smart bulbs qualify
  (e.g. Philips Hue, WiZ, IKEA Tradfri, Shelly RGBW, Tuya CCT), via
  any Home Assistant integration (Zigbee2MQTT, ZHA, Matter, native
  cloud integrations, etc.).
- Each light must be assigned to an **Area** in Home Assistant.
- One `input_boolean` to act as the master enable toggle.
- A **sun entity** (`sun.*`). Home Assistant ships with `sun.sun` out
  of the box once you've configured your home location, so this is
  almost always already satisfied. The blueprint defaults to `sun.sun`
  but lets you pick a different sun entity if you maintain a custom
  one.

## Installation

### Import via My Home Assistant

[![Open your Home Assistant instance and show the blueprint import dialog with a specific blueprint pre-filled.](https://my.home-assistant.io/badges/blueprint_import.svg)](https://my.home-assistant.io/redirect/blueprint_import/?blueprint_url=https%3A%2F%2Fgithub.com%2Fandre-sam%2Flight-smart-control%2Fblob%2Fmain%2Flight-smart-control.yaml)

### Or manually

1. Copy [light-smart-control.yaml](light-smart-control.yaml) into
   `config/blueprints/automation/light-smart-control/` on your Home
   Assistant instance.
2. **Settings → Automations & Scenes → Blueprints** - the blueprint
   should appear as *"Light Smart Control (Adaptive Color
   Temperature)"*.
3. Click **Create Automation** from the blueprint, assign each Area
   to a room type, and save.

## Usage tips

- **Assign Areas first.** Before creating the automation, make sure
  every relevant light is assigned to an Area. New lights you later
  assign to a mapped Area are picked up automatically - no blueprint
  edit needed.
- **One automation is enough.** You only need a single instance for
  the whole house. Re-instantiate only if you want e.g. different
  schedules per zone.
- **Don't fight your dimmer.** If you regularly grab a physical dimmer
  to brighten/dim a light, increase *Skip if recently changed* (e.g.
  to 30 min). Set it to `0` to always re-assert the slot value.
- **Brightness off.** Untick *Also adjust brightness* if you only want
  Kelvin changes. Useful when brightness is already controlled by
  motion or scenes.
- **Excludes.** Add RGB scene lamps or accent lights to the *Excluded
  lights* input so they keep whatever colour you set.

## How it works (brief)

Every minute, the automation:

1. Walks every Area you mapped, collects each Area's lights via
   `area_entities()`, and tags each light with its room type.
2. For each light, finds the two anchors bracketing the current time
   and **linearly interpolates** Kelvin + brightness between them.
3. For each light that's currently `on`:
   - skip if its `last_changed` is within the manual-override window;
   - skip if it doesn't support color temperature;
   - clamp the interpolated Kelvin to the bulb's supported range;
   - skip if the Kelvin delta is below tolerance;
   - call `light.turn_on` with `kelvin` (and optionally
     `brightness_pct`) and a soft transition (60s by default).

That's it - no state machine, no flag entities required.

## License

MIT.
