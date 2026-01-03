---
layout: post
title: "Building an OSRS-Inspired 3D Engine with Claude Code Over Christmas Break"
date: 2026-01-03
categories: [gamedev, ai]
---

Over the Christmas holidays, I set out to explore what it would be like to build a game with Claude Code as my primary development partner. The result: **3DRS**, a first-person 3D engine inspired by Old School RuneScape, built from scratch in C++ with Raylib. In 12 days and 101 commits, we went from a simple grass shader to a feature-rich game with procedural textures, dynamic lighting, a full quest system, and an autonomous AI agent that can test the game without human intervention.

![Daytime overview of Lumbridge](/assets/3drs/hero.png)
*Procedural terrain, trees, brick buildings, wooden bridge, and the HUD with minimap and inventory.*

The final numbers: ~19,000 lines of C++, 20 procedural shaders, 4 map regions (Lumbridge, Varrock, Al Kharid, Wilderness), 3,100+ trees, 350+ enemies, and systems for combat, skills, quests, inventory, and banking.

## Development Timeline

**Day 1 (Dec 23)** laid the groundwork: first-person movement, a procedural grass shader, combat with trolls, map file loading, enemy AI, and sound effects.

**Day 2 (Dec 24)** added terrain height with procedural noise, animated water shaders, multiple biomes, spatial hashing for O(1) proximity queries, a running/energy system, and a full day/night cycle with shadows.

**Day 3 (Dec 25)** brought bloom and SSAO post-processing, procedural fire for campfires, winter mode with snow particles, a banking system, and mining.

**Day 4 (Dec 26)** expanded with seasons (spring/summer/autumn/winter), falling leaf particles, frustum culling for the massive Wilderness region (3000 trees, 600 rocks, 300 enemies), ranged combat with bow and arrow, and blood splatter particles.

**Days 5-12** focused on refinement: LLM-powered monster generation, the scripted input system for automated testing, the validation subagent, procedural tree and brick shaders with bump mapping, and instanced grass rendering.

## Procedural Everything

Early on I decided: no image textures. Every visual is generated in fragment shaders. Terrain uses noise-based color variation for grass and sand. Water has multi-octave animated noise with sparkle highlights. Walls (brick, stone, wood) use bump mapping for depth. Fire is animated procedural flames. The sky is a time-of-day gradient. Foliage uses SDF-based leaf shapes with procedural veins. The advantage is infinite variation—no two bricks look the same, grass has subtle color differences, and water never tiles.

![Autumn foliage with campfire](/assets/3drs/blog_autumn_campfire.png)
*Autumn mode with procedurally-colored foliage and a crackling campfire.*

## Lighting and Seasons

![Night scene with point lights from street lamps](/assets/3drs/blog_night_lamps_20260103_164245.png)
*Street lamps turn on automatically at dusk.*

The lighting system simulates a 20-minute day/night cycle. Directional sunlight moves across the sky, casting dynamic shadows via shadow mapping with PCF soft edges. Point lights handle street lamps (which turn on at dusk) and campfires (always on). Exponential fog color-matches the sky, and post-processing adds bloom and SSAO.

![Winter mode with snow particles](/assets/3drs/blog_winter_snow_new.png)
*Winter transforms the world with snow coverage, falling snowflakes, and evergreen trees.*

Seasons affect terrain shaders (snow in winter, vibrant greens in spring), tree colors (green in summer, orange/red in autumn, evergreen in winter), and particle systems (snow in winter, falling leaves in autumn).

## The Quest System

Quests are data-driven text files, not C++ code:

```
quest pest_control
name Pest Control
npc guard

objective chitin
objective chitin

reward_gil 100
reward_quest_points 1

dialogue_start
The bugs are out of control!
Please bring me two pieces of chitin.
.

dialogue_complete
The village is safe. Thank you!
.
```

This made it trivial to add quests without recompiling—Claude could write quest definitions and test them immediately.

![Quest accept dialogue](/assets/3drs/quest_accept.png)
*Talking to the Guard NPC to accept a quest.*

![Quest completion](/assets/3drs/quest_complete.png)
*Quest dialogue updates as you progress.*

## The Autonomous Validation Agent

This is where things got interesting. I created a validation subagent—a specialized Claude Code agent that can test game features without human intervention. It has access to a scripted input system that simulates keypresses, mouse clicks, and player teleportation. It can take screenshots and analyze them with vision.

