# GaggiMate Brew Profile Generator ☕

Generate, manage, and deploy espresso brew profiles for [GaggiMate](https://gaggimate.eu/) Pro machines. Built around Lance Hedrick's espresso methodology with support for community profile discovery via the GaggiMate Discord.

## What This Does

1. **Identify a coffee bean** — from a photo of the bag or by name
2. **Auto-generate an optimized GaggiMate Pro profile** — temperature, pressure strategy, ratio, and dose selected based on roast level, origin, processing method, and freshness
3. **Browse community profiles** from the GaggiMate Discord `#profiles` channel
4. **Push profiles directly to your machine** via WebSocket — saved, favorited, and selected in one command

## Quick Start

### Requirements

```bash
pip install websockets aiohttp
```

### Generate a Profile

```bash
python3 scripts/generate-profile.py \
  --label "Ethiopian Yirgacheffe" \
  --roast light \
  --origin ethiopia \
  --process washed \
  --output my-profile.json
```

The generator auto-selects strategy, temperature, ratio, and dose based on the bean characteristics. You can override any parameter:

```bash
python3 scripts/generate-profile.py \
  --label "My Custom Profile" \
  --roast medium \
  --origin colombia \
  --process natural \
  --dose 18 \
  --ratio 2.2 \
  --temp 91 \
  --strategy lever \
  --style espresso \
  --freshness fresh \
  --output custom.json
```

### Push to Your Machine

```bash
# Push (save + favorite + select) in one step
python3 scripts/gaggimate-ws.py push my-profile.json

# Or manage profiles individually
python3 scripts/gaggimate-ws.py list
python3 scripts/gaggimate-ws.py save my-profile.json
python3 scripts/gaggimate-ws.py favorite <profile-id>
python3 scripts/gaggimate-ws.py select <profile-id>
python3 scripts/gaggimate-ws.py delete <profile-id>
python3 scripts/gaggimate-ws.py get <profile-id>
```

By default connects to `gaggimate.local`. Override with:
```bash
export GAGGIMATE_HOST=192.168.1.100
```

### Browse Discord Community Profiles

Search and download profiles shared by the GaggiMate community on Discord:

```bash
# Set your Discord token (user token from browser DevTools)
export DISCORD_TOKEN="your-token-here"

# List recent community profiles
python3 scripts/discord-profiles.py list

# Search by name
python3 scripts/discord-profiles.py search "lever"
python3 scripts/discord-profiles.py search "allonge"

# Get recommendations for your bean
python3 scripts/discord-profiles.py recommend light
python3 scripts/discord-profiles.py recommend dark
python3 scripts/discord-profiles.py recommend medium-light kenya

# Download a specific profile thread
python3 scripts/discord-profiles.py download <thread-id>

# Bulk download
python3 scripts/discord-profiles.py download-all --limit 30
```

## Profile Strategies

The generator includes 6 pressure profile strategies based on Lance Hedrick's methodology:

| Strategy | Best For | Peak Pressure | Shot Time | Description |
|----------|----------|---------------|-----------|-------------|
| **bloom** | Light washed, African origins | 6-7 bar | 25-30s | Fill → bloom (no pump) → gentle ramp → brew |
| **turbo** | Ultra light, processed, aged beans | 2-6 bar | 15-20s | Coarse grind, high flow, fast extraction |
| **lever** | Medium roasts, balanced beans | 8-9 bar declining | 30-40s | Mimics spring lever machines (Cremina-style) |
| **declining** | Medium-dark and dark roasts | 9→5.5 bar | 25-35s | Traditional with controlled pressure decline |
| **flat** | Any (safe starting point) | 9 bar sustained | 25-35s | Classic flat 9 bar |
| **low-contact** | Ultra light geisha-type | 1-3 bar | 15-20s | Very fast flow, minimal contact, "espresso adjacent" |

### Auto-Selection Logic

When `--strategy auto` (default), the generator picks based on:

| Roast | Process | Strategy |
|-------|---------|----------|
| Light | Washed | bloom |
| Light | Natural | turbo |
| Medium-light | Washed (African) | bloom |
| Medium-light | Washed (other) | lever |
| Medium | Any | lever |
| Medium-dark / Dark | Any | declining |
| Any | Anaerobic / Co-ferment | turbo |
| Any (aged >8 weeks) | Any | turbo |

## Parameters Reference

### `generate-profile.py`

| Parameter | Values | Default | Notes |
|-----------|--------|---------|-------|
| `--label` | string | *(required)* | Profile name shown on machine |
| `--roast` | light, medium-light, medium, medium-dark, dark | *(required)* | Drives strategy/temp/ratio |
| `--origin` | ethiopia, colombia, brazil, kenya, sumatra, guatemala, costarica, blend, other | other | Affects strategy for certain roasts |
| `--process` | washed, natural, honey, anaerobic, co-ferment, unknown | washed | Anaerobic/co-ferment → gentle extraction |
| `--dose` | 17-20 | auto | 20g for light, 18g for darker |
| `--ratio` | 1.5-3.5 | auto | Based on Lance's ratio rules |
| `--temp` | 85-96 | auto | Based on roast + process |
| `--strategy` | auto, bloom, turbo, lever, declining, flat, low-contact | auto | Override auto-selection |
| `--style` | espresso, ristretto, lungo, milk, allonge | espresso | Adjusts ratio target |
| `--freshness` | fresh, rested, aged | fresh | Aged beans → turbo strategy |
| `--output` | file path | stdout | Where to write JSON |

### Temperature Defaults (Auto)

| Roast | Temp | Lance's Reasoning |
|-------|------|-------------------|
| Light | 92°C | High enough for sweetness, not so hot it's bitter |
| Medium-light | 91°C | Sweet spot |
| Medium | 90°C | Baseline default |
| Medium-dark | 88°C | Sub-90 for darker roasts |
| Dark | 86°C | Well sub-90 to avoid harshness |
| Anaerobic/Co-ferment | 86-88°C | Low to preserve process flavors |
| Aged | 88°C | Lower temp for depleted CO2 |

### Ratio Defaults (Auto)

| Roast | Ratio | Lance's Reasoning |
|-------|-------|-------------------|
| Light | 1:2.8 | Need more water to push past initial acids |
| Medium-light | 1:2.5 | Balanced extraction |
| Medium | 1:2.2 | Classic range |
| Medium-dark | 1:1.8 | Soluble enough, don't over-extract |
| Dark | 1:1.7 | Very soluble, keep it short |

## Methodology

This tool encodes Lance Hedrick's espresso approach (distilled from [his videos](https://www.youtube.com/@LanceHedrick)):

1. **Pressure is a red herring** — balance matters more than hitting 9 bar. World-class espresso is pulled at 2-4 bar.
2. **Ratio is the #1 extraction lever** — extend ratio before going finer.
3. **Coarser is the default direction** — more even flow through the puck, less channeling.
4. **Temperature: lower than you think** — default 90°C, rarely above 93-94.
5. **Heavily processed coffees need gentleness** — low temp, coarser, moderate ratio.
6. **Aged coffee: accept low pressure** — don't go finer to compensate for lost CO2.
7. **Crema ≠ quality** — a balanced thin shot beats a thick channeled mess.

See [`references/lance-hedrick-methodology.md`](references/lance-hedrick-methodology.md) for the complete framework including his dialing-in decision tree.

## Machine Compatibility

Built and tested on **Rancilio Silvia with GaggiMate Pro**, but the profiles are standard GaggiMate JSON and work on any GaggiMate-equipped machine:

- Gaggia Classic / Classic Pro / Evo
- Rancilio Silvia (all versions)
- Any machine with GaggiMate Pro (pressure transducer + flow profiling)
- GaggiMate Standard (basic profiles only — `type: "standard"`)

### Rancilio Silvia Notes

- Boiler wall temp reads 10-14°C higher than actual puck temperature
- GaggiMate applies a configurable offset (default 5°C for Silvia)
- Profile temperatures = **desired brew water temperature** (offset handled automatically)
- Vibratory pump ~4.5 ml/s flow rate — pressure drops faster than commercial machines
- Thick brass boiler — slow temp changes, no cold water inlet
- Boiler refill plugin recommended (heating element in water)

## Profile JSON Format

Profiles follow the [GaggiMate schema](references/profile-schema.json). A Pro profile looks like:

```json
{
  "id": "uuid-here",
  "label": "My Coffee",
  "type": "pro",
  "description": "Light roast, bloom profile",
  "temperature": 92,
  "phases": [
    {
      "name": "Fill",
      "phase": "preinfusion",
      "valve": 1,
      "duration": 4,
      "temperature": 0,
      "pump": {"target": "flow", "pressure": 2, "flow": 4},
      "transition": {"type": "instant", "duration": 0, "adaptive": 1}
    },
    {
      "name": "Brew",
      "phase": "brew",
      "valve": 1,
      "duration": 35,
      "temperature": 0,
      "pump": {"target": "pressure", "pressure": 7, "flow": 4},
      "transition": {"type": "ease-in", "duration": 4, "adaptive": 1},
      "targets": [
        {"type": "volumetric", "operator": "gte", "value": 56.0}
      ]
    }
  ]
}
```

### Key Fields

- **pump.target**: `"pressure"` or `"flow"` — what the PID controls
- **pump.pressure / pump.flow**: Target values. `-1` = maintain current value at phase start
- **transition.type**: `instant`, `linear`, `ease-in`, `ease-out`, `ease-in-out`
- **transition.adaptive**: `1` = start from current actual value (not previous target)
- **targets**: Stop conditions — `volumetric` (scale weight), `pressure`, `flow`, `pumped` (ml in phase)

## LLM Integration

The `references/barista-persona.md` file contains a system prompt that turns any LLM into a Lance Hedrick-style espresso consultant. Use it for:

- Reasoning about unusual beans or edge cases
- Explaining *why* a specific profile suits a bean
- Iterating based on taste feedback ("it was sour", "bitter finish")
- Photo identification of bean bags

The recommended flow is: **LLM identifies bean → LLM decides parameters → `generate-profile.py` creates valid JSON → push to machine**.

## Examples

See the [`examples/`](examples/) directory for pre-generated profiles:

- `light-washed-bloom.json` — Light Ethiopian, bloom strategy
- `medium-lever.json` — Medium Colombian, lever strategy
- `dark-declining.json` — Dark Italian blend, declining strategy
- `anaerobic-turbo.json` — Anaerobic natural, turbo strategy
- `geisha-allonge.json` — Light Geisha, allongé style

## Community

- [GaggiMate Discord](https://discord.gg/APw7rgPGPf) — `#profiles` channel for shared profiles
- [GaggiMate Docs](https://docs.gaggimate.eu/docs/profiles/) — Official profile documentation
- [GaggiMate Repo](https://github.com/jniebuhr/gaggimate) — Firmware source code

## Credits

- **GaggiMate** by [Jokim Niebuhr](https://github.com/jniebuhr) — the firmware and hardware making this possible
- **Lance Hedrick** — espresso methodology and philosophy ([YouTube](https://www.youtube.com/@LanceHedrick))
- Profile strategies informed by the GaggiMate Discord community

## License

MIT
