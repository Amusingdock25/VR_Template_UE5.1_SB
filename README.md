# VR_Template_UE5.1_SB

For the source-built version of UE5.1

This is the one you want for dedicated servers, a "call function by name" blueprints node, and a Android permissions sanitizer.

This version will receive free updates.

If you do not have the Unreal Engine built from source, use the launcher version: [https://github.com/Incurian/VR_Template_UE5.1_EL](https://github.com/Incurian/VR_Template_UE5.1_EL)

---

## Introduction

**Note to self**: remove keys before publishing  
Do not remove this note, you'll forget next time

Get help from our Discord server any time: [https://discord.gg/ANt8bbh3rA](https://discord.gg/ANt8bbh3rA)

VR is a hard problem. There are too many ways to move and too few ways to stop the player from moving; any solutions need to be computationally cheap and not make the player sick. VR gives you effortless presence and intuitive interactions, but it makes breaks from reality all the more jarring. Making the world too much like reality can backfire because the player's ability to interact with it is significantly handicapped compared to reality; perfect simulations that don't feel right are counter-productive. It needs to be as realistic as can be experienced properly, but not one iota more.

The solutions are made more difficult by my decision to use blueprint scripting exclusively, but I think it's worthwhile because blueprints are surprisingly powerful considering how easy they are to use.

Please report bugs or opportunities to improve the blueprints. I'm begging you.

This will be more of a quick-start guide than a detailed manual for each function. I will try to make the comments within the blueprints explanatory, and make it easy to switch common options. The idea is that you will know that a certain function exists and where, if not necessarily how it works. If you have any questions please ask.

---

## Framework

Ensure these are selected either in **ProjectSettings -> Maps & Modes**, or in **WorldSettings -> GameModeOverride**  
Note that there are custom collision channels, and they are required. About half of all bugs are from improper collision settings.

**refs**:  
[Gameplay Framework Quick Reference (UE5.1)](https://docs.unrealengine.com/5.1/en-US/gameplay-framework-quick-reference-in-unreal-engine/)  
[Game Mode and Game State (UE5.1)](https://docs.unrealengine.com/5.1/en-US/game-mode-and-game-state-in-unreal-engine/)

- **Game Mode: GMode_22**  
  Accepts new connections from controllers, spawns a pawn, orders controller to possess pawn  
  Spawns the "router" which helps certain blueprints find and talk to each other

- **Player Controller: PC_VR_22**  
  Sets variables in Pawn and PC so they can reference each other  
  Checks whether the VR setup has already been completed this instance  
  Forces VR Setup if necessary, saves setup information locally and in the Game Instance, then prompts the Pawn to continue further initialization stuff

- **Game Instance: GInstance_22**  
  Holds the variable for Pawn VR setup once completed so it doesn't need to be repeated.

---

## Pawn: Pawn_VR_22

**SCENES**  
There are many scene components within the Pawn blueprint. These are the most important.

- **RootBox**  
  This represents the origin of your VR playspace, in realspace. If it moves, everything about your pawn moves with it, including tracked components. For 360 locomotion, this is the component that moves. It also attaches itself to the ground so that if the ground moves, you follow it. Transforms applying to the "actor" are the same as applying them to the RootBox.

- **VROO Box**  
  The VR Origin Offset is a child of the RootBox and a parent to the tracked components. While the RootBox is the VR Origin in real space, the VROO acts as the Pawn's origin in virtual space. Separating these two ideas allows us to "recenter" the pawn without invoking XR API functions that may work differently on different systems. This is the scene whose rotation matters for the pawn. It is offset for minor adjustments or recentering.

- **Camera**  
  This is a tracked input; it automatically updates its transform (relative to the VROO) to match HMD movement in realspace. It is not moved manually; the RootBox or VROO, or other scenes that help track the player's location, are moved around it.

- **Motion Controllers MC_Left and MC_Right**  
  These are tracked components. At this time, the template uses a separate mesh for left and right hands (the alternative is to invert the scale on one mesh). There are two sets of hands for each motion controller, a "ghost" hand and a "solid" hand. The ghost hand always shows the player where his hand is relative to the camera 1:1 in realspace and virtualspace, and is only visible to the owning player. The solid hand shows where the pawn's hand is as far as the game is concerned. The solid and the ghost hands are typically in exactly the same spot, but there may be some difference due to collision or item interaction. The ghost hand does not collide or interact.

There are many other scene components. They mostly serve as visualization aids, and common named reference points. The help position the body, determine player height, etc. I will mention them as they come up.

**EVENTS**

- **StartUp**  
  The BeginPlay lives on a separate event graph with the pawn blueprint called "StartUp." It waits until the pawn is properly possessed by the player controller before running. Mostly it initializes variables, takes note of initial tracking data, and moves some scene components to their starting positions. When complete it toggles "Common Setup Complete" (distinct from VR Setup), though it may wait to make sure tracking is working properly.

- **VRMenu**  
  This event graph holds all of the VR-specific setup functions, like figuring out the relative positions of the player and the floor (both in realspace and virtualspace). This is probably the most complicated part of the blueprint, but probably the most solid; I don't expect many changes to be made here. There is also currently a VR menu living in this event graph that lets the player control the setup process and override certain settings during gameplay. I hate the way this is currently implemented and will be moving it to a separate blueprint that allows for better ergonomics for the player and more flexibility for the developer. Do not get attached to the menu or add too much that depends on it.

- **EventGraph**  
  This is where "Tick" lives, but it won't go until the common setup is complete.
  
  - **"Pre"**  
    The first thing it does on tick is update the real and virtual floor location for later use. Control Inputs are copied into variables and processed for later use. Some of them are used to execute actions immediately but they shouldn't be.
  
  - **"During"**  
    Everything to do with Hands then fires, it has its own Event Graph page. Hands will include item interaction, collision, climbing locomotion, and sometimes physics. The appropriate "movement sequence" fires, depending on whether the pawn is walking or floating. These each have their own page. This mostly includes locomotion stuff, collision, physics, and whether to switch to a different movement regime. The pawn is moved directly.
  
  - **"Post"**  
    Body parts are moved to the appropriate spot. The effects of any collisions are applied. Various velocities are recorded for future use.

Everything else is details. There are a lot of details but you should have an idea of where to find them now.

---

## Incomplete list of potential Pawn behavior

- Reset Orientation and Position without XR API call
- Float in zero g, moving with thrusters or by pushing off from walls with hands, or from physics collision, bouncing off walls
- Switching from floating to walking when coming into contact with a floor of approximately the same normal as the pawn
- Climbing walls, vaulting over walls
- 360 Locomotion over various terrain, up normal can be snapped in various ways or follow the ground
- Edge detection to prevent pawns from walking off cliffs both in 360 and roomscale
- Collision detection that prevents pawns from walking through things but doesn't interfere too much with locomotion
- The ability to lean over a counter or something without collision detection bouncing you back right away
- Pawns bumping their heads on the ceiling will be forced to crouch until they can fit
- Jumping by realspace jumping (or a quick bounce on tippy toes), with the jump velocity modified by hand swinging and prior pawn speed and floor velocity
- I think the direction the pawn is looking relative to the movement direction also does something
- A walking pawn will move along with the floor if it is also moving/rotating
- Smooth hand animations that depending on what controller buttons are pressed
- Hands can collide, push, climb walls, grab items, interact with held items, push buttons
- Pawn can be "seated" with constraints

---

## Buttons

There are several blueprints and components associated with buttons, including the pawn. The reason it's so complicated is that I want to be able to dynamically reassign the target functions from a huge library and changing actors, and I want to cut down the amount of work it takes to set that up.

**BUTTON COMPONENT**  
This is the actual button. It moves when it's pressed, shows its state, and can trigger a function somewhere. There are many things set up by
