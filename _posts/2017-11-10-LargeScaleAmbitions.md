---
title: Large Scale Ambitions
tags: elixir go c++ json exsim
---
Learning new things is important for every developer. I've 
[mentioned](https://ccpsnorlax.blogspot.is/2017/04/learning-new-things.html) 
this before, and in the spirit of doing just that, I've started a 
somewhat ambitious project.  

I want to do a large-scale simulation, using 
[Elixir](https://elixir-lang.org/) and 
[Go](https://golang.org/), 
coupled with a physics simulation in C++. I've never done anything 
in Elixir before, and only played a little bit with Go, but I  figure, 
[how hard can it be](https://www.youtube.com/watch?v=uL0ROeZw7wA)?  

## Exsim
I've dubbed this project **exsim** - it's a simulation done in 
Elixir. Someday I'll think about a more catchy name - for now I'm 
just focusing on the technical bits. Here's an overview of the 
system as I see it today:

[![](https://2.bp.blogspot.com/-f9o0Ti1izQQ/WgTLF5KkH7I/AAAAAAAA5Cc/obYptnbxOrchChfqIA017ROOHjfjS4FOQCEwYBhgL/s1600/exsim-overview.png)](https://2.bp.blogspot.com/-f9o0Ti1izQQ/WgTLF5KkH7I/AAAAAAAA5Cc/obYptnbxOrchChfqIA017ROOHjfjS4FOQCEwYBhgL/s1600/exsim-overview.png)

**[exsim](https://github.com/snorristurluson/exsim)** sits at the 
heart of it - this is the main server, implemented in Elixir.

**[exsim-physics](https://github.com/snorristurluson/exsim-physics)** 
is the physics simulation. It is implemented in C++, using the 
[Bullet](http://bulletphysics.org/wordpress/) physics library.

**[exsim-physics-viewer](https://github.com/snorristurluson/exsim-physics-viewer)** 
is a simple viewer for the state of the physics simulation, written 
in Go.

**[exsim-bot](https://github.com/snorristurluson/exsim-bot)** 
is a bot for testing exsim, written in Go.

**exsim-client** is the game client, for interacting with the 
simulation. I haven't actually started this yet - more on that later.  

Note that this is very much work in progress - I'm not describing a 
completed project, my plan is to post a series of blogs about this 
project as it evolves, warts and all. I fully expect to experience some failures and may do stupid things because I'm learning the language - I'll write about those things as well.</div>

So what am I simulating? Well, I've been working on 
[EVE Online](https://www.eveonline.com/) for quite some time now, 
so of course I'm going to do a space simulation. The purpose of 
this exercise is to learn two new programming languages and get 
familiar with the physics system so I don't want to spend time on 
an original game design. I'm not saying I'm going to reimplement 
EVE, but I am going to take some aspects of it and see how they 
could work in Elixir and Go.  

## The server
The core of this project is **exsim** - this is the game server, 
done in Elixir. It's what players (bots) connect to - it maintains 
the state of the world and each ship in it and distributes the state 
to game clients (bots) as appropriate.

To begin with, all communication between components is done in 
JSON format. This may not be the most efficient format, but I've 
opted to at least begin with a text based format. This allows me 
start up the server and connect to it with a simple telnet client, 
typing in the commands on the fly. I chose JSON because it is well 
supported in Elixir, Go and C++ so I don't have to spend time on 
writing a parser.

## The physics server

The physics server is really just a thin wrapper around the Bullet 
physics library. The physics server runs a solar system, with a 
collection of ships in it. Eventually there will be a sun, planets, 
space stations and jump gates (to travel to other solar systems) in 
there as well.

The physics server has a master socket connection to the server, 
and can have one or more viewer connections as well. The master 
connection can issue commands to the server, to add/remove ships, 
set ship properties, step the simulation and get the state. The 
viewer connections get a copy of the state, to enable a debug 
view of the state of the simulation.

The commands are again in JSON format (as well as the responses), 
using [RapidJSON](http://rapidjson.org/) for parsing (and emitting) 
the JSON strings.

## The physics viewer
The physics viewer is a simple 2D viewer implemented in Go, using 
the [Pixel](https://github.com/faiface/pixel) 2D game library. It 
opens a socket connection to the physics server and listens for 
state packets. It draws a shape for each ship listed in the state, 
giving a quick visualization of the state of the solar system. For 
this to work, I'm limiting all movements to the x-y plane - 
eventually this will have to be replaced with a 3D viewer.

## Bots
Botting in a live game is bad, but running bots for testing is 
awesome. I don't want to have to run multiple clients connecting to 
the server to test things - I want to run one program that simulates 
dozens of players. Eventually I want to use bots to load test the 
server, not with dozens, but thousands or tens of thousands of 
simulated players.

So far, I've only implemented one extremely simple bot. It connects, 
sets a random target location to fly towards, waits a few seconds 
and sets a new target location, and so on.

## Current status
The server accepts connections from the bots, adding ships to the 
physics scene, ticks the simulation and broadcasts the state. 
There is no partitioning of the solar system - every ship is 
notified of the state of every other ship. There is also no error 
handling, not even of closed connections. I need to harden that up 
somewhat soon, if only to make it easier to iterate on things.

The next step is to see how many ships I can throw into the solar 
system and see where the bottlenecks are. In some very preliminary 
tests I'm seeing indications that JSON may not be feasible to use 
in any large scale communications, but I need to run some more 
tests before I decide to toss it out.
