---
title: Back to basics
tags: vulkan c++
---
I want to implement my own game engine, using Vulkan for graphics. The primary driver for
this is to learn to use Vulkan, but I also want to try some things in the overall architecture
and infrastructure of a game engine.

I realize that the scope of this is enormous - somewhat overwhelming, in fact, but we'll see
how this goes. The point is to have fun and to learn something - if it never gets finished
then so be it.

## Why Vulkan?
So, why Vulkan rather than DX12? I just want to try to break free from Microsoft. I don't mind
Windows, and I like Visual Studio. I've worked on a Windows game for the last 11 years so I'm
familiar with the whole Microsoft eco system around games.

I also like having choices, and the last couple of years have actually made an effort to work
on macOS or Linux rather than Windows, whenever I can. So for this project I will actually
take it to the other extreme and work primarily on Linux. Vulkan is the only API that runs
on Linux and Windows, and even macOS with the help of MoltenVK.

## Limiting scope
In order to limit the scope somewhat, I've decided to focus on 2D rendering. Much of the
architecture of the engine will be the same, even if I skip the 3D part for now. I also
want to get the point eventually of making some simple games with the engine, and doing that
in 2D means it'll be easier for me to generate some programmer art, rather than trying to
do 3D modelling.

## Where is it?
The source is up on GitHub: https://github.com/snorristurluson/vulkan-sprites

At the time of writing, I've got some basic rendering going, based on the Vulkan
tutorial found at https://vulkan-tutorial.com/

Any feedback is most welcome!