When I implement a feature, I invoke the agent with a description like "Verify the Random button in the time menu changes the lighting." The agent reads the source files, generates a test script, runs the game headless, captures screenshots, analyzes them, and returns a PASS/FAIL report.

The scripting DSL supports commands like:

```
warp <x> <y> <z>              # teleport player
face <yaw> <pitch>            # set camera direction
press <KEY>                   # press key for 1 frame
hold <KEY> <frames>           # hold key
click <x> <y>                 # mouse click
set_time <0.0-1.0>            # set time of day
set_season <season>           # change season
screenshot <label>            # capture screenshot
```

Traditional game testing requires a human to launch the game, navigate to the feature, perform actions, visually verify, and report. The validation agent automates this entire loop. When it implements a feature, it can verify it works without human involvement. For features that need real-time feel (combat timing, audio sync), it falls back to `MANUAL_TEST_REQUIRED` with instructions.

Because headless mode doesn't need a display, you can run multiple instances in parallel. The screenshots for this blog post were captured by spawning 6 game instances simultaneously, each with a different script (daytime, winter, autumn, night, water, dawn), all rendering and saving screenshots at once.

## The Map System

Maps use a text format with include directives:

```
include lumbridge.map 0 0
include varrock.map 0 -200
include alkharid.map 100 50
```

Each region file has entity placements: `wall`, `tree`, `npc`, `enemy`, `water`. This made world-building collaborative—Claude could add content and immediately test with `./build/game --test`, which validates loading without opening a window.

![Al Kharid desert region](/assets/3drs/blog_alkharid_desert_20260103_164316.png)
*Al Kharid: sand terrain, brick walls, scorpion enemies.*

## Technical Notes

The rendering pipeline is multi-pass: shadow pass (depth-only to 2048x2048 shadow map), main pass (scene with lighting to off-screen texture), post-processing (bloom extraction/blur, SSAO), and composite (combine everything to screen).

Performance relies on frustum culling (Gribb/Hartmann plane extraction with sphere tests), spatial hashing (O(1) proximity queries), instanced rendering (19,600 grass blades in one draw call), and chunk streaming (grass loaded around player position).

Player progress saves to JSON with automatic migration. When I changed quest progress from index-based to ID-based, Claude wrote a migration script to convert existing saves.

## Voice and LLM Help

NPC dialogue is spoken aloud using Piper TTS, a local neural text-to-speech engine. Each NPC type gets a voice (deep male for guards, neutral for friendly NPCs). The models run locally (~60MB each), synthesizing in real-time without API calls.

Pressing `H` opens a help dialog where you can ask Claude questions about your current quest. The system prevents spoilers—it only sends Claude info about objectives you've completed or are currently working on, never future ones. The prompt includes strict rules: only help with current objectives, don't reveal future objectives or rewards, keep responses to 2-3 sentences, stay in-character. The API call runs on a background thread so the game doesn't freeze.

![LLM quest help](/assets/3drs/llm_help.png)
*Claude provides context-aware hints without spoiling future objectives.*

## Water and Environment

![Water bridge scene](/assets/3drs/blog_water_bridge_20260103_164250.png)
*Animated water with procedural ripples, crossed by a wooden bridge.*

The water shader uses multi-octave simplex noise for waves, animated scrolling patterns, sparkle highlights that catch sunlight, and color variation based on depth.

![Trees and grass with campfire](/assets/3drs/blog_features_grass_trees_20260103_164315.png)
*Procedural trees, grass blades swaying in the wind, and animated fire.*

## Lessons

<!-- TODO: Fill this section with your personal reflections -->

[Your observations about working with Claude Code—what worked, what was challenging, how it changed your workflow]

[Your thoughts on the validation agent—was it worth the investment? How did it change your confidence in making changes?]

[Technical takeaways from procedural generation, shader development, C++ patterns]

[What you'd do differently—architecture decisions, feature prioritization, testing strategies]

## Conclusion

Building 3DRS was an experiment in AI-assisted game development. Claude wasn't just autocompleting code—it was designing systems, writing shaders, creating content (quests, monsters, maps), testing its own work via the validation agent, and migrating data formats.

The validation agent represents something new: AI that can verify its own work visually. When Claude implements a feature and then tests it by running the game and looking at screenshots, we're getting closer to software development with minimal human intervention.

---

*Screenshots captured via automated parallel headless runs—the same system the validation agent uses.*
