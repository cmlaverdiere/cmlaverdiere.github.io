---
layout: post
title: "Building an OSRS-Inspired 3D Engine with Claude Code Over Christmas Break"
date: 2026-01-03
categories: [gamedev, ai]
---

Over the Christmas break, I set out to explore what it would be like to build a game with Claude Code as my primary development partner. The result: **3DRS**, a first-person 3D engine inspired by Old School RuneScape (OSRS), built from scratch in C++ with Raylib. In 12 days and 101 commits, we went from a simple grass shader to a feature-rich game with procedural textures, dynamic lighting, a full quest system, and even an autonomous AI agent that can test the game without human intervention.

![Daytime overview showing procedural grass, trees, and the HUD](/assets/3drs/blog_daytime_overview_20260103_164233.png)
*The game features procedural terrain, trees, grass blades, and a full HUD with minimap and inventory.*

## The Scope of What We Built

Here are the final numbers:

- **101 commits** over 12 days (Dec 23 - Jan 3)
- **~19,000 lines** of C++ code
- **20 procedural shaders** (no texture images used)
- **4 map regions**: Lumbridge, Varrock, Al Kharid, and the Wilderness
- **3,100+ trees**, 600+ rocks, 350+ enemies, 11 NPCs
- **Multiple game systems**: combat, skills, quests, inventory, banking, shops

## Development Timeline

### Day 1 (Dec 23): Foundation
The first commit laid the groundwork: first-person movement with a procedural grass shader. By the end of Day 1, I had:
- Combat system with trolls and XP rewards
- Map file loading for walls and structures
- Enemy AI that attacks back
- Sound effects and inventory clicking

### Day 2 (Dec 24): World Building
- Terrain height system with procedural noise
- Water rendering with animated shaders
- Multiple regions (Varrock, Al Kharid) with different biomes
- Spatial hashing for O(1) proximity queries
- Running/energy system
- Full day/night lighting cycle with sun and shadows

### Day 3 (Dec 25): Polish & Systems
- Bloom and SSAO post-processing effects
- Procedural fire shader for campfires
- Winter mode with snow particles
- Banking system with 48-slot storage
- Mining skill with ore rocks

### Day 4 (Dec 26): Content Expansion
- Seasonal system (spring/summer/autumn/winter) with particle effects
- Autumn leaf particles when chopping trees
- Frustum culling for the Wilderness (3000 trees, 600 rocks, 300 enemies)
- Bow and arrow ranged combat
- Blood splatter particle system
- Item stacking

### Days 5-12: Refinement & Automation
- LLM-powered monster generation via egg incubation
- Scripted input system for automated testing
- The validation subagent (more on this below)
- Procedural tree and brick shaders with bump mapping
- Instanced grass rendering for performance

## Procedural Everything

One design decision I made early: **no image textures**. Every visual in the game is generated procedurally in fragment shaders. This constraint led to some interesting shader work:

![Autumn foliage with campfire](/assets/3drs/blog_autumn_campfire.png)
*Autumn mode with procedurally-colored foliage in reds, oranges, and golds, and a crackling campfire.*

### The Shader Collection

- **Terrain**: Grass and sand with noise-based color variation
- **Water**: Multi-octave animated noise with sparkle highlights
- **Walls**: Brick, stone, and wood with bump mapping for depth
- **Fire**: Animated procedural flames using noise functions
- **Sky**: Gradient based on time of day with sun/moon positioning
- **Foliage**: SDF-based leaf shapes with procedural vein patterns
- **Blood**: Stretched droplet particles with specular highlights

The advantage of procedural textures is infinite variation - no two bricks look exactly the same, grass has subtle color differences, and water never tiles.

## The Day/Night Cycle

![Night scene with point lights from street lamps](/assets/3drs/blog_night_lamps_20260103_164245.png)
*Night time: street lamps turn on automatically at dusk, illuminating the village.*

The lighting system simulates a 20-minute day/night cycle with:

- **Directional sunlight** that moves across the sky
- **Dynamic shadows** via shadow mapping with PCF soft edges
- **Point lights** for street lamps (turn on at dusk) and campfires (always on)
- **Exponential fog** that color-matches the sky
- **Post-processing**: bloom for bright highlights, SSAO for ambient depth

## Seasonal Weather

![Winter mode with snow particles](/assets/3drs/blog_winter_snow_new.png)
*Winter mode transforms the world with snow coverage, falling snowflakes, and evergreen trees.*

The season system affects:

