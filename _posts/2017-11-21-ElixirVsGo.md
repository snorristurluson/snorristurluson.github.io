I knew when I set out on my [exsim project](../LargeScaleAmbitions)
that it was ambitious - mixing three programming languages in one
project, two of which were new to me. Elixir is very promising for handling lots of connections, and in general
running concurrent tasks making good use of multi-core processors, which is why I found it interesting.

I decided to use Go for implementing bots running against the server as that is another language I've been meaning
to learn. With my background of using C++ and Python, Go is very easy to get to grips with. The syntax is C-like,
the channel and go routine concepts are similar to Stackless 
[channels and tasklets](https://www.gdcvault.com/play/1020578/Multitasking-with)
as we use on EVE Online.

I must admit that I've been struggling with Elixir, and find it hard to keep myself motivated to continue that
struggle. I don't want to give up on it at all - I'm just realizing that picking up Go and Elixir simultaneously
is more than my brain can handle.

One of the things I wanted to do when starting out on this project was to compare Go and Elixir for implementing
the server. My plan was to do the server in Elixir, get some basic functionality going, some bots, debugging viewer
and a simple client, then redo the server in Go and see how it would compare, both in ease of implementation and
performance.

I have a very basic server running in Elixir, as described in my [last post](../WaitingForAnAnswer).
It takes in commands from the connecting player (bot) and routes it to the physics simulation, then gets the
state of the physics simulation and routes it to each player. The basic functionality that is still missing is
some sort of weapon and damage calculations - a very important part of a space fight simulation.

Instead of tackling that bit next in Elixir, I went ahead and implemented the server functionality so far in Go.
Once I have the full (well, basic set) functionality implemented in Go I'll consider revisiting this in Elixir.
As always, the source is up on GitHub: [https://github.com/snorristurluson/gosim]()

## Go vs Elixir
So, now that I have implemented the same feature set in both languages, how do they compare? First impression,
this was much easier to do in Go. To be fair, the overall architecture is the same as when I did it in Elixir,
but then again, I had a pretty clear picture in my mind before as well.

Once I had everything working the same way as before, I also found that the performance in Go was much better than
in Elixir. I sort of expected that straight-line performance of Go (compiled) vs Elixir (interpreted) would be
better, but I also expected Elixir's concurrency handling to make up for it.

When gauging the performance, I'm running with 500 bots on my MacBook Pro. Everything is running on the same machine,
the physics simulation, the server and the bots.
 * In Elixir, I'm seeing approx. 200ms per tick, and the Activity Monitor shows CPU usage for Beam at 150%
 * In Go, I'm seeing approx. 20ms per tick, with CPU usage at 12%
 
Of course it's more than likely I'm doing stupid things in Elixir and the performance could be improved,
but at this point I would rather continue in Go and see where that takes me on this adventure. If it gets me
multi-core performance gains without the hassle, I'm happy - if I run into brick walls, I'll look forward to
revisiting Elixir.

## The Go implementation
The main program starts a loop that listens for connections on a socket. For each connection made, a go routine
is spawned to handle authentication. Once authenticated, a Player structure is created, along with a Ship that
is added to a Solarsystem. The Player then enters a loop (as a go routine) that handles incoming commands. Those
commands are queued up on the Ship, to be handled in an orderly fashion by the Solarsystem.

The Solarsystem also spawns a go routine for ticking the system. On each tick, any queued commands from Ships
are sent to the physics simulation, the simulation is stepped and the resulting state retrieved. The state is
sent to all Ships, to distribute it to the corresponding players. A go routine is spawned for each ship, to
handle the processing of the state concurrently for all ships.

In my opinion, Go rivals Elixir in its concurrency handling - it's just as easy to handle the number of connections
and set up tasks to ensure that the cores are kept busy - for my use case, at least. 