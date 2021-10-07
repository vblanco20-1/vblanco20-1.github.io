---
layout: post
title: "Remastering rRootage for Nintendo Switch"
excerpt_separator: "<!--more-->"
feature-img: "assets/img/screenshots/rrootage.jpg"
thumbnail: "assets/img/screenshots/rrootage.jpg"
categories:
  - Code Experiment
tags:
  - Nintendo
  - Code
  - Performance
---

As my latest side project, ive ported a 2002 open source game to nintendo switch. The game is rRootage, made by Kenta Cho, a japanese developer who has released tons of open source games. Its a bullet hell game that has been ported to many platforms over the years, such as Android, IOS, and PsP homebrew. I chose that game to port because I like bullet hells and because this is a easy project to port due to its minimal codebase and dependencies while it has a license that allows porting to consoles.  In here, i write about how I ported it to Nintendo Switch

<!--more-->

The code license is BSD2, which is mostly like MIT. Licenses like MIT and BSD2 are completely fine to port to consoles, but GPL licenses are completely impossible to port.

In short, the GPL license forces the developer to release the source code of the executables if any small part of the project uses GPL code, and this conflicts with the console NDAs. Meanwhile MIT and other similar licenses only require you to give attribution, but they dont require you to post all the source code, so its possible to use in consoles.

The process of porting the game was done in stages, which i mostly follow in all the porting projects i do as its mostly a general workflow. 

### Stage 0: Analysis
First thing is to grab the source code and look at it, seeing what are its dependencies and what languages it uses. Upon checking the game, I see that the game is mostly written in C, with a few dinosaur era Cpp files (pre-cpp11) and uses SDL 1 and OpenGL (super old) as dependency. I know that SDL is ported to switch as part of SDL 2, and OpenGL runs on nintendo switch mostly as-is, which makes this a specially simple project to port as there are no great blockers. This was done while i was considering to do the port or not, as if this game had dependencies that didn't work on switch i wouldn't have chosen it. If i was doing the port for a 3rd party, its here where i would calculate the cost of doing the port.

### Stage 1: Modernizing the build
The game is from 2002, using makefiles, and a fairly weird build process on windows. The first thing to do is updating the build so that we can easily build it in modern VS2019 natively. 
I used Premake, which is a build system that runs a lua script that generates a visual studio project. This makes it very useful as it can be extended to configure the extra build options that a console build needs. Using CMake is also doable, but those extended configuration settings can be a problem there. 
Started by grabbing the codebase, and writing a premake script that duplicates what the makefile does. In this case, the project was very simple, just building the code inside the src folder and linking to dependencies. So adapting the build was not a problem.  With this i had the game building in modern visual studio. 

### Stage 2: Updating the project to modern libraries.
The game was using SDL 1 and some of its Cpp code did not build in modern cpp settings, giving some issues around std::auto_ptr, so I fixed that and made sure the game builds as cpp14. The update from SDL 1 to SDL 2 was necessary as there is a SDL 2 version for switch, but no such thing for SDL 1. The game was mostly using SDL for windowing and input management, so that was something that wasn't much of a problem to upgrade. What was a issue was SDL_Mixer, an audio extension for SDL1, which did not have a version that worked on nintendo switch. For the time being, i removed the audio from the game to be fixed later.

### Stage 3: Running game on switch.
With the game using SDL2, we can now try to target the Nintendo switch itself. With the premake scripts i had, i was able to configure the vs solution creation to contain the correct configuration to compile on switch. Just compiling isn't enough, it's necessary to build the application package and link to metadata assets, so that needed a few more project options. Once all of that was done, I was able to have the game run on switch, for about an instant as it crashes on early load. 

Looking at the crash, it's caused by the game trying to load some texture assets and the filesystem operations failing. In consoles, you don't have a normal file system, so to load assets you need to use some apis to mount the application data/cartridge data and have the paths be on a specific format. Once the file loading was fixed, the game now opened to the main menu on switch. But with no input, fairly slow graphics, and no sound.

### Stage 4: Making it fully playable.
The input system for the game was using keyboard keys using SDL api. With the SDL 2 upgrade the code was changed, but it was still using keyboard keys. Now its time to implement actual gamepad support so that the game can be played on the switch. Luckily, in SDL 2 there is a multiplatform gamepad api. I implemented support for gamepads using that API on the PC, and it pretty much worked on the switch. There was a bit of trouble with the specific gamepad assignments as attached joycons aren't quite the same as detached joycons, but it worked.
For audio, I had no other choice but to implement my own version of SDL_Mixer using the native SDL 2 audio Apis. This took a while as i'm not used to dealing with audio code, but using a implementation i found in the internet as starting point, i got everything running correctly on switch and pc. 
Pro tip. Do not use headphones when dealing with low level audio code. High chance of bad sounds at full blast.
With sound and input now working, the game is actually properly playable on switch, just a bit on the slow side.