- **Terrain shaders**: Snow coverage in winter, vibrant greens in spring
- **Tree colors**: Green (summer), orange/red (autumn), bare/evergreen (winter)
- **Particle systems**: Snow (winter), falling leaves (autumn)
- **Background music**: Season-specific procedural ambient generation

## The Quest System

Quests are data-driven, defined in text files rather than code:

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

dialogue_stage_1
Still waiting on that chitin...
.

dialogue_complete
The village is safe. Thank you!
.
```

This made it trivial to add new quests without touching C++ code - Claude could write quest definitions directly and test them immediately.

## The Autonomous Validation Agent

This is where things get interesting from an AI perspective. I created a **validation subagent** - a specialized Claude Code agent that can autonomously test game features without human intervention.

### How It Works

The validation agent has access to:
1. A **scripted input system** that can simulate keypresses, mouse clicks, and player warping
2. The ability to **take screenshots** at any point
3. **Vision capabilities** to analyze those screenshots

When I implement a new feature, I can invoke the validation agent with a high-level description:

```
Task(subagent_type="validate",
     prompt="Verify the Random button in the time menu works -
             it should appear in the menu and clicking it should
             visibly change the time of day (lighting/sky color).")
```

The agent then:
1. Reads the relevant source files to understand the feature
2. Generates a test script with the DSL commands
3. Runs the game headless with scripted input
4. Captures screenshots at key moments
5. Analyzes the screenshots to verify correctness
6. Returns a PASS/FAIL report

### The Scripting DSL

The scripted input system supports commands like:

```
# Player positioning
warp <x> <y> <z>              # Teleport player to position
face <yaw> <pitch>            # Set camera direction
look_at <x> <y> <z>           # Point camera at world position

# Input simulation
press <KEY>                   # Press key for 1 frame
hold <KEY> <frames>           # Hold key for N frames
click <x> <y>                 # Mouse click at screen position

# State manipulation
set_time <0.0-1.0>            # Set time of day
set_season <spring|summer|autumn|winter>
give_item <ITEM_NAME>         # Add item to inventory

# Capture
screenshot <label>            # Save labeled screenshot
```

A typical test script looks like:

```
# Test: Time Menu Random Button
set_season summer
set_time 0.4
wait 30

screenshot baseline

# Open time menu
press T
wait 30
screenshot menu_open

# Click Random button (5th button)
click 760 500
wait 60
screenshot after_random
```

### Why This Matters

Traditional game testing requires a human to:
1. Launch the game
2. Navigate to the feature
3. Perform the test actions
4. Visually verify the results
5. Report findings

With the validation agent, this entire loop is automated. When Claude implements a new feature, it can immediately verify it works correctly without any human involvement. This is a significant step toward **fully autonomous software development**.

The agent can also fall back gracefully - for features that require real-time feel (combat timing, audio sync), it returns `MANUAL_TEST_REQUIRED` with clear instructions for human testing.

### Sample Test Output

```
## Test: Time Menu Random Button
**Status:** PASS

### Scenarios Tested:
1. Menu Opens Correctly - PASS
   - Screenshot: menu_open.png
   - Expected: Time menu visible with 5 buttons
   - Observed: Menu displayed with Dawn/Day/Dusk/Night/Random buttons

2. Random Button Changes Time - PASS
   - Screenshot: after_random.png
   - Expected: Sky color/lighting different from baseline
   - Observed: Time changed from 0.4 to 0.72, sky shifted to dusk colors

### Summary:
Random button functions correctly, changing time of day to a random value.
```

## The Map System

Maps use a simple text format with include directives for modular regions:

```
# world.map
include lumbridge.map 0 0
include varrock.map 0 -200
include alkharid.map 100 50
include wilderness.map -50 0
```

Each region file contains entity placements:

```
# lumbridge.map
player_spawn 0 0 10

# Buildings
wall 8 0 -12 4 8 4 brick
wall -8 0 -12 4 8 4 brick

# NPCs
npc guard 10 0 15
npc banker 5 0 -220

# Environment
tree -12 0 0
tree -13 0 2 oak
water 0 0 20 10 30

# Enemies
enemy troll 20 0 30
enemy scorpion 110 0 70
```

This made world building collaborative - Claude could add content to maps and immediately test with `./build/game --test` (which validates loading without opening a window).

![Al Kharid desert region with sand terrain and brick walls](/assets/3drs/blog_alkharid_desert_20260103_164316.png)
*Al Kharid: A desert region with sand terrain, brick walls, and scorpion enemies.*

## Technical Highlights

### Rendering Pipeline

The engine uses a multi-pass deferred-style pipeline:

1. **Shadow Pass**: Depth-only rendering to 2048x2048 shadow map
2. **Main Pass**: Render scene with lighting and shadows to off-screen texture
3. **Post-Processing**: Bloom extraction, blur, and SSAO calculation
4. **Composite**: Combine scene, bloom, and AO; output to screen

### Performance Optimizations

- **Frustum culling**: Gribb/Hartmann plane extraction with sphere tests
- **Spatial hashing**: O(1) proximity queries for collision detection
- **Instanced rendering**: 19,600 grass blades in a single draw call
- **Chunk streaming**: Grass chunks loaded around player position

### Save System

Player progress saves to JSON with automatic migration support. When I changed quest progress from index-based to ID-based (for stability when adding new quests), Claude automatically wrote a migration script to convert existing saves.

## The Voice System

NPC dialogue is spoken aloud using Piper TTS - a local neural text-to-speech engine. Each NPC type gets assigned a voice (deep male for guards, neutral for friendly NPCs). The voice models run entirely locally (~60MB each), so dialogue is synthesized in real-time without API calls.

## In-Game LLM Quest Help

One of the more experimental features: pressing `H` opens a help dialog where you can ask Claude questions about your current quest. The system is designed to **prevent spoilers** - it only tells Claude about objectives you've already completed or are currently working on, never future ones.

### How It Works

When you open the help UI:

1. The game identifies your active quest
2. It builds a context prompt containing:
   - Quest name and description
   - Completed objectives
   - Current objective only
   - Relevant items in your inventory
3. Your question is sent to Claude's API with strict instructions:
   - Only help with current/completed objectives
   - Never reveal future objectives or rewards
   - Keep responses to 2-3 sentences
   - Stay in-character as a friendly game guide

### The Spoiler-Prevention Prompt

```cpp
"IMPORTANT RULES:\n"
"1. ONLY help with the current objective or previously completed objectives\n"
"2. DO NOT reveal or hint at any future objectives\n"
"3. DO NOT spoil quest rewards\n"
"4. Keep responses concise (2-3 sentences)\n"
"5. Stay in-character as a friendly game guide\n"
```

### Why This is Interesting

Traditional game hint systems are static - they show the same hints regardless of context. With an LLM, hints can be:

- **Context-aware**: Claude knows what items you have and can tailor advice
- **Conversational**: You can ask follow-up questions in natural language
- **Dynamic**: The same question gets different answers based on your progress

The API call happens on a background thread so the game doesn't freeze while waiting for a response. The UI shows an animated "Waiting for response..." with a cancel option.

## Water and Environment

![Water bridge scene](/assets/3drs/blog_water_bridge_20260103_164250.png)
*Animated water with procedural ripples and reflections, crossed by a wooden bridge.*

The water shader features:
- Multi-octave simplex noise for wave simulation
- Animated scrolling patterns
- Sparkle highlights that catch the sunlight
- Color variation based on depth

![Trees and grass with campfire](/assets/3drs/blog_features_grass_trees_20260103_164315.png)
*Procedural trees, individual grass blades swaying in the wind, and a campfire with animated flames.*

## What I Learned

<!-- TODO: Fill this section with your personal reflections -->

### Working with Claude Code

[Your observations about the development experience with Claude Code - what worked well, what was challenging, how it changed your workflow]

### On Autonomous Testing

[Your thoughts on the validation agent approach - was it worth the investment? How did it change your confidence in making changes?]

### Technical Takeaways

[Any insights from the technical implementation - procedural generation, shader development, C++ patterns, etc.]

### What I'd Do Differently

[Hindsight observations - architecture decisions, feature prioritization, testing strategies]

## What's Next?

The codebase is open for continued development. Some areas that could be expanded:

- **More quests** using the data-driven system
- **Additional skills** (fishing, cooking, smithing)
- **Multiplayer** via network sync
- **Mobile port** (Raylib supports iOS/Android)

## Conclusion

Building 3DRS was an experiment in AI-assisted game development, and I came away impressed by how far we've come. Claude wasn't just autocompleting code - it was:

- Designing systems architecture
- Writing shaders and debugging visual issues
- Creating content (quests, monsters, map regions)
- Testing its own work via the validation agent
- Migrating data formats when needed

The validation agent represents something new: **AI that can verify its own work visually**. When Claude implements a feature and then tests it by actually running the game and looking at the screenshots, we're getting closer to a world where software can be developed with minimal human intervention.

---

*Screenshots captured via automated headless testing - the same system the validation agent uses.*
