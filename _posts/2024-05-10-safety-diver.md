---
title: Safety Diver Devlog
color: "#e86a3f"
class: black-text
project: safety-diver
---

The [Uncrank'd jam](https://itch.io/jam/uncrankdjam) is here, and the themes are Reflection and Float. After a bit of brainstorming i decided to give buoyancy physics a try, to stay in theme of making games based on simple physics principles. The deadline is on **May 11th 2024 at 1:00 PM GMT**

{% assign images = "/assets/images/safety-diver" %}

# Day 1 (3/5):
- implemented basic buoyancy!
	- the player is calculated as a sphere with a radius of 15
	- had to figure out how to calculate the volume of a cut sphere
	- player's rotor is significantly less influential in the air
- made a simple animated player sprite
![Safety diver gameplay]({{ images }}/playdate-20240503-192018.gif)

# Day 2:
- made the camera move freely on the vertical axis!
- with camera drag too!
- added some super simple water and air resistance physics - the player is slowed down more in water (but not too much, otherwise it wouldnt be fun!)
![Safety diver gameplay]({{ images }}/playdate-20240504-124532.gif)
- also added a depth HUD
![Safety diver gameplay]({{ images }}/playdate-20240504-130535.gif)
- im thinking the basic premise can be that you dive down for gold and stuff while avoiding hazards, and have to make it back to the surface for all the collected items to count!
- i'm very much enjoying taking a little break from spaceshipment, this has been fun so far!

# Day 3:
- added coins for the player to collect!
- added a score display to show how many coins the player has collected
![Safety diver gameplay]({{ images }}/playdate-20240505-142708.gif)
- next we add hazards for the player to avoid (and maybe tweak the movement to give the player a chance to avoid it)

# Day 5:
- added hazards!
- the player now carries an amount of treasure with them (which they lose if they touch a hazard) which gets added to the main score when they resurface
![Safety diver gameplay]({{ images }}/playdate-20240507-225931.gif)
- also added a splash and coin pickup sound effects
- next will end the game if the player comes up with 0 held coins
	- and proper hazard collisions
	- as well as motor, bringing treasure to surface and zap (colliding with hazard) sounds

# Day 6:
 - added a camera limit at the bottom of the sea
 ![Safety diver gameplay]({{ images }}/playdate-20240508-165007.gif)
 - and added real hazard collisions!
 ![Safety diver gameplay]({{ images }}/playdate-20240508-164815.gif)
 - zap sound when colliding with hazards
 - rewrote the entire codebase to access `PlaydateAPI *pd` from one place - pd_pointer.h
 - more abstraction!
 - fixed bug where you can often see past the floor of the ocean when approaching it

# Day 7:
- tomorrow's the second to last day, so it's time to get cracking on what's left to finish for the game!
- added a game over/restart sequence (might add high score storage later)
- resurface score increase sound effects
- restructured my code to move all level logic from game.c to level/level.c
- made a nice pretty game over screen
- simple motor sound (sounds the same out of water and in)

# Day 8: The FINAL Day
- with the final day today, i took the time to clean up the game and its itch.io page
- added a little bonk sound for hitting the walls
- improved the motor sound
- released the game and the github page!
