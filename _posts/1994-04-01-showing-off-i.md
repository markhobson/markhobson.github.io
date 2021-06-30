---
title: Showing Off (Part I)
description: Graphical demos crop up everywhere, and in this new series Mark Hobson reveals the secrets behind the scrolling text and bouncing starfields.
date: 1994-04-01 00:00:00 +0000
last_modified_at: 2021-06-30 00:00:00 +0000
header:
  overlay_color: "#368"
excerpt: Graphical demos crop up everywhere, and in this new series Mark Hobson reveals the secrets behind the scrolling text and bouncing starfields.
---

_This article first appeared in [Acorn User April 1994](https://archive.org/details/AcornUser141-Apr94/page/n89/mode/2up)._

Demos are strange programs. They have no purpose except to impress the viewer, but useless as they are, I have yet to meet someone who hasn't got one hiding away on his hard disc, or somewhere in his disc box. 

![Screenshot](/assets/posts/1994-04-01-showing-off-i/screenshot.png)
_Build up your own demo with Mark Hobson's modules_

The typical demo contains features such as scrolling messages, starfields and bouncing balls. Even as demos become more complex, with 3D vector graphics and vector-balls, they are still simply a collection of graphical routines that are combined to produce the final result.

The authors try to squeeze as much processor power as possible out of the machine - this enables even more things to happen in their demos so that the result seems almost impossible.

This complexity that surrounds demos often leads most people to collect and watch them, but not necessarily write them.

## Modular system

In this series I will present a modular system to create demos. Each month I will provide the necessary code to create a module for the next part of the demo, including features such as scrolling text, stars, music, reflections and so on, all to produce the final demo.

If you want to be able to add routines you must be able to write in Arm code, but even if you can't program, you can alter parameters and options to create your own personalised demo.

The beauty of the modular system is that each routine can be written and tested separately from the rest of the demo. Each module is then compiled to produce a module file, and is subsequently added to the others to produce the final result.

The user can choose which modules are needed, customise them and alter their variables if necessary. In this first article we will be looking at how the system works and implementing the scrolltext module.

## Running the demo

On the disc you will find a directory called `Modules`, which contains further sub-directories for each module installed so far. Each of these sub-directories contains the various files which are needed to create that specific module.

Every module needs a Basic program called `Source`. This is the program which compiles the module together, and saves the final code and data, in the same directory, as a data file called `Module`. Any resources that the module needs, such as sprites or tables, can be found in the module's directory.

These module data files can be linked to run simultaneously as a demo by using the `Compile` program. This takes a demo-script text file and installs the stated modules. The demo-script file language is very simple, so simple in fact that there is currently only one command with the syntax:

```
install:<module name>
```

Anything else is ignored. All the modules specified are loaded into memory and a short piece of conditional Arm code is assembled. The order in which the modules are installed is the order that they are called up every frame.

This final product of running `Compile` can be saved to disc as a run-only demo (with file type Utility), and it can also be executed straight away by double-clicking on it. The name of this file on the cover disc is Demo. Run it to show the scrolltext created by this month's installment.

Before the module is compiled, there are usually several variables in the `Source` program that can be altered to change its output, but once it has been compiled, these are fixed.

## File formats

The format of a module file is fairly straightforward. The first three words must be as follows:

```
+0 = offset to initialisation routine
+4 = offset to code to be called every frame
+8 = offset to finalisation routine
```

The offset is defined as a relative address from the start of the module. If the offset equals zero, the corresponding code is not called. The rest of the module is made up of code and data.

When a demo is to be compiled, it always needs a module called `Swop`. This module provides the code to swop between two screen banks for smooth animation. It also clears the current screen bank and gets the new screen address. This address is held in `R12` when the modules are called up every frame, so that they can write directly to the screen memory.

There are only a few guidelines which the modules must follow: they must be written so that they can be relocatable, and they must preserve the registers on exit.

## Sprite files

When a module that uses a sprite file is compiled, it can convert the sprites into an easier format by using the function `FNconvert`. This function is found in the main directory and is used as a Basic library by each of the modules during compilation.

The function takes the sprite file and creates raw mode 13 screen data for each sprite in the file. This is stored at an address specified by the module. The function is called as follows:

```
FNconvert(module_name$,data_address%)
```

It returns the following values depending on the outcome:

```
-1 = sprite file was not found
 0 = sprites converted OK
>0 = number of sprites that were not converted because their width was not divisible by 4
```

The sprites are converted because the raw data is usually easier and faster to deal with than the standard sprite file format.

## Triggers

Most demos use the scrolltext to introduce the different parts of the demo, and the scrolltext module has been designed to cater for this.

Within the scrolltext, special 'trigger' characters can be inserted which do not actually appear on the screen, but set a corresponding bit in a flag. This flag is held in `R11` and can be checked by any module to see if it has been triggered.

The trigger characters can be any lower case letter, and the bits are set as follows:

```
R11 = %000000abcdefghijklmnopqrstuvwxyz
where a bit is set to 1 when that character appears.
```

It is up to the module to decide whether it has been triggered yet and which letter triggers it off. In the source programs that will follow in this series, I have defined two variables to accommodate for this: `trigger$` is the special character that triggers the routine, and `trigger%` is `TRUE` or `FALSE`, depending on whether the code to check for the trigger is to be included.

These same trigger characters can also be used to change the waveform used by the scrolltext - this will be covered in more detail later.

## Scrolltext module

Most people think that writing a scrolltext routine must be difficult, but this is not necessarily the case. There are three main problems to overcome when writing the code:

* Plotting the sprites, including those only partly on the screen
* Keeping track of the position in the text and taking care of trigger characters
* Changing the position of the letters to follow different waveforms

Firstly, let's deal with the problem of plotting a letter. I have decided to use a 16x16 pixel font for the scrolltext which I have found to be a reasonable size.

Considering that we are going to plot about 20 letters onto the screen every frame, the plotting routine has to be fast. For this speed, it is necessary that the screen address that we plot at is word-aligned (divisible by four), otherwise we would have to deal with complicated shifts or use very slow `LDRB` and `STRB` instructions.

Another important aspect of sprite plotting is masks. A mask allows parts of a sprite to be transparent, and for this we need another sprite containing the mask data.

The mask data is a similar sprite containing black and white pixels, where white indicates a corresponding transparent pixel. The mask sprites are created automatically by the `FNconvert` function and are stored directly after each mask-containing sprite.

Because the hexadecimal number for the colour white is `&FF`, we can use this data to `AND` with the screen data. The sprite data can then be `OR`ed into the result to return, back to the screen, what is to be written:

![Figure 1](/assets/posts/1994-04-01-showing-off-i/figure1.png)
_Using sprite masks to superimpose letters over the screen_

We have to repeat this process four times for each word in the line. Therefore we can replace the `LDR` and `STR` instructions with `LDM`s and `STM`s respectively.

Now we can plot one line and we need to repeat this 16 times for the whole sprite. We could set a counter to 16 and decrement it every loop until it reaches zero, but this is too slow. What we can do though, is actually assemble the code the necessary number of times, in this case, 16. This method may take up more memory, but it executes faster.

## Off the screen

So now we know how to plot complete sprites with masks, but we have the problem of sprites scrolling on and off the screen. First of all we need to know by how much the text is scrolling. As we are plotting each letter at a word-aligned address, then it is logical that we should scroll by a word (4 pixels) every frame.

Consider that we have plotted a letter on the left-hand edge of the screen, and we want it to scroll by one word to the left. The actual outcome would be that we are only displaying the three right-hand words in each row, instead of all four. Next frame, it would be the two right-hand words and so on:

![Figure 2](/assets/posts/1994-04-01-showing-off-i/figure2.png)
_Scrolling letters to the left_

So what we need is a variation of the previous sprite-plotting routine that plots only part of the sprite. Now we have three routines to plot letters: one to plot a whole letter, another to plot a letter scrolling off the screen and one to plot a letter scrolling on.

These routines then have to be called correctly to plot the scrolltext onto the screen. For this we need two main variables: the offset that the left-hand letter on the screen is from the start of the text data, and the amount that each letter is to be offset from its normal position.

So for each letter plotted onto the screen, the correct routine is called to plot it at an offset of `(letter_number * 20) - letter_offset` where `letter_number` is 0 for the left-hand letter, incrementing up to 16 for the right-hand letter, and `letter_offset` is the offset in bytes that each letter should be scrolled to the left (ranging from 0 to 16).

Every frame, `letter_offset` is incremented by 4 until it reaches 20, which means that the text has scrolled the width of a letter, 16 bytes of letter plus 4 bytes space. So now we can reset this variable to zero, and move one character forward through the scrolltext.

When the main loop is plotting each letter, there are two main conditions that have to be checked for:

* Has the pointer to the letter exceeded the length of the scrolltext?
* Is the current letter a trigger character?

When the pointer exceeds the length of the scrolltext data, the length is subtracted from the pointer to effectively 'wrap' around to the start. After the pointer has been validated, the corresponding byte in the scrolltext data is checked to see whether it is a trigger character ('a' to 'z'). If it is, the appropriate bit is set in the trigger flag and that character is ignored. The trigger flag is always preserved so that it is not reset every time the routine is called.

## Wavey text

So far we have assumed that the y-position of each letter in the text remains constant. This is fine if you want a straight scrolltext, but many demos include scrolltexts that flow around the screen following interesting waveforms.

To implement this feature is actually quite simple. All we need is a table of y-position offsets, and a base offset from the start of the data.

What happens is that for each letter, a y-position is read from the table and the new screen position is calculated, the offset being incremented as the scrolltext is plotted from left to right. Every frame the base offset is incremented, until it reaches the end, where it is reset to the start of the table.

All we need to do now is create the waveform data. This could be done as a series of loops creating different waves, but I decided to create a waveform script file which the program processes.

This script file contains the necessary information to create each table, which the source program does before compiling the module. Each waveform is described as a series of the following commands in a text file. Commands prefixed with an asterisk are optional.

```
id: = the trigger character that activates this waveform, contained in quotes
length: = the number of entries that this waveform table requires
*var: = the initial value of a variable that can be used within the final equation (see later)
*inc: = the amount by which the variable increments each entry
y: = an equation returning the y-position for the letter, determined as the number of lines from the top of the screen. This equation can contain functions such as SIN, COS, DEG, RAD and so on. Also, it can reference the variable var, initialised with the var: and inc: commands (see above)
```

The following example shows how to implement a simple sine-wave:

```
id:="a"
length:=17*4
var:=0
inc:=360 / length
y:=80 + (74 * SIN(RAD(var)))
```

The lines above define the following: associate waveform with trigger character `a`; set length of scrolltext to the width of 4 screens; initialise variable var to 0 (var will hold the angle used in the sine wave calculation); set increment amount to 360/length (note use of `length`); and finally set the equation to:

```
y = y_centre + (radius * SIN(angle))
```

Several waveforms can be defined within the `Waveform` text file, and the actual scrolltext can activate them by inserting their trigger character.

There is also a program called `TryWave` in the scrolltext module's directory to display waveforms on screen. Simply run this, type in the waveform equations and press RETURN to see the resulting waveform shape.
 
We have now implemented all the necessary features in our scrolltext module to change the waveform and trigger other routines. Next month, we will be adding starfields and reflections to our demo, but until then, happy scrolling!
