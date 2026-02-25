# Cosmic Heat - Documentation

Cosmic Heat is a 2D top-down space shooter written in Python using the Pygame library. The player pilots a spaceship through an endless gauntlet of enemy ships, three unique bosses, falling meteors, and black holes, while collecting power-ups and score coins. The game supports both keyboard and joystick input, runs at 60 FPS in a 1200x800 window, and features a fully scrolling background that transitions through four visual stages as the player progresses.


## Table of Contents

1. [Project Structure](#project-structure)
2. [Setup and Running](#setup-and-running)
3. [Controls](#controls)
4. [Game Architecture](#game-architecture)
5. [Main Menu](#main-menu)
6. [Game Loop](#game-loop)
7. [Background System](#background-system)
8. [Player](#player)
9. [Player Bullets](#player-bullets)
10. [Enemies](#enemies)
11. [Bosses](#bosses)
12. [Environmental Hazards](#environmental-hazards)
13. [Power-Ups and Collectibles](#power-ups-and-collectibles)
14. [Explosion System](#explosion-system)
15. [Scoring System](#scoring-system)
16. [HUD](#hud)
17. [Audio](#audio)
18. [Game Over and Reset](#game-over-and-reset)
19. [Sprite Groups](#sprite-groups)

## Project Structure

```
Cosmic Heat/
    main.py              -- Core game loop (~818 lines), all gameplay logic, rendering, and state
    menu.py              -- Main menu screen with Play / Exit buttons (~137 lines)
    controls.py          -- Player movement functions for keyboard and joystick (~47 lines)
    functions.py         -- Utility functions: background music, game over, game win (~39 lines)
    requirements.txt     -- Dependencies (pygame only)
    classes/
        constants.py     -- Global constants: screen size, FPS, colors, tuning values
        player.py        -- Player class with 8-directional movement
        bullets.py       -- Player bullet (Bullet class)
        enemies.py       -- Enemy1, Enemy2, Enemy2Bullet classes
        bosses.py        -- Boss1, Boss2, Boss3 and Boss1Bullet, Boss2Bullet, Boss3Bullet
        meteors.py       -- Meteors (diagonal), Meteors2 (vertical), BlackHole
        explosions.py    -- Explosion and Explosion2 animated sprite classes
        refill.py        -- BulletRefill, HealthRefill, DoubleRefill, ExtraScore
    images/
        bg/              -- 4 background images (background.jpg, background2-4.png)
        boss/            -- Boss sprites (boss1.png, boss2_1.png, boss3.png)
        bullets/         -- Bullet sprites for player, enemies, and all 3 bosses
        enemy/           -- Enemy sprites (3 Enemy1 variants, 2 Enemy2 variants)
        explosion/       -- 8-frame explosion animation sequence
        explosion2/      -- 18-frame explosion animation sequence (larger)
        explosion3/      -- 18-frame explosion animation sequence (boss kills)
        hole/            -- Black hole sprites (2 variants)
        meteors/         -- Meteor sprites (4 diagonal variants, 4 vertical variants)
        refill/          -- Power-up sprites (health, bullet, double)
        score/           -- Score coin sprite
        player.png       -- Player ship sprite
        life_bar.png     -- HUD health bar icon
        bullet_bar.png   -- HUD ammo bar icon
        mainmenu.jpg     -- Main menu background
        ch.png           -- Game logo
    game_sounds/
        background_music.mp3   -- In-game background music
        menu.mp3               -- Main menu music
        gameover.mp3           -- Death music
        win.mp3                -- Boss defeat jingle
        warning.mp3            -- Boss spawn alert
        shooting/              -- shoot.mp3, shoot2.mp3, boss1shoot.mp3, boss2shoot.mp3
        explosions/            -- explosion1.wav, explosion2.wav, explosion3.wav
        damage/                -- black_hole.mp3
        refill/                -- bullet_refill.wav, health_refill.wav, double_refill.mp3, extra_score.mp3
```

All game logic resides in `main.py` as a single procedural game loop. Entity classes are defined under the `classes/` directory and each class is responsible for its own movement logic, image loading, and sound playback. Asset paths are hardcoded as relative strings throughout the codebase, meaning the game must be run from the project root directory.


## Setup and Running

**Requirements:** Python 3.x, Pygame

```
git clone https://github.com/Dave-YP/cosmic-heat-pygame.git
cd cosmic-heat-pygame
python -m venv env
source env/Scripts/activate      # On Windows: env\Scripts\activate
pip install -r requirements.txt
python main.py
```

The only dependency listed in `requirements.txt` is `pygame`. No specific version is pinned.

On launch, the main menu is displayed first. After the player selects "Play," the menu module hands control to the main game loop.




### Controls


| Action     | Key              | Notes |
|------------|------------------|-------|
| Move       | Arrow keys       | 4-directional plus diagonals by holding two keys |
| Shoot      | Space            | Semi-automatic; hold to fire continuously |
| Pause      | P or Pause key   | Toggles pause; displays "PAUSE" text on screen |
| Exit       | Escape           | Immediately terminates the process via `sys.exit(0)` |

Diagonal movement is supported by holding two arrow keys simultaneously. For example, holding Up and Left calls `move_up_left()`, which moves the player diagonally at full speed in both axes (not normalized, so diagonal movement is roughly 1.41x faster than cardinal movement).

Shooting is semi-automatic. On the first press, a bullet fires immediately if `SHOOT_DELAY` (150ms) has elapsed since the last shot. While Space remains held, the game continues firing at the `SHOOT_DELAY` interval each frame where the timing condition is met. Each shot costs 1 ammo from the 200-unit pool. If ammo reaches 0, no bullets are fired regardless of input.


## Game Architecture

The game is built on Pygame's sprite system. All game entities (enemies, bullets, meteors, refills, explosions) extend `pygame.sprite.Sprite` and are organized into `pygame.sprite.Group` instances. There are 17 separate sprite groups in total (see [Sprite Groups](#sprite-groups)). Collision detection between groups is handled through `pygame.sprite.spritecollide` (for bullet-vs-entity), while single-entity checks use `rect.colliderect`.

The architecture is entirely procedural. `main.py` manages all state through module-level global variables and a single `while running` loop. There is no state machine, scene manager, or entity-component system. The menu is a separate module (`menu.py`) that runs its own independent `pygame.init()`, event loop, and display before handing control to the main game.

Key architectural characteristics:

- **Per-instance asset loading.** Each sprite class loads its own image and sound assets inside `__init__`. This means every new `Bullet`, `Explosion`, or `Boss1Bullet` instance re-reads the corresponding files from disk. Images passed as constructor arguments (enemies, meteors, refills) are the exception -- these are loaded once in `main.py` and shared via reference.
- **Global state.** All game state -- score, hi-score, player health, ammo, boss spawn flags, all sprite groups -- is stored as module-level variables in `main.py`. There are no encapsulating classes or data structures.
- **No persistence.** The game does not save anything to disk. Hi-score resets when the process exits. There are no settings, save files, or configuration.
- **Single-file game loop.** All spawning logic, collision handling, difficulty scaling, HUD rendering, and state reset on death exist inline within the `while running` loop in `main.py`.

### Module Dependencies

```
main.py
  |-- menu.py                  (Main menu, runs before game loop)
  |-- controls.py              (move_player, move_player_with_joystick)
  |-- functions.py             (show_game_over, music_background, show_game_win)
  |-- classes/constants.py     (WIDTH, HEIGHT, FPS, SHOOT_DELAY, colors)
  |-- classes/player.py        (Player)
  |-- classes/bullets.py       (Bullet)
  |-- classes/enemies.py       (Enemy1, Enemy2)
  |-- classes/bosses.py        (Boss1, Boss2, Boss3)
  |-- classes/meteors.py       (Meteors, Meteors2, BlackHole)
  |-- classes/explosions.py    (Explosion, Explosion2)
  |-- classes/refill.py        (BulletRefill, HealthRefill, DoubleRefill, ExtraScore)
```


## Main Menu

Defined in `menu.py`. The menu is a self-contained module with its own Pygame initialization, event loop, and display. It runs to completion before the main game starts.

**Visual layout:**
- Full-screen background image (`images/mainmenu.jpg`) scaled to 1200x800.
- Game logo (`images/ch.png`) centered horizontally near the top.
- Two buttons centered on screen: "Play" and "Exit", rendered as rounded rectangles with text.
- The currently selected button has a red border highlight.

**Navigation:**
- Mouse click on a button activates it.
- Arrow keys Up/Down switch between Play and Exit.
- Enter key activates the selected button.
- Joystick D-pad (hat) switches selection; button 0 activates.

**On "Play" press:**
1. An explosion sound effect plays.
2. The `animate_screen()` function runs, which shakes the background image randomly for 20 iterations (creating a screen-shake effect).
3. The menu loop exits, the screen fills with black, and `main.py` takes over.

**Music:** The menu plays `game_sounds/menu.mp3` on loop at 0.25 volume using 20 mixer channels.


## Game Loop

The main loop in `main.py` (line 133 onward) runs at 60 FPS via `clock.tick(FPS)` and processes the following each frame, in this exact order:

1. **Event handling** -- Iterates all Pygame events. Handles `QUIT`, `KEYDOWN`, `KEYUP`, `JOYBUTTONDOWN`, and `JOYBUTTONUP`. On `KEYDOWN` for Space, a bullet is fired immediately (subject to ammo and `SHOOT_DELAY`). Escape calls `sys.exit(0)` directly. P toggles the `paused` flag.

2. **Continuous shooting check** -- Outside the event loop, if Space or joystick button 0 is still held (`is_shooting` is True) and the `SHOOT_DELAY` has elapsed since `last_shot_time`, another bullet is created and ammo is decremented.

3. **Joystick movement** -- If a joystick is connected and the game is not paused, `move_player_with_joystick` is called. This reads the analog stick axes and directly modifies `player.rect.x` and `player.rect.y`.

4. **Pause check** -- If paused, "PAUSE" is rendered in white Comic Sans MS at screen center, the display is flipped, and the loop skips to the next iteration via `continue`. No game logic runs while paused.

5. **Keyboard movement** -- `move_player(keys, player)` is called. This reads the currently pressed keys and calls the appropriate Player movement methods, handling all 8 directions.

6. **Background rendering and scrolling** -- The background is drawn twice (current position and a copy above/below) to create a seamless vertical scroll. `bg_y_shift` increments by 1 each frame (plus 2 more if score > 3000). When it reaches 0, it resets to `-HEIGHT`.

7. **Background transitions** -- The current background image swaps based on score thresholds (3000, 10000, 15000). Resets to the original when score returns to 0 (after death).

8. **Hi-score update** -- `hi_score` is set to `score` if the current score exceeds it.

9. **Enemy and hazard spawning** -- Random rolls each frame determine whether new entities spawn. Each entity type has its own probability and score threshold (detailed in the Enemies, Bosses, and Hazards sections).

10. **Game over check** -- If `player_life <= 0`, `show_game_over(score)` is called (displays "GAME OVER" with final score for 4 seconds), then all state variables and sprite groups are reset.

11. **Entity update and collision loop** -- Each sprite group is iterated individually. For each entity, the pattern is:
    - Call `update()` to advance position/animation.
    - Call `draw(screen)` to render.
    - Check `colliderect` with the player for contact damage.
    - Check `spritecollide` with the player's bullet group for shooting.
    - On hit, spawn an explosion, award score, possibly drop a refill, and `kill()` the entity.

12. **Player rendering** -- The player image is blitted to the screen at its current `rect` position.

13. **Bullet rendering and cleanup** -- Each player bullet is updated and drawn. Bullets that exit the top of the screen are killed.

14. **HUD rendering** -- Health bar, ammo bar, score, and hi-score are drawn (see [HUD](#hud)).

15. **Display flip** -- `pygame.display.flip()` pushes the completed frame to the screen.

16. **Frame rate cap** -- `clock.tick(FPS)` limits the frame rate to 60.


---


## Background System

The game uses a vertically scrolling background to simulate forward motion through space. The implementation works as follows:

**Scrolling mechanic:**
- A variable `bg_y_shift` starts at `-HEIGHT` (-800) and increments by 1 pixel each frame.
- The current background image is drawn at y-offset `bg_y_shift`, and a copy is drawn directly above (or below) it to fill the gap.
- When `bg_y_shift >= 0`, it resets to `-HEIGHT`, creating a seamless loop.
- After 3000 score, `bg_y_shift` increments by an additional 2 pixels per frame, accelerating the scroll from 1 to 3 pixels per frame total.

**Background transitions:**
There are four background images loaded at startup:

| Image                          | Active When     |
|--------------------------------|-----------------|
| `images/bg/background.jpg`     | Score 0 - 2999  |
| `images/bg/background2.png`    | Score 3000 - 9999 |
| `images/bg/background3.png`    | Score 10000 - 14999 |
| `images/bg/background4.png`    | Score 15000+    |

Transitions happen instantly by swapping the `current_image` reference. There is no crossfade or animation -- the new background simply appears on the next frame. When the player dies and score resets to 0, the background reverts to the first image.



## Player

Defined in `classes/player.py`. The Player class does not extend `pygame.sprite.Sprite`. It manages its own `pygame.Rect` and image independently.

| Property          | Value                | Details |
|-------------------|----------------------|---------|
| Starting health   | 200                  | Managed as `player_life` in `main.py`, not as a Player attribute |
| Starting ammo     | 200                  | Managed as `bullet_counter` in `main.py` |
| Speed             | 10 pixels per frame  | Applied per movement method call |
| Hitbox size       | 100x100 pixels       | Defined by the Rect in `__init__` |
| Starting position | (500, 700)           | `WIDTH//2 - 100` for x, `HEIGHT - 100` for y |

**Movement system:**
The Player has 13 methods for movement control:
- `move_left`, `move_right`, `move_up`, `move_down` -- move 10 pixels in the named direction. `move_left` also flips the sprite image horizontally; `move_right` restores it.
- `move_up_left`, `move_up_right`, `move_down_left`, `move_down_right` -- move 10 pixels on both axes simultaneously. Diagonal speed is not normalized, so the effective speed is approximately 14.14 pixels per frame on diagonals.
- `stop`, `stop_left`, `stop_right`, `stop_up`, `stop_down` -- all are empty (`pass`). They exist as hooks but perform no action.

**Boundary clamping:** Each movement method checks the player's rect against screen edges and refuses to move if it would go out of bounds. For example, `move_left` only moves if `self.rect.left > 0`.

**Image handling:** The player image is loaded from `images/player.png`. A copy of the original is stored in `self.original_image`. When moving left, the image is horizontally flipped. When the player releases the shoot button, the image is restored from `original_image`.

**Note on health and ammo:** These are not attributes of the Player class itself. They are stored as separate global variables (`player_life` and `bullet_counter`) in `main.py` and managed entirely within the game loop.



## Player Bullets

Defined in `classes/bullets.py` as the `Bullet` class (extends `pygame.sprite.Sprite`).

| Property     | Value |
|--------------|-------|
| Speed        | 10 pixels per frame, moving upward |
| Image        | `images/bullets/bullet1.png` |
| Spawn offset | Centered on player's x, 10 pixels above player's top edge |
| Sound        | `game_sounds/shooting/shoot.mp3` at 0.4 volume |

Each bullet fires upward (`rect.move_ip(0, -self.speed)`). It is killed when its top edge passes y=1 (top of screen). The sound effect plays on instantiation, meaning every bullet creates a new `Sound` object and plays it immediately.

The player can hold the shoot button for continuous fire. The minimum interval between shots is 150ms (`SHOOT_DELAY`). At 60 FPS that translates to roughly one bullet every 9 frames. Each shot costs 1 ammo. When ammo hits 0, shooting stops. The maximum number of bullets on screen at once is effectively unlimited, but the fire rate and ammo pool naturally limit it.



## Enemies

### Enemy1

Defined in `classes/enemies.py`. The basic enemy type, present from frame 1.

**Spawning:**
- Probability: 1/121 chance per frame (~0.83% per frame, roughly one every 2 seconds on average).
- Spawn location: Random x between 100 and WIDTH-50, random y between -HEIGHT and -50 (above the visible screen, so it drifts into view).
- Image: Randomly selected from 3 visual variants (`enemy1_1.png`, `enemy1_2.png`, `enemy1_3.png`).
- There is no cap on how many Enemy1 can exist simultaneously.

**Movement behavior:**
- Starts with a random direction chosen from 8 options (all combinations of -1, 0, 1 for x and y, excluding (0,0) based on the choices given).
- Moves at speed 4 per frame in its current direction.
- When it hits a screen edge, it bounces by picking a new random direction that moves it away from that edge. For example, hitting the left edge picks from directions that include positive x.
- Clamped to stay within 5 pixels of each screen edge.

**Enemy-enemy collision repulsion:**
When two Enemy1 sprites overlap (detected via `pygame.sprite.spritecollide`), a physics-like repulsion is applied:
1. The distance vector between the two enemies is calculated.
2. Each enemy's direction is reflected across the distance vector using `Vector2.reflect()`.
3. A repulsion force pushes them apart, scaled by `ENEMY_FORCE` (4) and the overlap amount.
4. Both enemies' positions are adjusted by the repulsion vector.

This creates emergent bouncing behavior where groups of enemies ricochet off each other organically.

**On death:**
- Killed by player bullet: awards 50 score, spawns an Explosion, 1/9 chance to drop a BulletRefill at the enemy's position, 1/9 chance to spawn a HealthRefill at a random screen position.
- Killed by player collision: awards 20 score, spawns an Explosion, deals 10 damage to the player. No item drops.

### Enemy2

Also defined in `classes/enemies.py`. Unlocks at 3000 score.

**Spawning:**
- Probability: 1/41 chance per frame, but only if fewer than 2 are currently on screen.
- Spawn location: Random x between 200 and WIDTH-100, random y above the screen.
- Image: Randomly selected from 2 visual variants (`enemy2_1.png`, `enemy2_2.png`).

**Movement behavior (Phase 1 -- Shooting):**
- Moves horizontally, bouncing off the left and right screen edges.
- Speed: 4 pixels per frame.
- Uses the same enemy-enemy collision repulsion system as Enemy1 (including collisions with other Enemy2 instances).
- Stays at or above y=5 (clamped to the upper portion of the screen during this phase).

**Shooting behavior:**
- Increments a `shoot_timer` every frame. When it reaches 60 (once per second), fires a single `Enemy2Bullet` straight down from its center-bottom position. Then resets the timer.
- Tracks total shots fired via `shots_fired`.

**Movement behavior (Phase 2 -- Kamikaze, after 10 shots):**
- Once `shots_fired` reaches 10, the enemy stops shooting.
- Speed increases to 10 pixels per frame.
- Calculates a normalized direction vector directly toward the player's current position each frame.
- Flies straight at the player until it either hits or goes off screen.

**Enemy2 Bullets:**
- Image: `images/bullets/bullet4.png`
- Speed: 8 pixels per frame, straight down.
- Sound: `game_sounds/shooting/shoot2.mp3` at 0.3 volume.
- Killed when its top edge passes the bottom of the screen.
- Deals 10 damage to the player on contact.

**On death:**
- Killed by player bullet: awards 80 score, spawns an Explosion2 (larger explosion), 1/21 chance to drop a DoubleRefill.
- Killed by player collision: awards 20 score, spawns an Explosion2, deals 40 damage to the player. No item drops.



## Bosses

All bosses are defined in `classes/bosses.py`. Each boss spawns exactly once when the player's score reaches its threshold for the first time. A warning sound (`game_sounds/warning.mp3`) plays on spawn. Each boss has a visible health bar rendered above its sprite (red background with green fill proportional to remaining health).

All three bosses share a two-phase behavior pattern: a shooting phase followed by a kamikaze phase.

### Boss1

| Property              | Value |
|-----------------------|-------|
| Score threshold       | 5000  |
| Health                | 150   |
| Damage per bullet hit | 5 (requires 30 player bullets to kill) |
| Movement speed        | 6     |
| Bullet speed          | 10    |
| Fire rate             | Every 60 frames (1 second) |
| Shots before kamikaze | 20    |
| Kill reward           | 400 score |
| Body contact damage   | 20 per frame |
| Bullet damage         | 20 per hit |

**Movement (Phase 1):**
- Moves horizontally at speed 6, bouncing off left and right edges.
- Has a sinusoidal wobble applied to both x and y: `sin(ticks * 0.01) * 3`, which gives it a slight oscillating drift that makes its path less predictable.
- Clamped to stay at y >= 50 (upper portion of the screen).

**Attack pattern:**
- Every 60 frames, fires a triple-shot: three `Boss1Bullet` projectiles from center-left (-20), center, and center-right (+20). All three fire straight down at speed 10.
- After 20 volleys (60 total bullets fired), enters Phase 2.

**Movement (Phase 2 -- Kamikaze):**
- Speed increases to 10.
- Calculates a normalized direction toward the player and flies directly at them each frame.

**Boss1 Bullet:**
- Image: `images/bullets/bulletboss1.png`
- Fires straight down at speed 10.
- Sound: `game_sounds/shooting/boss1shoot.mp3` at 0.4 volume.

### Boss2

| Property              | Value |
|-----------------------|-------|
| Score threshold       | 10000 |
| Health                | 150   |
| Damage per bullet hit | 8 (requires 19 player bullets to kill) |
| Movement speed        | 5 (3.54 on diagonals, normalized) |
| Bullet speed          | 11    |
| Fire rate             | Every 100 frames (~1.67 seconds) |
| Shots before kamikaze | 20    |
| Kill reward           | 800 score |
| Body contact damage   | 2 per frame |
| Bullet damage         | 20 per hit |

**Movement (Phase 1):**
- Moves in 8 directions. Unlike Enemy1, Boss2 normalizes its speed on diagonals by dividing by sqrt(2), so it moves at a consistent effective speed in all directions.
- Bounces off all four screen edges. When hitting an edge, the perpendicular axis direction is also adjusted to avoid getting stuck.
- The top boundary is set to y=70 rather than 0, keeping the boss away from the very top of the screen.
- Same sinusoidal wobble as Boss1, but at amplitude 2 instead of 3.

**Attack pattern:**
- Every 100 frames, fires a single aimed `Boss2Bullet` toward the player's current position. The bullet's direction is calculated as a normalized vector from the boss to the player at the moment of firing.
- After 20 shots, enters Phase 2.

**Movement (Phase 2 -- Kamikaze):**
- Calculates a normalized direction toward the player each frame and flies at them.
- Speed remains at the diagonal-adjusted rate (approximately 3.54).

**Boss2 Bullet:**
- Image: `images/bullets/bulletboss2.png`
- Speed: 11, travels in the aimed direction.
- The bullet sprite rotates each frame to face its direction of travel using `atan2` and `pygame.transform.rotate`.
- Sound: `game_sounds/shooting/boss2shoot.mp3` at 0.4 volume.

### Boss3

| Property              | Value |
|-----------------------|-------|
| Score threshold       | 15000 |
| Health                | 200   |
| Damage per bullet hit | 6 (requires 34 player bullets to kill) |
| Movement speed        | 5 (3.54 on diagonals, normalized) |
| Bullet speed          | 15 (fastest projectile in the game) |
| Fire rate             | Every 120 frames (2 seconds) |
| Shots before kamikaze | 20    |
| Teleport interval     | Every 160 frames (~2.67 seconds) |
| Kill reward           | 1000 score |
| Body contact damage   | 1 per frame |
| Bullet damage         | 20 per hit |

**Movement (Phase 1):**
- Identical to Boss2's 8-directional movement with diagonal normalization and edge bouncing.
- Same sinusoidal wobble at amplitude 2.

**Teleportation:**
- Boss3 has a unique mechanic: a `teleport_timer` that increments every frame. When it reaches `teleport_interval` (160), the boss instantly teleports to a random position within (50 to WIDTH-50, 100 to HEIGHT-100) and the timer resets. This teleportation occurs in both Phase 1 and Phase 2, and is independent of the movement/shooting phases.

**Attack pattern:**
- Every 120 frames, fires a single aimed `Boss3Bullet` toward the player. Same aiming system as Boss2.
- After 20 shots, enters Phase 2 (kamikaze).

**Movement (Phase 2 -- Kamikaze):**
- Same as Boss2: flies toward the player at the diagonal-adjusted speed while still teleporting every 160 frames.

**Boss3 Bullet:**
- Image: `images/bullets/bulletboss3.png`
- Speed: 15, the fastest projectile in the game. Travels in the aimed direction.
- Rotates to face its direction of travel, same as Boss2Bullet.
- Sound: Reuses `game_sounds/shooting/boss2shoot.mp3` at 0.4 volume.

### Boss Drop Table

All three bosses have a 1/21 chance to drop a DoubleRefill on death.


---


## Environmental Hazards

### Meteors (diagonal)

Defined as `Meteors` in `classes/meteors.py`. Unlocks at 3000 score.

- **Spawn rate:** 1/101 chance per frame when score > 3000.
- **Spawn location:** Random x between 0 and 50, random y between 0 and 50 (top-left corner). This means they always enter from the upper-left.
- **Movement:** Travels diagonally downward and to the right (`direction_x = 1`, `direction_y = 1`). Base speed is 2 pixels per frame on each axis.
- **Rotation:** Continuously rotates by 1 degree per frame using `pygame.transform.rotozoom`. The rect is re-centered after each rotation to prevent drift.
- **Despawn:** Killed when its bottom edge exceeds `HEIGHT + 50` or its right edge exceeds `WIDTH + 50`.
- **Image:** 4 visual variants (`meteor_1.png` through `meteor_4.png`), randomly selected at spawn.
- **Player collision:** Deals 10 damage, awards 50 score, spawns an Explosion, and destroys the meteor.
- **Shot by player:** Awards 80 score, spawns an Explosion, 1/11 chance to drop a DoubleRefill.
- **Speed scaling:** Speed is overwritten each frame based on score (4 at 3000, 6 at 10000, 8 at 15000, 10 at 20000+).

### Meteors2 (vertical)

Defined as `Meteors2` in `classes/meteors.py`. Present from the start.

- **Spawn rate:** 1/91 chance per frame.
- **Spawn location:** Random x between 100 and WIDTH-50, random y above the screen.
- **Movement:** Falls straight down (`direction_x = 0`, `direction_y = 1`). Base speed is 2 pixels per frame.
- **Rotation:** Same continuous rotation as diagonal meteors.
- **Despawn:** Killed when its bottom edge exceeds `HEIGHT + 300`.
- **Image:** 4 visual variants (`meteor2_1.png` through `meteor2_4.png`).
- **Player collision:** Deals 10 damage, awards 20 score.
- **Shot by player:** Awards 40 score, 1/21 chance to drop a DoubleRefill.
- **Speed scaling:** Same as diagonal meteors: 4, 6, 8, 10 at 3000, 10000, 15000, 20000+ respectively.

### Black Holes

Defined as `BlackHole` in `classes/meteors.py`. Unlocks at 1000 score.

- **Spawn rate:** 1/501 chance per frame.
- **Spawn location:** Random x between 100 and WIDTH-50, random y above the screen.
- **Movement:** Falls straight down at speed 2, with continuous rotation.
- **Despawn:** Killed when bottom edge exceeds `HEIGHT + 300`.
- **Image:** 2 visual variants (`black_hole.png`, `black_hole2.png`).
- **Damage:** Deals 1 damage per frame of overlap. Unlike meteors, black holes are not destroyed on contact. The player takes continuous damage as long as they overlap with the black hole hitbox. At 60 FPS, sustained contact drains 60 HP per second.
- **Indestructible:** Cannot be destroyed by player bullets. There is no collision check between bullets and black holes in the game loop.
- **Sound:** Plays `game_sounds/damage/black_hole.mp3` each frame of contact.


---


## Power-Ups and Collectibles

All power-up classes are in `classes/refill.py`. All extend `pygame.sprite.Sprite`. Refills (BulletRefill, HealthRefill, DoubleRefill) share a common movement pattern: they bounce around the screen randomly, changing direction with a 1/51 chance per frame. Their speed is 1 pixel per frame (DoubleRefill is 2). They are clamped to stay within screen bounds.

### Bullet Refill

| Property     | Value |
|--------------|-------|
| Ammo restored | 50 (capped at 200) |
| Movement speed | 1 |
| Sound        | `game_sounds/refill/bullet_refill.wav` at 0.4 volume |

- Spawns at the death location of an Enemy1, with a 1/9 chance per Enemy1 killed by a player bullet.
- Always consumed on contact, even if ammo is already full.

### Health Refill

| Property      | Value |
|---------------|-------|
| Health restored | 50 (capped at 200) |
| Movement speed | 1 |
| Sound         | `game_sounds/refill/health_refill.wav` at 0.4 volume |

- Spawns with a 1/9 chance when an Enemy1 is killed by a player bullet. Unlike BulletRefill, it spawns at a random screen position rather than at the enemy's death location.
- Always consumed on contact, even if health is full.

### Double Refill

| Property          | Value |
|-------------------|-------|
| Health restored   | 50 (capped at 200) |
| Ammo restored     | 50 (capped at 200) |
| Movement speed    | 2 |
| Sound             | `game_sounds/refill/double_refill.mp3` at 0.4 volume |

- The premium power-up. Restores both resources at once.
- Drop sources and rates:

| Source                | Drop chance |
|-----------------------|-------------|
| Diagonal meteor shot  | 1/11        |
| Vertical meteor shot  | 1/21        |
| Enemy2 killed         | 1/21        |
| Boss1 killed          | 1/21        |
| Boss2 killed          | 1/21        |
| Boss3 killed          | 1/21        |

### Extra Score (Score Coin)

| Property     | Value |
|--------------|-------|
| Score awarded | 20 |
| Movement speed | 2 (base), scales with score |
| Sound        | `game_sounds/refill/extra_score.mp3` at 0.4 volume |

- Spawns independently of enemies, with a 1/61 chance per frame.
- Falls straight down (no bouncing).
- The score coin image (`images/score/score_coin.png`) is also used as the icon next to the score display in the HUD.
- Killed when bottom edge exceeds `HEIGHT + 100`.
- Speed increases with score: 2 (base), 4 (at 10000), 6 (at 15000), 8 (at 20000+). Note: the 3000 threshold sets speed to 2 (same as base), so the first real increase is at 10000.


---


## Explosion System

Defined in `classes/explosions.py`. Explosions are frame-based animated sprites.

### Explosion (standard)

- Used for: Enemy1 deaths, meteor destructions, boss bullet impacts on the player.
- Animation: 8 frames loaded from `images/explosion/explosion0.png` through `explosion7.png`.
- Frame rate: 60ms per frame.
- Sound: Randomly selects one of 3 explosion sounds (`explosion1.wav`, `explosion2.wav`, `explosion3.wav`) at 0.3 volume. Sound plays once when the second frame is displayed (not the first frame).
- The sprite kills itself after the last frame is displayed.

### Explosion2 (large)

- Used for: Enemy2 deaths, boss body hits, boss deaths.
- Animation: 18 frames loaded from `images/explosion2/explosion0.png` through `explosion17.png`.
- Frame rate: 60ms per frame (longer total duration due to more frames).
- Sound: Uses only `explosion3.wav` at 0.3 volume.
- Same lifecycle as standard Explosion.

Boss deaths also use a third set of explosion images (`explosion3/`), loaded as `explosion3_images`, which is a different 18-frame sequence used for the final large explosion when a boss is destroyed.


---


## Scoring System

| Action                           | Score Awarded |
|----------------------------------|---------------|
| Enemy1 killed by bullet          | +50           |
| Enemy1 collision with player     | +20           |
| Enemy2 killed by bullet          | +80           |
| Enemy2 collision with player     | +20           |
| Meteor (diagonal) shot           | +80           |
| Meteor (diagonal) player hit     | +50           |
| Meteor2 (vertical) shot          | +40           |
| Meteor2 (vertical) player hit    | +20           |
| Score Coin collected             | +20           |
| Boss1 killed                     | +400          |
| Boss2 killed                     | +800          |
| Boss3 killed                     | +1000         |

The hi-score is tracked as a separate global variable (`hi_score`), updated every frame if `score > hi_score`. It is session-only and is not saved to disk. On game over, score resets to 0 but hi-score persists until the process exits.

Note that the player earns score when enemies or meteors collide with them, meaning taking damage is also a source of score. This is likely a design choice to reward aggressive play, though it also means the player's score increases passively from unavoidable collisions.


## HUD

The heads-up display is rendered each frame as the final step before `pygame.display.flip()`.

### Health Bar (top-left, y=10)

- Composed of a semi-transparent surface (alpha 216) with two layers: the `life_bar.png` icon on the left and a colored fill bar to the right (starting at x=35).
- The fill width is proportional to `player_life / 200 * 200` pixels, clamped between 0 and 200.
- Fill color changes based on health:
  - Above 50 HP: pale green (152, 251, 152).
  - 50 HP or below: black (0, 0, 0). This serves as a visual low-health warning.
- Height: 30 pixels for the fill, 25 pixels for the container surface.

### Bullet Counter (below health bar, y = health bar height + 20)

- Same composition as the health bar: icon on the left, colored fill to the right.
- Fill width is proportional to `bullet_counter / 200 * 200`.
- Fill color:
  - Above 50 ammo: red (255, 23, 23).
  - 50 ammo or below: black (0, 0, 0).
- Uses `bullet_bar.png` as the icon.

### Score (top-right)

- Rendered using Comic Sans MS, size 30, in pale goldenrod color (238, 232, 170).
- Positioned 10 pixels from the right edge, with the score coin image (`score_coin.png`) displayed to the right of the text.

### Hi-Score (top-center)

- Rendered using Comic Sans MS, size 20, in white (255, 255, 255).
- Semi-transparent (alpha 128).
- Centered horizontally, y=0.
- Displays as "HI-SCORE: {value}".

### Boss Health Bars

- Displayed above each active boss sprite when the boss is alive.
- A 5-pixel tall rectangle: red background (full width = boss max health), green foreground (width = current health).
- Positioned 5 pixels above the boss sprite, centered on the boss's x position.



## Audio

### Mixer Configuration

The game initializes `pygame.mixer` with 20 channels. All channels are set to 0.25 volume. The menu performs its own `pygame.mixer.init()` and channel setup independently.

### Music Tracks

| Track                              | Context             | Playback |
|------------------------------------|----------------------|----------|
| `game_sounds/menu.mp3`            | Main menu            | Looped (-1), 0.25 volume |
| `game_sounds/background_music.mp3`| During gameplay      | Looped (-1), 0.25 volume |
| `game_sounds/gameover.mp3`        | Player death screen  | Plays once |
| `game_sounds/win.mp3`             | Boss defeat screen   | Plays once |
| `game_sounds/warning.mp3`         | Boss spawn alert     | Plays once, as a Sound (not music channel) |

When the player dies, `gameover.mp3` plays with a 4-second delay, after which `background_music.mp3` resumes.

### Sound Effects

Sound effects are loaded by individual sprite classes in their constructors. Each instance creates its own `Sound` object.

| Sound File                                  | Used By           | Volume |
|---------------------------------------------|-------------------|--------|
| `game_sounds/shooting/shoot.mp3`           | Bullet (player)   | 0.4    |
| `game_sounds/shooting/shoot2.mp3`          | Enemy2Bullet      | 0.3    |
| `game_sounds/shooting/boss1shoot.mp3`      | Boss1Bullet       | 0.4    |
| `game_sounds/shooting/boss2shoot.mp3`      | Boss2Bullet, Boss3Bullet | 0.4 |
| `game_sounds/explosions/explosion1.wav`    | Explosion (random) | 0.3   |
| `game_sounds/explosions/explosion2.wav`    | Explosion (random) | 0.3   |
| `game_sounds/explosions/explosion3.wav`    | Explosion, Explosion2 | 0.3 |
| `game_sounds/damage/black_hole.mp3`        | BlackHole          | loaded by class |
| `game_sounds/refill/bullet_refill.wav`     | BulletRefill       | 0.4    |
| `game_sounds/refill/health_refill.wav`     | HealthRefill       | 0.4    |
| `game_sounds/refill/double_refill.mp3`     | DoubleRefill       | 0.4    |
| `game_sounds/refill/extra_score.mp3`       | ExtraScore         | 0.4    |


---


## Game Over and Reset

When `player_life` drops to 0 or below, the following sequence executes:

1. `show_game_over(score)` is called from `functions.py`. This function:
   - Renders "GAME OVER" in Impact font, size 50, in dark red (139, 0, 0) at screen center.
   - Renders "Final Score: {score}" in Impact font, size 30, in white, 50 pixels below.
   - Flips the display.
   - Loads and plays `gameover.mp3`.
   - Pauses the game for 4000 milliseconds (`pygame.time.delay`).
   - Resumes background music.

2. All state variables are reset:
   - `score` = 0 (but `hi_score` is preserved).
   - `player_life` = 200.
   - `bullet_counter` = 200.
   - `player.rect.topleft` = `initial_player_pos` (center-bottom of screen).
   - `boss1_spawned`, `boss2_spawned`, `boss3_spawned` = False.
   - `boss1_health` = 150, `boss2_health` = 150, `boss3_health` = 200.

3. All 14 sprite groups are emptied: bullets, all enemy groups, all boss groups, all refill groups, meteors, black holes, explosions.

4. The game loop continues immediately. There is no "return to menu" option, no "press any key to continue," and no transition screen. The player is simply placed back at the starting position with everything cleared, and the game restarts from score 0. The background reverts to the first image because the score is checked as `score == 0`.



## Sprite Groups

`main.py` declares 17 sprite groups at module level to organize all game entities:

| Group Name           | Contents                    | Max Count |
|----------------------|-----------------------------|-----------|
| `bullets`            | Player bullets              | Unlimited |
| `enemy1_group`       | Enemy1 instances            | Unlimited |
| `enemy2_group`       | Enemy2 instances            | 2         |
| `enemy2_bullets`     | Enemy2Bullet instances      | Unlimited |
| `boss1_group`        | Boss1 (0 or 1)              | 1         |
| `boss2_group`        | Boss2 (0 or 1)              | 1         |
| `boss3_group`        | Boss3 (0 or 1)              | 1         |
| `boss1_bullets`      | Boss1Bullet instances       | Unlimited |
| `boss2_bullets`      | Boss2Bullet instances       | Unlimited |
| `boss3_bullets`      | Boss3Bullet instances       | Unlimited |
| `bullet_refill_group`| BulletRefill instances      | Unlimited |
| `health_refill_group`| HealthRefill instances      | Unlimited |
| `double_refill_group`| DoubleRefill instances      | Unlimited |
| `extra_score_group`  | ExtraScore (coin) instances | Unlimited |
| `meteor_group`       | Meteors (diagonal)          | Unlimited |
| `meteor2_group`      | Meteors2 (vertical)         | Unlimited |
| `black_hole_group`   | BlackHole instances         | Unlimited |
| `explosions`         | Explosion instances         | Unlimited |
| `explosions2`        | Explosion2 instances        | Unlimited |

All groups are emptied on game over.
