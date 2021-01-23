---
title: Unreal gotchas
tags: ue4
---
Here are some things I've run into lately while working in Unreal Engine, where I've spent way too much
time tracking down the issue. By writing these down I'm hoping I'll better remember them in the future,
and maybe I can save somebody else time as well.

##Collision detection for fast moving projectiles
Recently I added projectiles to our game. I created an actor with a static mesh, gave it an impulse
to throw it into the world and checked the 'Simulation generates hit events' checkbox. Nice and simple,
and everything seemed to work. On hit events, I added an explosion effect and destroyed the projectile.

The problem was, every now and then the projectile would just go through the ground without triggering
the explosion - it wasn't getting the hit event.

The solution was to enable Continuous Collision Detection (Use CCD) for the projectile - this is under the advanced
portion of the Collision section for the static mesh component. This took me way too long to find - I was
looking for something called sweep or something like that - Use CCD is not very intuitive.

Note that this applies also when using the ProjectileMovement component, or any time you have something fast
moving.

##Anim montage for short animation clips
I was setting up a gun to fire projectiles using the Gameplay Ability System in Unreal. In the ability,
I had have a PlayMontageAndWait node to play a recoil animation when firing. The problem was that it only
seemed to play a frame or two from the animation. After spending quite some time trying to debug GAS issues
with my setup, I finally started suspecting the montage itself. It was a very simple montage, created from
a single animation, with default settings for everything, and we'd done this for some character animations
without seeing any issues.

I finally realized that this particular animation was only 0.23 seconds long, and the default montage
settings are to fade in/out in 0.25 seconds. After setting those to 0, the montage played just fine.
 