### Stage 5: Optimization
I ran the game through a profiler to see what was going slow. I found that the game logic update was taking sub 1 millisecond, but the graphics side was taking a lot of time on the CPU side. The gpu side had no trouble with it. After all it's a wireframe rendered game from 2002. But whats happening on the CPU side?

The game, as old as it is, was using OpenGL fixed pipeline graphics, doing glBegin(triangles) type code and executing multiple gl calls per bullet . This was so bad that it got deprecated back around 2004. Nowadays you only see that sort of code in terrible university graphics courses. 
Even bigger of an issue is that it was rendering 1 bullet at a time, with each bullet drawing 3 shapes with different pipeline settings. This meant that the game could do pipeline switches up to 1000 times a frame. Both of the issues became a big hit to performance. 
To fix them, I started by rearranging the code. Instead of switching the graphics pipeline 3 times per bullet, I grouped all of the bullets, and drawed them together. This did change the graphics a bit due to the changes to draw order, but the performance increase was very considerable, as it now does 3 pipeline switches when rendering the bullets instead of 1000+. This change made the game run at 60 fps in most levels, but the levels with 500+ bullets on them would still drop below 60 fps.
The next fix was to remove the glBegin(polygon) 6-calls-per-triangle type of rendering, and use vertex buffers like everyone does. This was not quite as simple, as the game was using the matrix stack of OpenGL too to transform the shapes, and most of the shapes were rendered using things like triangle strip, or triangle fan. I began by implementing my own opengl matrix stack, then I implemented my own version of the opengl fixed pipeline calls that instead save the vertex data into an array, and then draw it all at once.

This upgrade made the game run way better on all platforms, and with it, the game now never goes past 5 milliseconds per frame. The game could easily run at 200 fps on the nintendo switch, but the screen can only do 60 fps, so its v-synced and just idling the chips for most of the frame, saving a lot of power. This made the game use very little battery. I also configured the switch power mode into power-saving mode to run at lower clocks.

### Stage 6: Polish
The game now runs super well on the switch and its fully featured. But.. It runs too fast. The game is designed around lagging when the screen is full of bullets, so the game running at 60 fps all cases means that those levels become completely unwinnable. Looking into the game code, i solved it by implementing a variable framerate system.Due to the design of the game, it was pretty simple to add framerate interpolation. Now the game always has a render speed of 60 frames a second, but when the screen is full of bullets, the game logic only updates the bullets 20 times a second (3x slowdown), and for the other 40 frames it interpolates their positions to run smoothly. 
As part of the polish pass, I also improved the joystick handling to add smooth movement. The original game was designed to work with arrow keys so you could only move in 8 directions, but I made it work with full analog control.
I also made sure that savegames work fine with the nintendo switch, as they were broken before with their files not persisting. 

### Stage 7: Nintendo QA
The game is now “done”. Its time to pass it through nintendo and get it approved.
Nintendo, like all other platforms, has a QA process with a set of requirements your game needs to do to get allowed in the platform. Its things like “savegames work fine”,  “input doesnt lock the game” and so on. I had a good read at the requirements from the nintendo documentation, made an Excel sheet with each of the items, and went one by one myself to see if it works or doesnt. This required a few more fixes to the game and a bit more code changes. Once the game was  ready, i sent it to nintendo for testing.  This first testing round ended in failure, as i had a bug related to switching input modes in a weird way i didnt anticipate. I fixed that problem, and sent it again. The second time nintendo gave it the approval and then i could move forward with selling the game.

### Stage 8: Release
As the last stage, i had to build the store pages and get them approved to for release. I had a bit of problem with them so the european release is going to be sooner than the american and japanese releases. Things you need to do for it is writing the store page text, the french translation, building some promotional images, and grabbing some screenshots. In this case i did everything myself, with the french being translated by deepL and doublechecked by a french friend. The promotional images are of a fairly terrible quality, but its just what happens when you dont have an artist to do it. I spent 300 euros in fiverr trying to get some people to do it but i only got garbage.

At the time of posting, the game page is on the nintendo switch e-shop, and will release on 14th of October. American and Japanese versions are coming a bit later due to some mistakes with the timing for review on their store pages, as each region is configured on its own. I do not expect to break even on this project, but i did it as a way of getting experience doing a full release of a project on nintendo switch from beginning to end. Normally im more experienced with sony platforms and steam, as ive released games on those, but i hadnt released a game on nintendo switch yet.  If the game breaks even profit, i will send a part of it to the original author, and port more of his games into the platform. Maybe also porting them to ps4.


