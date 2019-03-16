---
title: "Writing a game for the Commodore Amiga"
date: 2019-03-13T21:49:32-05:00
---

{{< figure src="modsurfer.png" title="A rhythm game for the Commodore Amiga. Tiles, colored by pitch, synchronize with instruments in the music. Missing a tile silences the instrument." >}}

[ModSurfer](https://amigageek.com/modsurfer/) ([video](https://www.youtube.com/watch?v=Jl_jp1rkMa8)) was my entry to the 2018 English Amiga Board game development competition. This six month contest rode a wave of renewed interest in the [Commodore Amiga](https://en.wikipedia.org/wiki/Amiga), a series of computers which originated in the 1980s and succeeded the very popular 8-bit Commodore 64.

The game targets the Amiga 500, an entry level model launched in 1987. Sporting a 7MHz Motorola 68000, custom chips for audio, graphics, and blitting, and 1MB of RAM, the Amiga is an interesting platform to program. In this post I describe the game's architecture and some of the challenges I faced during development.

[Source code](https://github.com/jcornwall/modsurfer) is available for the curious.

## Development environment

The [amiga-gcc](https://github.com/bebbo/amiga-gcc) cross-compiler, a fork of gcc with a m68k-amigaos backend, is used for C compiling and linking. C proved to be a productive choice for most of the game logic. Performance critical code was written in assembly using the [VASM](http://sun.hasenbraten.de/vasm/) cross-assembler (included with amiga-gcc). VASM has a more ergonomic macro language than GAS, though I also used the latter for some inline assembly. The build system is implemented with a [Makefile](https://github.com/jcornwall/modsurfer/blob/master/Makefile).

{{< figure src="devenv.jpg" title="My development environment on Linux. Emacs code editor (left) and FS-UAE emulator (right) running on the keyboard-driven i3 window manager." >}}

Testing and debugging was done on an Amiga emulator. [FS-UAE](https://fs-uae.net/) and [WinUAE](http://www.winuae.net/) both share the same core and provide cycle-accurate emulation of the 68000. Most of the custom chipset is faithfully emulated. Both emulators provide the same debugger (or "monitor" in retro terminology) with memory/register inspection, disassembly, breakpoints, and watchpoints.

Debugging symbols (and profiling with gprof) are supported by newer releases of WinUAE but I did not have time to experiment with this feature. Instead, I used printf debugging and exercised the classic programmer's mantra "when in doubt, comment it out" to track down bugs. In a pure assembly project it would have been easier to make use of breakpoints and exception traps.

I also tested the game on a 25 year old Amiga A1200. This revealed a few bugs with registers which had not been programmed correctly. I'm unsure whether these differences stemmed from my emulated Amiga OS configuration, or if the emulator deviated from hardware behavior.

Real hardware also emphasized a display flicker issue when starting and ending the game, which was barely perceptible in emulation. To fix this I had to rewrite a signficant piece of the display code. I recommend testing regularly on hardware.

## Game plan

ModSurfer was inspired by the PC game [AudioSurf](https://store.steampowered.com/app/12900/AudioSurf/), which constructs a 3D track from an MP3 file with visual elements synchronized to sounds. The player moves left/right across three lanes, trying to hit the visual elements to score points. AudioSurf performs audio analysis on the MP3 file to construct its track.

The MP3 format is too large and processor intensive to play on a base Amiga 500 (although it is playable on accelerated Amigas). Instead, most Amiga music is composed in the [MOD](https://en.wikipedia.org/wiki/MOD_(file_format)) format. MOD is the oldest file format in the "tracker" family. Tracked music separates the audio data into short, reusable instrument samples. Timing data and effects are used to construct a song.

This data separation has some key advantages for the game. The same set of audio samples are reused throughout the song, with resampling and effects applied. This limits the amount of audio data that needs to be analyzed. In addition, timing information for every instrument is encoded in the music module.

{{< figure src="protracker.png" title="Protracker, the most popular Amiga music tracker software. Four audio channels can play up to 31 samples with many programmable effects." >}}

The most basic unit of a module is the division, consisting of four 32-bit hexadecimal values. These encode a sample number, sample rate (pitch), and an effect, for each of the Amiga's four audio channels. The MOD player advances through divisions at fixed time intervals, although the interval can be changed through effects.

64 divisions are organized into a pattern. A song consists of a series of pattern indexes, allowing patterns to be reused in different parts of the music.

The ambitious plan I set out at the beginning of the six month development period could be summarized as: identify the lead instrument in each pattern, generate visuals synchronized with the instrument, and silence the instrument if the player is in the wrong lane when the instrument plays.

## The menu

The frontend of the game lets the user select a MOD file to play from their filesystem (floppy or hard disk). Amiga OS has a sophisticated (for its time) GUI subsystem called [Intuition](https://en.wikipedia.org/wiki/Intuition_(Amiga)). Later revisions of the OS supported quite elaborate GUI toolkits. The base Amiga 500, however, has a fairly limited one. I decided to go old-school and create a GUI from scratch, using graphics and input events.

{{< figure src="menu.png" title="Mouse-driven menu with a file browser on the right and music file metadata on the left. The game supports any music file in the MOD format." >}}

Like many computers from the era the Amiga uses a planar bitmap layout. The menu bitmap consists of three separate 320x256 bitplanes. A bit grouping across all bitplanes corresponds to a pixel. Three bitplanes form indices (0-7) into a palette. The Amiga 500's palette records colors in RGB, with four bits per component (0-15).

Menu graphics rely heavily on the Amiga's blitter processor. Line drawing and filling are hardware features. Text is formed by blitting from a character map to the screen. The blitter handles shifts and masks across the 16-bit words efficiently.

Mouse and keyboard input are received through a high-priority input handler. The Amiga OS has a microkernel architecture. Tasks communicate efficiently through unprotected, shared physical memory. One of these tasks is the [input device](https://wiki.amigaos.net/wiki/Input_Device). This task allows programs to receive (and steal) input events from the hardware.

The filesystem is supported by another task, the DOS device. This provides an abstract path hierarchy to hide the details of floppy and hard disk hardware. Amiga almost had a Unix-derived disk operating system, but one of Commodore's (many) failings led to it being replaced by the less desirable [TRIPOS](https://en.wikipedia.org/wiki/TRIPOS).

## Banging the hardware

In contrast to the menu, which is an OS-friendly task, the game itself bypasses all OS abstractions, including task switching. This was common in many Amiga games because the computer's limited capabilities led to a very tight performance budget. Any abstraction overhead or task switching jitter could cause a missed frame.

This brings me to my favorite part of Amiga programming. Here's a snippet of code from the game:

{{< highlight c >}}
custom.bltafwm = (desc ? right_word_mask : left_word_mask);
custom.bltalwm = (desc ? left_word_mask : right_word_mask);
custom.bltadat = 0xFFFF;
custom.bltbpt = (APTR)src_start_b;
custom.bltcpt = (APTR)dst_start_b;
custom.bltdpt = (APTR)dst_start_b;
custom.bltsize = (copy_h << BLTSIZE_H0_SHF) | width_words;
{{< /highlight >}}

"custom" is a structure containing 16-bit words, each representing a custom hardware chip register. The custom symbol is provided by libamiga.a and evaluates to absolute address 0xDFF000. This is the physical base address at which many hardware registers are mapped.

The C code above is equivalent to BASIC "pokes" on 8-bit systems. I find it delightful to program at such a low level in a structured language.

In practice, most register poking is done indirectly by the Amiga's "copper" (coprocessor) chip. This is a programmable engine which synchronizes with the raster beam to allow precisely timed register changes. Its command stream looks like this:

{{< highlight c >}}
0x8301fffe        //  Wait for vpos >= 0x83 and hpos >= 0x00
                  //  VP 83, VE 7f; HP 00, HE fe; BFD 1
0x0108000e        //  BPL1MOD := 0x000e
0x010a000e        //  BPL2MOD := 0x000e
0x01020099        //  BPLCON1 := 0x0099
0x01820000        //  COLOR01 := 0x0000
0x01840b30        //  COLOR02 := 0x0b30
0x01860000        //  COLOR03 := 0x0000
0x01880000        //  COLOR04 := 0x0000
0x018a0704        //  COLOR05 := 0x0704
0x018c0002        /*  COLOR06 := 0x0002
{{< /highlight >}}

The first line waits for a specific scanline of the display. The following three lines configure a horizontal shift for the bitmap. The remaining lines modify the palette to select 6 arbitrary colors per scanline.

ModSurfer exploits this processor for pseudo 3D (explained later) and palette changes, to get more than 50 different colors with an 8 color palette.

## Audio analysis

Two criteria are used to identify lead instruments, both varying per-pattern: average pitch and play count. The game assigns a score to each instrument played in the pattern and selects the best candidate.

Play count is easily derived from the pattern data. It is used to avoid instruments with few notes per pattern. This keeps the gameplay interesting even if the pitch suggests it might be a lead instrument. This forms one part of the instrument's score.

Pitch is more challenging. There are two factors to consider: the frequencies in the sample data, and the sample rates chosen in the pattern.

Resampling is an Amiga hardware feature which allows the pitch of an instrument to be shifted up or down. This is how different notes (typically within an octave) are played. It's easy to account for by computing the average sample rate played by the pattern. A larger number implies a higher pitch.

The sample rate alone, however, cannot distinguish a bass instrument from percussion. Both instruments may be recorded at the same sample rate, but percussion will have higher frequencies in its sample data. To account for this the game calculates the dominant frequency in the sample data of each instrument.

{{< figure src="spectra.jpg" title="Frequency spectra for bass (left) and percussion (right) instruments. Dominant frequencies help to identify the kind of instrument." >}}

A Fourier transform is applied to the sample data to derive its frequency spectrum. 512 samples from each instrument are passed through a 16-bit fixed-point FFT. The highest peak denotes the dominant frequency. The algorithm completes for all instruments in under 5 seconds on large MODs. This is fast but not particularly resilient. I would have preferred to average multiple peaks.

Finally, the dominant frequency is combined with the per-pattern average sample rate. This is compared to a fixed pitch approximating lead instruments, between bass and percussion. A closer match gives a better score. It isn't infallible but is much more effective than I expected.

## Pseudo 3D

Fast rhythm games need responsive visuals and input. To achieve this I set a hard frame rate target of 50 FPS (matching the 50Hz PAL standard). I also wanted the game to have 3D graphics. A high frame rate, however, is very difficult to achieve in 3D games on the Amiga 500.

The solution employs some [pseudo 3D](http://www.extentofthejam.com/pseudo/) trickery, common to many games of the Amiga's era. Perspective is simulated by shifting scanlines to the left or right, with larger shifts nearer the camera. Scanline shifting was fast and commonly available in computers of the time. 3D calculations are done in fixed-point, and perspective divide is implemented with lookup tables to minimize CPU load.

{{< figure src="shifting.png" title="Camera centered (left) and moved to the left (right). Scanlines are shifted to the right to simulate perspective. The shift is larger nearer the camera." >}}

The display is designed around a single bitmap, showing a road rendered in perspective with the camera centered. This bitmap is not modified during the game. All visual animation is achieved through scanline shifting, palette changes, and hardware sprites (for the ball). The diagram above illustrates perspective scanline shifts as the camera moves to the left. Shorter at the top, longer nearer the camera.

This method is fast and effective. Its main downside is that the centered, perspective rendered bitmap has some "baked in" sub-pixel error. Scanline shifting also introduces some sub-pixel error. When these errors combine they can become super-pixel, leading to jaggy artifacts. Still, the effect is quite convincing.

As alluded to earlier, the copper chip waits for each scanline and programs in the desired shift. The CPU calculates these shift values during every frame. BPLxMOD is the word (16-bit) shift for the next scanline. BPLCON1 is the sub-word shift for the current scanline.

{{< highlight c >}}
0x8301fffe        //  Wait for vpos >= 0x83 and hpos >= 0x00
                  //  VP 83, VE 7f; HP 00, HE fe; BFD 1
0x0108000e        //  BPL1MOD := 0x000e
0x010a000e        //  BPL2MOD := 0x000e
0x01020099        /*  BPLCON1 := 0x0099
{{< /highlight >}}

## Animating the tiles

Visual tiles represent notes of the lead instrument. As the music proceeds the tiles approach in different lanes. These would be quite challenging to display as sprites, due to their number and varying scanline shifts. They would also be expensive to clear and redraw on every frame.

{{< figure src="bitmap.png" title="The bitmap, which remains unmodified throughout the game. Palette values for colors 1-6 are changed by the copper on every scanline." >}}

Instead, we again exploit the copper chip. The diagram above shows the actual bitmap used by the game, in false colors. On every scanline the copper reprograms the palette for colors 1-6. When a tile should appear in a lane its color is set according to the instrument's pitch. When it should not appear the color is set to black. The road stripes and VU meters are colored in a similar way.

{{< highlight c >}}
0x8301fffe        //  Wait for vpos >= 0x83 and hpos >= 0x00
                  //  VP 83, VE 7f; HP 00, HE fe; BFD 1
...
0x01820000        //  COLOR01 := 0x0000
0x01840b30        //  COLOR02 := 0x0b30
0x01860000        //  COLOR03 := 0x0000
0x01880000        //  COLOR04 := 0x0000
0x018a0704        //  COLOR05 := 0x0704
0x018c0002        /*  COLOR06 := 0x0002
{{< /highlight >}}

This is a very efficient method to change large parts of the screen on every frame. The CPU recalculates the colors to appear on each scanline and writes them into the copper's program. This allows the copper to make precisely timed color changes, as the raster proceeds down the screen, without tying up the CPU. The program is "double-buffered", changing an unused copy while the copper runs.

## Color cycling

The final visual element is the ball. This graphic was inspired by the [Boing Ball](http://amiga.lychesis.net/special/ColorCycling/Boing.html), a famous early Amiga animated demo and Amiga's mascot. Its rotation animation is achieved by changing colors in the palette. The sprite bitmap does not change.

In ModSurfer the ball is implemented with four sprites. Two 16x32 pixel 2-bitplane sprites combine horizontally to form a 32x32 ball. Two more sprites combine with these to make a 4-bitplane, 16-color "attached" 32x32 sprite. This large number of colors is necessary for the color cycling effect.

The sprite bitmap is only changed to show different frames of rotation left/right. Rotation around the X axis (forwards) is implemented through color cycling, to save memory. During each frame the sprite palette is shifted, moving each color forwards by one place and wrapping the last one back to the beginning.

{{< figure src="balls.png" title="Cycling colors in the palette simulates forward rotation. The color gradients encoded in the bitmap are shown to the right." >}}

14 colors are used in total, 7 red and 7 white. Solid areas of color are in fact gradients of these 14 colors, as illustrated in the diagram above. As the palette shifts, the red/white boundaries move, giving the illusion of forwards rotation.

Gradients in the sprite bitmap are spaced so that they move faster where the ball is further from the camera. Frames of left/right rotation are generated by a miniature ray tracer at build time, which also encodes the color gradients.

## Audio playback

MOD interpretation and audio rendering are largely handled by the open source [ptplayer](http://aminet.net/package/mus/play/ptplayer) library. This saved a lot of time during development, allowing me to focus on the gameplay and visuals. I did, however, make some small changes to the library to assist gameplay.

ptplayer runs in a timer interrupt to support MODs with fine timing requirements. This made it challenging to synchronize with the visuals. I settled on a method which resets the camera position each time ptplayer advances one division in the MOD. Frames in-between use an interpolated position calculation, based on the current speed, for smooth motion. In practice this worked flawlessly for music of all speeds.

To make the game more interactive I wanted to silence an instrument when the player missed the corresponding tile. The other instruments would remain audible. I achieved this with a simple hook into the ptplayer code, providing the number of the next sample to be silenced. If the player hits the tile in time then this value is reset.

## Conclusion

I really enjoyed the time I spent on the game development contest. ModSurfer worked out much better than I'd hoped, given its uncertain algorithmic basis. It's fun and quite addictive with faster music. Learning to program the Amiga hardware was an indulgent return to my programming roots. I understand why we moved towards a world of software abstractions but, my goodness, the old world was so much fun!