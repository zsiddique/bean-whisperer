---
name: bean-whisperer
description: Generate espresso brew profiles for GaggiMate Pro on Rancilio Silvia. Use when the user provides a coffee bean (photo or name) and wants a brewing profile created, uploaded, or managed on their GaggiMate machine. Triggers on phrases like "new bean", "brew profile", "coffee profile", "espresso profile", "gaggimate", "dial in", "new coffee", "shot was sour", "shot was bitter", "too acidic", "too thin", "taste feedback", "dial it in". Also handles listing, deleting, or modifying existing profiles on the machine.
metadata:
  openclaw:
    emoji: "☕"
    requires:
      bins: ["python3"]
---

# BeanWhisperer — Espresso Profile Generator

Generate and deploy espresso profiles for GaggiMate Pro on a Rancilio Silvia. Based on Lance Hedrick's espresso methodology.

## Machine Setup
- **Machine**: Rancilio Silvia with GaggiMate Pro (pressure transducer + flow profiling)
- **Host**: `gaggimate.local` — WebSocket at `ws://gaggimate.local/ws` (override with `GAGGIMATE_HOST` env var)
- **Scale**: Bluetooth scale connected (volumetric targets reliable)
- **Baskets**: 18g and 20g available
- **Temp offset**: 5°C configured in GaggiMate (profile temps = desired brew temp)

## Methodology: Lance Hedrick's Approach

Read `references/lance-hedrick-methodology.md` for the full framework. Key principles:
1. **Pressure is a red herring** — balance > 9 bar. 2-4 bar shots win competitions.
2. **Ratio is #1 extraction lever** — extend ratio before going finer.
3. **Coarser is default direction** — more even flow, less channeling.
4. **Temperature: lower than you think** — default 90°C, rarely above 93-94.
5. **Heavily processed = gentle** — low temp, coarser, moderate ratio.
6. **Aged coffee = accept low pressure** — don't go finer to compensate.

## Workflow: New Bean → Profile

### 1. Identify the Bean
**Photo**: Extract bean name, roaster, origin, roast level, processing method, tasting notes.
**Name**: Search web for origin, roast, process, flavor notes.

### 2. Gather Parameters (ask if not provided)
- **Basket size**: 18g or 20g (auto-recommend based on roast)
- **Ratio**: Auto per Lance's rules (light=1:2.8, medium=1:2.2, dark=1:1.7)
- **Style**: espresso, ristretto, lungo, milk drink, allongé
- **Freshness**: fresh (<4wk), rested (4-8wk), aged (>8wk)
- **Base profile**: Use existing machine profile or generate fresh

### 3. Generate Profile (two modes)

**Static mode** (default — fast, deterministic, schema-compliant):
```bash
python3 scripts/generate-profile.py \
  --label "Bean Name" \
  --roast <level> --origin <origin> --process <method> \
  --dose <g> --ratio <ratio> --temp <temp> \
  --strategy <auto|flat|declining|bloom|lever|turbo|low-contact> \
  --style <espresso|ristretto|lungo|milk|allonge> \
  --freshness <fresh|rested|aged> \
  --output /tmp/profile.json
```

**LLM mode** (for edge cases, unusual beans, taste-based iteration):
Read `references/barista-persona.md` for the Lance Hedrick persona system prompt. Use LLM to reason about the bean, decide parameters, then pass to static generator for valid JSON. LLM explains the "why" behind every choice.

### 4. Review with User
Present: strategy, phases, temperature, dose/ratio, expected shot time, expected pressure. Explain WHY this strategy suits their bean using the sour-sweet-bitter framework. Be honest about trade-offs.

### 5. Deploy to Machine
```bash
python3 scripts/gaggimate-ws.py push /tmp/profile.json
```
Saves + favorites + selects in one step. Requires `pip3 install websockets`.

### 6. Post-Shot Iteration (LLM mode)
If user reports taste feedback ("it was sour", "bitter finish", "too thin"):
- Sour → extend ratio 5g, DON'T go finer first
- Bitter/dry → reduce ratio or recommend coarser grind
- Sour + bitter (channeling) → go COARSER (counterintuitive!)
- Thin/watery → slightly finer, or switch from turbo to lever
- Generate adjusted profile and push

