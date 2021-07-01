---
title: Showing Off (Part II)
description: Mark Hobson continues his trip into the shady world of the graphics demo with two new modules.
date: 1994-05-01 00:00:00 +0000
last_modified_at: 2021-07-01 00:00:00 +0000
header:
  overlay_color: "#368"
excerpt: Mark Hobson continues his trip into the shady world of the graphics demo with two new modules.
---

_This article first appeared in [Acorn User May 1994](https://archive.org/details/AcornUser142-May94/page/n93/mode/2up)._

After last month's rather heavy introduction to our modular demo writing system, we’ll now take a relatively straightforward look at two new modules.

The first produces a nice water reflection of the screen, and the second one produces the legendary starfield.

![Screenshot](/assets/posts/1994-05-01-showing-off-ii/screenshot.png)
_Scrolling text, reflections and your very own starfield_

## Water module

If you want to simulate reflections in water, then there are several points to be taken into consideration: 

* The image to be reflected is just above the top of the water
* The colour of the reflected image is based around blue
* Usually, the reflected image tends to oscillate upon the ripples

Although these points may seem obvious, it is as well to bear them in mind when designing the program.

Looking at the first point, it's obvious that we need to mirror the section of the screen above the water level in order to produce the reflection. Therefore, any horizontal screen line below water level will be a reflection of the corresponding screen line above the start of the water.

So initially, to produce a simple reflection, we copy the *n*th line above water level to the *n*th line below to give us a mirrored image of a part of the screen. For this demo, I decided that the water should occupy the bottom third of the screen, reflecting the middle third of the screen. 

## Blue shades

After plotting the reflected area we need to change the colour of each pixel to a slightly bluer shade. To do this, we need to understand how the colour value of a pixel is defined in terms of red, green and blue.

As we are writing directly to the screen memory, we have to deal with each pixel in a format which is suited to the video hardware. This is why the colour format may seem slightly illogical.

Below shows the colour format for one byte. The red, green and blue values are held in two bits each, allowing a range between 0 and 3.

![Figure 1](/assets/posts/1994-05-01-showing-off-ii/figure1.png)
_The colour format for a pixel in a 256-colour mode_

If the three colours could each produce four different brightnesses, then there would be a maximum combination of 64 colours. Therefore we have two more bits, called the tint, acting as a third bit for each of the three colours, thus producing the 256 colours.

So, getting back to our water, we want to make the colour of each pixel slightly more blue. Therefore, we could make sure that all the bits within the pixel's colour byte that represent blue are always set.

Basically, this just involves `OR`ing the pixel with the binary value `%10001000`, to force the blue bits to be set.

Now we know how to change the shade of a pixel, we have to apply this to the water area, in this case, a third of the screen. If we did this byte by byte it would take ages, so the `LDM` and `STM` instructions are used instead. What happens is this:

* As much screen memory as possible is loaded into the registers using `LDM`
* For each word, the corresponding four bytes are extracted
* These are then `OR`ed with `%10001000` to shade them blue
* They are then combined back into words
* Finally they are written back to the reflected screen position using `STM`.

## Ripples

Now all we need to complete our reflection is to simulate the motion of waves flowing though it. The method used is fairly similar to how the scrolltext waveforms were produced last month.

A table of y-position offsets in the form of a sine wave is created before the program is run. Then, as each line is reflected, the destination address is offset by the value in the table. As the program proceeds to the next line, it also proceeds to the next entry in the table.

Effectively, what happens is that each horizontal line in the reflection is moved up so many lines, according to the height of the wave at that point. Rotating through the table of offsets in each frame will cause the sine wave to 'travel' through the reflection, so producing a moving wave.

To add that final touch of realism, an element of perspective is added. Without it, waves at the back of the reflection are as big as ones at the front, giving no sense of distance. So all we need to do is create another table, this time containing some scale factors.

The idea is that each line has its unique scale factor which is multiplied by the height of the wave at that point, therefore scaling it down according to how far away it is. This has the effect of waves further away being smaller than closer waves.

As the scale factors grade the wave down, the factor actually ranges between one and zero. Therefore it is a real number and cannot directly be stored in the table, which can only hold integers.

So the real number is multiplied by a constant (usually in the form of *log<sub>2</sub>n)* to turn it into an integer, and then stored. After the program has eventually multiplied by the scale factor, it must compensate for it not being a real number by dividing by the constant (or in our case, by performing an `ASR #n`, shifting to the right *n* bits).

Now that we have added wave motion to our reflection, the reflected image does tend to stretch vertically because of the sine wave. Due to this fact, we can now get away with reflecting the top two-thirds of the screen into the bottom third.

This would normally look wrong as the reflected image is usually the same size as the image, but in this case it works well. One last thing to mention is that as each line has been offset according to the sine table, we cannot rely on every line to be filled with a reflection. Consequently this would leave black lines.

This is easily cured by scanning down the reflection for any black lines; if one is found, simply copy the line above it. The program doesn't actually check the whole line to see if it is black, only the first word on the line, which is cleared before plotting.

If you run the compiled demo on the cover disc, you'll see that the water scrolls onto the screen; this is achieved by the water module itself.

## Starfield module

The starfield is perhaps the most common feature in demos, along with scrolltexts, and it would seem criminal to some for a demo-writing series to completely ignore this tradition. Therefore, I’ve written a very simple parallax right-scrolling starfield.

The basic procedure of any starfield is this:

* Move the stars into their new position
* Display the new starfield

All necessary data about each star, like position, speed and colour is held in a table, which is created before the program is run.

To move a star into its new position, we have to add its speed value to the current screen position, and store it back into the table. Because of the way that the screen memory is laid out, we do not need to check if the star moves off the right-hand side of the screen, as it appears on the left-hand side of the next line down.

The fact that the star appears on the next line is unnoticeable to the eye, so the star only needs repositioning when it overshoots the bottom of the starfield. In this case, the star is moved to the top of the screen.

## Plotting

Now we just need to plot the starfield. Again, the program scans through the table, plotting the pixel value at the screen offset as defined in each entry.

One important point is that if something has already been plotted on screen, the starfield should scroll 'underneath' it. To do this, all we need to do when plotting a star is to check whether the pixel already on screen is black; if it isn't, then don't plot the starfield pixel.

This allows us to draw the screen and then plot the starfield last. The advantage of using this method is obvious.

Say, for example, that we wanted an irregular shaped background sprite with a starfield scrolling underneath it. We could do this in two ways: we could plot the starfield first, and then the sprite on top with a mask; or we could plot the sprite without a mask, and then the starfield underneath.

Obviously the latter method is quicker, because it doesn't require plotting with a mask.

Finally, to allow the stars to move at fractions of a pixel-per-frame, both the position and speed values in the table are initially multiplied by a constant, to allow for real numbers, in the same way as the water module achieves its scale-factor table.

So before the star's screen position is calculated, it is divided (or shifted) by the constant.

One final point is that when the star module is introduced into the demo, the stars appear to fade onto the screen. The star module achieves this effect by incrementing the number of stars from one up to the maximum number of 512.

As the colours of the stars are defined so that the dark stars are at the start of the table and the light stars are at the end of the table, it looks as if they slowly fade in.

## Applying

On this month's cover disc, two more directories have been added to the `Modules` directory: `Stars` and `Water`. These both contain three files, `Info`, `Module`, and `Source`.

`Source` is a Basic program with embedded assembler, which is used to create the `Module` file. `Info` describes which options can easily be changed by editing the start of the `Source` file.

If you want to change some aspects of the demo - turning the sea green for instance - you will need to change the `Source` file and then run it to recreate the `Module` file. Run the `!SetSysVar` utility first so that the program knows where to save the updated module.

You will then need to re-compile the demo using the new module. Run the `Compile` program, which will prompt you for a couple of filenames, and then save your new demo, with an option to run it.

It is best to copy all the demo files to another disc first and work from the copies, in case you accidentally stop one of the modules from working properly.

And that's it for this month. Next month I'll be rounding off the demo by adding modules for background sprites, bouncing balls and music.
