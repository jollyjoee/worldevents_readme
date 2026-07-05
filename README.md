# WorldEvents — Admin & Configuration Guide

This guide walks you through setting up, configuring, and testing a new World Event from scratch.

---

## Architecture Overview

A World Event runs in three main environments:
1. **The Overworld:** Where a temporary **Portal** structure is pasted at a random or predetermined location. Players walk into this portal to join the fight.
2. **The Arena World:** An isolated world (managed by Multiverse-Core) where the boss fight occurs.
3. **The Spectator Vantage Point:** Where players are sent (in spectator mode) if they die during the boss battle.

---

## Step 1: Set Up the Arena World

The boss fight takes place in a dedicated arena world to prevent griefing, keep the main worlds lag-free, and ensure safe spectator mode.

1. **Create the Arena World:**
   Create a flat or custom arena world using Multiverse-Core:
   ```bash
   /mv create world_events_arena normal -t flat
   ```
2. **Build your Arena:**
   Build the arena walls, barriers, and spawn points in this world.
3. **Configure Coordinates in `config.yml`:**
   Open `plugins/WorldEvents/config.yml` and set the spawn coordinates for both active fighters and spectators:
   ```yaml
   arena:
     world-name: "world_events_arena"
     # Point where players teleport when entering the portal
     spawn:
       x: 0.5
       y: 65.0
       z: 0.5
       yaw: 0.0
       pitch: 0.0
     # Point where dead players teleport to watch the remaining fight
     spectator-spawn:
       x: 0.5
       y: 80.0
       z: 0.5
       yaw: 0.0
       pitch: 45.0
   ```

---

## Step 2: Prepare the Portal Schematic

When an event starts, the plugin pastes a portal structure into the Overworld, backing up the natural terrain first. When the entry phase ends, the portal disappears and the terrain is rolled back perfectly.

1. **Build a Portal:**
   Build your portal frame (e.g., nether bricks, obsidian, decorative blocks) somewhere on your server.
2. **Select and Save with WorldEdit/FAWE:**
   * Stand in front of your portal structure.
   * Select the region containing the portal using your WorldEdit wand (`//wand`).
   * Stand exactly where you want the portal's center/bottom to align with the ground when pasted.
   * Copy the portal relative to your position:
     ```bash
     //copy
     ```
   * Save the selection as a schematic named `portal.schem`:
     ```bash
     //schem save portal
     ```
3. **Install the Schematic:**
   Move the generated file from `plugins/WorldEdit/schematics/portal.schem` to `plugins/WorldEvents/schematics/portal.schem`.

---

## Step 3: Configure Portal Placement

You can configure the portal to spawn **randomly** across the world or at **predefined** locations.

### Option A: Randomly Generated Portals (Dynamic Spawning)
The plugin will search for a safe solid block (away from lava, water, air, or high drops) within a designated radius from spawn.

Set the following in `config.yml`:
```yaml
portal:
  schematic-file: "portal.schem"
  # Allowed worlds for random spawning
  allowed-worlds:
    - "world"
  # Scan radius from the world spawn point
  search-radius: 1000
  # Max attempts to scan for a safe solid block before giving up and using fallbacks
  search-max-attempts: 20
```

### Option B: Predetermined Portal Locations (Fixed Spawning)
If you want the portal to always spawn at specific hubs or coordinates, list them under `fallback-locations`. If the random scanner fails—or if you set `search-max-attempts: 0` to force fixed coordinates—the plugin will cycle through these fallbacks.

Set the following in `config.yml`:
```yaml
portal:
  search-max-attempts: 0 # Force immediate use of predetermined fallback locations
  fallback-locations:
    - "world,100.5,64.0,200.5"  # Format: "world_name,x,y,z"
    - "world,-450.0,72.0,812.5"
```

---

## Step 4: Create a Boss Configuration File

Create a new file in `plugins/WorldEvents/bosses/` (e.g., `skeleton_king.yml`).

```yaml
# Must match the internal ID of your MythicMobs mob configuration
mythicmobs-id: "SkeletonKing"

# Display name used in server broadcasts (MiniMessage supported)
display-name: "<red><bold>The Skeleton King</bold></red>"

# The level of the spawned MythicMobs entity
boss-level: 5

# Base health when only 1 player enters the arena
base-health: 10000.0

# HP multiplier added per additional player
# Formula: HP = base-health * (1 + (scaling-factor * (participants - 1)))
# With 0.25 scaling and 5 players: 10000 * (1 + (0.25 * 4)) = 20000 HP
scaling-factor: 0.25

# Options: THRESHOLD_BONUS (standard), FLAT (everyone gets same), PROPORTIONAL (scales with damage)
loot-mode: THRESHOLD_BONUS

# Minimum damage share % required to qualify for loot
min-damage-percent: 2.0

# Weighted chance for the scheduler to select this boss (higher = more common)
weight: 10

# Seconds players have to celebrate/pick up items before being teleported back
post-fight-delay-seconds: 30

loot:
  # Rewards given to all qualifying participants
  participation:
    # 1. Standard Minecraft Item:
    - material: DIAMOND
      amount: 5
      display-name: "<aqua>Cursed Gem</aqua>"
      lore:
        - "<gray>A shining gem containing dark energy.</gray>"
    # 2. Nexo Custom Item:
    - nexo-id: ruby_ore
      amount: 2

  # Bonus rewards for the top 3 damage dealers (only active in THRESHOLD_BONUS mode)
  top-bonus:
    - material: NETHERITE_INGOT
      amount: 1
      display-name: "<gold>King's Tribute</gold>"
    - nexo-id: crown_helmet
      amount: 1

  # Console commands run on behalf of all qualifying players
  commands:
    - "eco give {player} 1500"
    - "xp give {player} 10 l"
```

---

## Step 5: Testing the Event

Use the following workflow to test your setup safely:

1. **Reload configurations:**
   If you made changes to the main config or boss files, run:
   ```bash
   /we reload
   ```
2. **Force-Start the Event:**
   Manually trigger the event for a specific boss configuration (e.g., `skeleton_king`):
   ```bash
   /we start skeleton_king
   ```
   *Note: To test random selection, run `/we start` with no arguments.*
3. **Verify Portal Spawning:**
   Check the coordinates broadcasted in the chat. Teleport to the portal and verify it pasted correctly and is spawning particles.
4. **Enter the Portal:**
   Walk into the portal structure. You should be cached, teleported to the arena world, and receive a welcome message.
5. **Monitor Status:**
   Run the status command to verify active details:
   ```bash
   /we status
   ```
6. **Simulate Death (Optional):**
   If you die, verify that:
   * You do **not** lose your items or experience.
   * You are converted to spectator mode and teleported to your configured spectator point.
7. **Simulate Victory:**
   Kill the boss. Verify that:
   * A server-wide broadcast announces the victory and shows the top damage statistics.
   * Loot is distributed (or queued to your mailbox if your inventory is full).
   * After 30 seconds (or your configured delay), you are restored to your pre-event location, inventory state, and game mode.
8. **Test Mailbox Claims:**
   Fill up your inventory completely and run a fight. Once the boss dies, verify you receive a notification about pending loot. Clear some inventory space and run:
   ```bash
   /we claim
   ```
9. **Emergency Cancelling:**
   If something goes wrong during testing, you can cancel the active event immediately using:
   ```bash
   /we cancel
   ```
   This immediately despawns the boss, rolls back the overworld portal terrain, and restores all participants back to their original positions safely.