## Strategy Selection (Lance's Framework)

| Roast | Process | Freshness | Strategy | Pressure | Time |
|-------|---------|-----------|----------|----------|------|
| Light | Washed | Fresh | Bloom | 6-7 bar | 25-30s |
| Light | Natural | Fresh | Turbo | 2-6 bar | 15-20s |
| Light | Any | Aged | Turbo | 2-4 bar | 15-20s |
| Med-Light | Washed (African) | Fresh | Bloom | 6-7 bar | 25-30s |
| Med-Light | Washed (other) | Fresh | Lever | 8-9→6 bar | 30-40s |
| Medium | Any | Fresh | Lever | 9→6 bar | 30-40s |
| Med-Dark | Any | Fresh | Declining | 9→5.5 bar | 25-35s |
| Dark | Any | Fresh | Declining | 9→5.5 bar | 20-25s |
| Any | Anaerobic/Co-ferment | Any | Turbo | 3-6 bar | 15-20s |

## Profile Management
```bash
python3 scripts/gaggimate-ws.py list              # Show all profiles
python3 scripts/gaggimate-ws.py get <id>           # Export profile JSON
python3 scripts/gaggimate-ws.py save profile.json  # Upload without selecting
python3 scripts/gaggimate-ws.py delete <id>        # Remove a profile
python3 scripts/gaggimate-ws.py push profile.json  # Save + favorite + select
```

## Edge Cases

- **Machine offline**: If `gaggimate-ws.py` fails with a connection error, tell the user to check that GaggiMate is powered on and reachable at the configured host. Suggest saving the profile JSON locally and pushing later.
- **Invalid/unclear bean photo**: If the photo is unreadable or not clearly a coffee bag, ask the user to provide bean details manually (roast, origin, process).
- **No Bluetooth scale connected**: Volumetric stop conditions (`targets.type: "volumetric"`) require a scale. If the user has no scale, switch to time-based stops by removing volumetric targets and relying on phase `duration` values instead.
- **GaggiMate Standard (not Pro)**: If the user has GaggiMate Standard (no pressure transducer), generate `"type": "standard"` profiles with only temperature and time — no pressure/flow phases.

## References
- **Lance Hedrick methodology**: `references/lance-hedrick-methodology.md` — full framework with temp/ratio/pressure rules
- **LLM barista persona**: `references/barista-persona.md` — system prompt for LLM-driven profile generation
- **Espresso knowledge base**: `references/espresso-knowledge.md` — origin/process/strategy details
- **Profile JSON schema**: `references/profile-schema.json`
- **WebSocket API**: `references/websocket-api.md`

## Discord Profile Research

The GaggiMate Discord has a `#profiles` channel where users share JSON profiles (like the Sir Lancelot's Lever profile Lance imported in his video). Use `discord-profiles.py` to search, browse, and download community profiles.

**Requires**: `DISCORD_TOKEN` env var (bot token with access to GaggiMate Discord guild 951416527721230336).

```bash
# Browse recent community profiles
python3 scripts/discord-profiles.py list --limit 30

# Search for profiles matching a style
python3 scripts/discord-profiles.py search "lever"
python3 scripts/discord-profiles.py search "light roast"

# Get AI recommendations based on bean characteristics
python3 scripts/discord-profiles.py recommend light kenya
python3 scripts/discord-profiles.py recommend dark

# Download a specific profile by Discord message ID
python3 scripts/discord-profiles.py download 1380352847387820082

# Bulk download all recent profiles
python3 scripts/discord-profiles.py download-all --limit 50
```

### Discord → Machine Workflow
1. Search/recommend profiles matching the user's bean
2. Download the best candidates
3. Present them with descriptions
4. User picks one → optionally modify (temp, ratio, stop conditions) for their specific bean
5. Push modified profile to machine via `gaggimate-ws.py push`

### When to Use Discord vs Generate Fresh
- **Discord first**: When the bean type is common (dark Italian, lever-style, standard medium)
- **Generate fresh**: When the bean is unusual (rare variety, unique processing, specific ratio needs)
- **Hybrid**: Download a community profile as base → modify temp/ratio/targets for the specific bean
