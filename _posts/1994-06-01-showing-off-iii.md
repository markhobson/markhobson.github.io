---
title: Showing Off (Part III)
description: In the last of his three-part series, Mark Hobson reveals how to plot static and moving sprites and add music to your demo.
date: 1994-06-01 00:00:00 +0000
last_modified_at: 2021-07-03 00:00:00 +0000
header:
  overlay_color: "#368"
excerpt: In the last of his three-part series, Mark Hobson reveals how to plot static and moving sprites and add music to your demo.
---

_This article first appeared in [Acorn User June 1994](https://archive.org/details/AcornUser143-Jun94/page/n89/mode/2up)._

In last month's article, we looked at two more modules that can be implemented into our ever-growing demo system. Unfortunately, this month sees the end of the series, but luckily I have hand-crafted three more mind-numbingly amazing modules to top off the demo.

So to ease you in this month, I will start by showing how large sprites can be plotted in a rather fast and cunning way.

![Screenshot 1](/assets/posts/1994-06-01-showing-off-iii/screenshot1.png)
_The finished routine as seen on the Acorn User Cover Disc_

## Background sprites

Most demos tend to have a large sprite as a backdrop, usually a logo or something similar, either stationary or moving around in the background.

There are two ways that one can plot sprites, either by writing a flexible sprite routine that can handle any sprite under any circumstances, or by writing several fast but specific sprite routines.

By a specific sprite routine I mean, for example, a piece of code that can only handle a certain size sprite, with no sprite-clipping facilities.

An average flexible sprite routine would plot a sprite byte by byte, row by row with masks if necessary, whilst checking whether to clip the sprite. This method is too slow when writing speed-critical programs such as demos.

By knowing the width and height of the sprite to be plotted, we can speed the program up considerably. Because the width is a known value, the code to plot one line of the sprite can be optimised by removing the need for a loop, and replacing it with the minimum number of `LDM` and `STM` instructions required.

We can also remove the loop to plot each line of the sprite, by assembling the code to display one line the necessary number of limes. This will be executed faster because it removes the instructions to decrement the count, compare it with zero and branch if not equal.

For large sprites, these changes can be highly noticeable, not only because it removes three extra instructions for each line plotted, but because a branch instruction breaks the pipelining process of the Arm chip.

Usually the Arm works on three instructions at a time: processing one, decoding the next and fetching the third from memory. This requires the instructions to occupy consecutive words in memory. Every time the program branches, the processor must 'discard' two instructions and start again, with a consequent time penalty.

The module on the cover disc displays the _Acorn User_ logo; a sprite with dimensions of 236 by 69 pixels. 236 pixels is 59 words of four pixels - seven blocks of eight words with one block of three extra.

![Figure 1](/assets/posts/1994-06-01-showing-off-iii/figure1.png)
_How to plot a fixed-width sprite to the screen_

The program can be altered to display different size sprites, because I have used a macro function to assemble the fastest code possible for a given width, breaking the width down as above.

As the module is introduced into the demo, the sprite appears to emerge out of the water. This is achieved by changing the destination address of the sprite each frame.

The module supplied doesn't support a method for the sprite to be moved each frame. This could quite easily be rectified, by building a table of destination screen offsets through which the program would cycle.

A possible table could allow the sprite to move vertically in a sine wave fashion. Remember though, that the screen offset must be word-aligned, thus limiting horizontal movement to steps of four pixels at a time.

## Bouncing balls

This module allows a number of fixed-sized sprites to be bounced within a bounding box. There are three main stages in creating such a module:

1. Deciding upon what data structure is needed
1. Plotting the sprites with masks
1. Moving the sprites whilst applying gravity

Firstly, the data structure required must be considered. For a bouncing object, we need to know not just the screen position of the sprite, but the *x* and *y* coordinates, as they will need to be changed independently later on.

Obviously, the velocities in both dimensions must be known to calculate the next position for each frame.

Because of the way the balls are introduced, the program also needs to know the vertical velocity after the object has hit the ground; this will be covered in more detail later. Therefore, each object will need the following data:

* *x* and *y* position
* horizontal and vertical velocities
* sprite number
* vertical velocity after bounce

and these will be stored in a table when the module is compiled.

To display the sprites, each frame requires a loop to scan through the table, plotting the necessary sprite at the given coordinates. The screen address is calculated from the *x* and *y* coordinates (in pixels), by multiplying *y* by 320, and adding *x* to the result.

In assembly, this can be performed using shifts, which are faster than multiplication:

```
screen offset = (y * 3Z0) + x
= (y * 256) + (y * 64) + x
= (y * 1<<8) + (y * 1<<6) + x
= (y << 8) + (y << 6) + x
```

Plotting the sprite at this address uses the same method as the logo module does, but with masks. So now all we need to concern ourselves with is the problem of creating a realistic bounce effect.

Consider bouncing an object within a bounding box without gravity. Every frame, the horizontal velocity - positive to the right, negative to the left - would be added to the *x* position, and the vertical velocity - positive downwards, negative upwards - would be added to the *y* position.

When the object reached one of the edges of the bounding box, the corresponding velocity would be inverted: negative to positive and vice versa. Now we must simulate a gravitational pull upon the object. For this effect, a constant value must be added to the vertical velocity each frame; this will produce a parabolic velocity.

The constant value is a measure of the strength of gravity, which is a measure of acceleration towards the base of the screen.

Purely for effect, the module allows the ball's initial position to be above the top of the screen, so the ball can be introduced by scrolling down into the screen.

This has one side-effect; when the ball's vertical velocity is inverted after colliding with the ground, it would eventually return to its initial height which is off the top of the screen, and therefore keep appearing and disappearing from view.

Correcting this involves resetting the vertical velocity to a preset value - held in the table - after hitting the ground. The preset value is, in effect, a measure of how high the ball will bounce, regardless of its initial position. This may go against the rules of physics, but it sure does look good.

## Music

Just about all demos these days tend to have some sort of music playing to set the atmosphere. One of the most common forms of widely available music are *SoundTracker* files.

In case you do not already know, *SoundTracker* originated as a public domain program on the Amiga which allowed people with little musical knowledge to create reasonable tunes. Fairly recently, several reincarnations have been produced for Acorn machines which allow the creation of music.

For a long time, stand-alone programs to play *SoundTracker* music have been readily available in the public domain. An exact explanation of how the play routine works would be out of the scope of this article, therefore I have provided a module which can be used within your own demos, without your needing to know the techniques behind it.

The module was developed from the source code of a play routine written by The Serial Port several years ago. It has been adapted to work just like any other module does under the modular demo system, so it does not require an extra Risc OS play routine module.

When compiled, the module takes a *SoundTracker* file called `Music` within its directory, and outputs a module composed of the play routine code and music data.

When the demo initialises and finalises the module, the music starts and stops respectively, with each frame the next part of the music is played, therefore it docs not run on interrupts as the Risc OS module versions do.

If you do wish to understand how the module plays the music, the source code is fully annotated.

![Screenshot 2](/assets/posts/1994-06-01-showing-off-iii/screenshot2.png)
_The look of the demo can be changed by editing the sprite files_

The music module is not actually used within the demo on the cover disc because the processor cannot handle all six modules running at 50 frames per second.

By limiting which modules are run, music can be played alongside the demo. Arm 3 machines should be able to cope, as well.

And that, I'm afraid, is the end of our demo writing series. I hope that it has helped to clarify the complexity surrounding the process of creating your own demos, and has presented a system to simplify the development.

Don't forget, if you create any modules or demos for use with the system, send them in. You never know, the best ones may even get published on the cover disc. Until then, happy demoing.
