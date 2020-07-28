---
layout: post
title: "Guide to UE4 performance"
excerpt_separator: "<!--more-->"
feature-img: "assets/img/screenshots/spacebattle.jpg"
thumbnail: "assets/img/screenshots/spacebattle.jpg"
categories:
  - Code Experiment
tags:
  - UE4
  - Code
  - Performance
---


All players like to play games that perform well in their machines, and yet, knowing what to do to have a well performing game isn't something easy to learn. On the GPU side, tweaking the settings of the engine is something well documented, but knowing how to code the game so it performs great is not so clear.

Over and over i have seen people making the same mistakes when developing games on Unreal Engine 4, so i decided to gather the information i've found around performance while working on projects of different scales, and share it with the community.

This guide will be useful for unity developers too, as a lot of them are very similar in both engines. Its focused around CPU and code performance, not GPU.

<!--more-->

### _1: Know your profiler_
The most important thing when optimizing anything is to be able to measure what you are doing. Using profiling tools is necessary if you want to check what is slow and what is fast, and know where to focus your efforts. Unreal engine has multiple of them, the latest of which is Unreal Insights. https://www.youtube.com/watch?v=TygjPe9XHTw

### _2: Setup reliable performance tests_
When you are profiling, its best if you have a somewhat reliable test where you can compare old vs new. A great one, if you are working on gpu optimization, is to move the camera to some locations, and record the performance always on those locations. For cpu code, you generally would try to isolate the objects you want to test into their own map, and simulate them.On multiplayer games, one of the best tools is to use replays to profile, as replays will re-run most of the code and are very reliable.

### _3: Optimize from day 1_
A lot of people keep parroting the “premature optimization is the root of all evils”. This is wrong, and a misquote. If you plan your game to be 60 FPS stable on a console, then you should aim towards that for the entire development process. Its not completely necessary that you keep the framerate at 60 FPS at all times, but its a good idea to target 55 FPS at least 80% of the time even on early stages. This will allow you to make sure that your game will always perform well, and will avoid a crunch at the end of the project.

### _4: Mind your counts_
 When developing a game, You generally have to think about the performance of the things that are being written. A good rule of thumb is to mind how many of “anything” your game has. Most games will have exactly 1 PlayerCharacter, so there is not much of a need to optimize there other than being careful with expensive calls. Blueprints are completely fine in such a case. When you get into the 10 of something, such as “enemies in a brawler”, then you have to be careful with things like animation, and the overheads of blueprints and default Character code. If you have 100 of something, then you need to design it around speed from the start. Slow code multiplied by 100 accumulates very quick. When starting to get into this counts, you should completely avoid blueprints unless its only for punctual events.

At 1000+ counts, not only you shouldn't use blueprint, but you should start to avoid unreal engine Actors and components entirely, and write everything on optimized C++.

### _5: Scene Components are expensive_
Any time an object in unreal moves, it will update the transforms of all of its scene components, and the scene components of children. This code is also not fast, at all, and single threaded. Be very careful with big component hierarchies. Its very common to see big hierarchies on things like vehicles or customizable characters. Consider using the skeletal mesh merge tool in unreal engine cpp codebase to merge meshes into one.

### _6: Avoid blueprint tick_
Blueprint tick has quite a considerable overhead compared to C++ tick. I have measured that around 100-150 of them, connected to nothing, will spend 1 millisecond of cpu time on consoles. If you need ticking, have the ticking in C++. Use timers and/or timelines if you want time based events on blueprints.
### _7: Enable/Disable tick while the game plays_
It's quite common to have objects that need ticking, but not for all the time. Make sure that your Actors and Components only tick whenever necessary. Use SetActorTickEnabled for this.
### _8: Consider lower tick rates_
Not every object has to update 60 times a second (or more!). A lot of them can work with lower tick rates. Like on 7, it's possible to change the tick rate at runtime, so it can be a good idea to have an object tick 2 times a second, until it’s “activated” and goes into full rate.

### _9: Logic Level of Detail_
Related to 7 and 8, in the same way you have level of detail meshes for graphics, you can also have level of detail on game objects. If you are working on a RPG, it would be a good idea to have a vastly simplified Actor for your npcs, and when the player gets close it swaps to the proper NPC actor with good animations and more complex logic. Also consider disabling physics on moving objects that are too far for the player to affect.

### _10: If a tree falls in a forest..._
Consider checking if something affects the player, and just don't do it if the player wont see it. Objects that are far from the player shouldn't do particle effects or sounds as the player wont see or hear them. Avoiding those events will make your game faster.

### _11:  Minimize the amount of components with replicated variables (Multiplayer)_
Unreal engine networking works by gathering what objects are nearby the player, and then it checks them for changed properties. The cost of checking the changed properties depends mostly on the number of objects, and not on the amount of them. It is much faster to have PlayerCharacter with 100 replicated properties, instead of having Player Character + 9 components, and 10 replicated properties each. This goes against good architecture, but there is no good way around it until PushModel networking is stable (unreal 4.25 and forward)

### _12:   Pool your commonly-spawning Actors_
Unreal engine Spawn can be very slow. Using it on things like projectiles or explosions might put too much stress on the garbage-collector, and cause hitches. It's a very good idea to make Projectiles, and other very commonly respawning objects, use an object pool. Robo Recall has an implementation you can check as an example.

### _13:  Be very careful with animation_
Animation trees are a very common bottleneck in unreal engine games. When creating your animation trees, make sure that everything is using the fast-path (little lightning icon), and you minimize the total amount of “blends” you have. Prefer using the state machine instead of blends as it's often faster.
### _14:  Avoid Overlap events on moveable objects_
Every component that has overlap events enabled will perform physics checks every time it moves, and that is quite slow. It is especially bad if you have overlaps on things like weapons, as objects attached to an animated mesh can “move” multiple times per frame. Overlap event main use is for things like Triggers vs the player character, so keep it for that. Do NOT use Overlap events for projectiles or weapons, use direct traces and sweeps there. Calling OverlapActors from your game logic is fine.
### _15: Minimize the amount of movement calls_
Related to 14 and 5. Every time you call a SetLocation or SetRotation, unreal will update the entire transform chain, and if there are physics objects, it will update physics. You should aim to strictly ONE call to SetLocation or related calls per frame on your moving objects. Unreal engine has a SetTransform and SetActorLocationAndRotation calls, that let you change rotation and location in 1 call instead of 2. Calling SetActorLocationAndRotation is twice faster than calling SetLocation and SetRotation separately.
### _16:  Group object logic_
As a general rule of thumb, having 1 “Manager actor” that controls a group of objects is often faster than having them have their logic on their own. If you are going to have high counts of anything it's likely a good idea to centralize them with some “manager” that will update all the objects. Great candidates are things like Projectiles, damage numbers, or some parts of AI logic. This also combines great with the “Logic LOD” idea, as you can decide in that global system if something should update or not. It also allows for things like only updating one-third of the objects every frame. This sea of thieves presentation talks about it https://www.youtube.com/watch?v=CBP5bpwkO54
### _17: Use the Replication Graph (Multiplayer)_
The replication graph is a feature Epic Games added to massively optimize the server performance of fortnite. The replication graph allows you to override the “what objects are relevant to this connection” logic that unreal networking logic has. This will allow you to change network update rates dynamically per zones, or optimize the calculations to know which objects are in replication range. ShooterGame has a full implementation of it where it does some interesting things, such as limiting the update rate of PlayerInfo such as only the data of 2 players will get sent every update.

With good usage of replication graph, you can likely reach 200 or more players in the same multiplayer game. Be careful as it’s a quite new feature and might be buggy.