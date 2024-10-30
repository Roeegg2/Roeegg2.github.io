---
title: "The NES Internals Series: chapter 2 - PPU Background"
series-name: "NES Internals"
chapter: 2
date: 2024-04-06 15:13:25 +0300
draft: false
tags : ["nes", "emulation", "assembly"]
---

## Prologue

If you haven't read the CPU writeup yet I highly recommend you go check that writeup first, as I'm building on top of stuff presented there. You can check it out [here](https://roeegg2.github.io/nes-internals-blog/cpu)

When I started learning how the NES PPU works, one of the things that were very difficult for me was that my mindset was that things are done in the simplest most elegant and logical way.
If you think the same, please yeet that mindset into a garbage can, because that's not the case here!

Although there is definitely logic in the way the PPU is working,
It's internals and the way some components work is very awkward - simply because this was the
cheapest/easiest option Nintendo had. Nevertheless, it definitely is cool to see the unique methods Nintendo
used here to try and squeeze the most out of the PPU's hardware.

## Terminology

**for Part 1:**

- **scanline** - the current line the PPU is currently rendering at (the "y" value)

- **dot/cycle** - the current cycle of the PPU, counting from the start of the scanline. So each new scanline,
this value gets reset to 0. The "x" value of a pixel (calculated using `current cycle - 1`, since cycle
0 is idle)

- **VRAM** - short for video memory. This is a volatile area of storage (unlike game ROMS) **inside the NES**
(important to remember for later on when we get into mappers) where information used to determine the color of
each pixel is stored. (in a nutshell, its like the CPU's ram, but for the PPU).

**for Part 2:**

- **Tile** - a unit used to logically divide the NES screen. Each tile is a square of 8x8 _pixels_.

- **Block** - a unit used to logically divide the NES screen. Each block is a square of 2x2 _tiles_
(or 16x16 _pixels_)

- **Quadrant** - (yet another unit) used to logically divide the NES screen. Each quadrant is a square of
2x2 _blocks_, which are 4x4 _tiles_ (or 32x32 _pixels_)

- **Sprite** - a name given to a piece of data to be rendered.
    > **_NOTE:_** This includes both background (the ground, the sky, mario super block, etc) and
    foreground (mostly characters and special effects). This is a different definition that
    _character/foreground sprite_ which is **kind of a sprite**, used to represent a foreground piece of
    rendering data

## Part 1 - Getting a mental image on what rendering looks like

At a fundamental level, the PPU goes over each pixel and sets its color, using certain information we will look at later on.
Here's a simple psuedo-code that shows in a very abstracted way what the PPU is doing:

```psuedo
for each line on screen:
    for each pixel in that line:
        set pixel color
```

The PPU goes over 261 scanlines, each one of them consists of 340 cycles.
each cycle between 1 and 256, it renders a pixel. `256-1 = 255` pixels total each scanline.

```psuedo
scanline 0 <- first cycle of the visible scanlines
scanline 1
scanline 2
.
.
.
scanline 239 <- last cycle of the visible scanlines
scanline 240 <- first cycle of the vertical blank scanlines
scanline 241
.
.
scanline 260 <- the last cycle of the vertical blank scanlines
scanline 261 <- the pre-render scanline

And after scanline 261, it goes back to scanline 0.
Unlike the CPU, which its flow of execution is determined by the code of the game, the PPU repeats this routine over and over and over all the time.
```

### Visible scanlines

As inferred from their name, these are scanlines which are seen on the NES screen.
There are some scanlines which are not visible - meaning a color is not getting outputted.
Why are there such scanlines? Well for a couple of reasons. When we go over these scanlines, I will explain why.

### Vertical blank (vblank) scanlines

These scanlines are not visible - meaning during these scanlines nothing is getting rendered on the screen.
The PPU does essentially nothing. (It only sets up some flags, we will see what flags and when later on)
During these scanlines the CPU can write data to VRAM to setup new data for the PPU to render next frame.

> **_NOTE:_** This also explain why nothing is displayed during these scanlines - the whole point of having these scanlines is to have a time frame where the PPU isn't rendering or reading data so the CPU can safely update the data for the next frame.
And if we didnt have these scanlines, the PPU would read while the CPU is still updating data, and so the PPU would read garbage data from VRAM and the internal registers, which would result in a glitchy unwanted picture.

The CPU **must** access VRAM and internal registers to set stuff for the next frame. (otherwise we would just get a single constant picture to render).

> **_NOTE:_** Although very rare, very few games do access VRAM outside of vblank in a controlled way. They do this to create some special effects on screen.

### Pre-render scanline

These scanlines are also not visible. Other than that, they are very similar to the visible scanlines. (with some other small changes)
As can be inferred from it's name, it's purpose is to prepare the data for the visible scanlines to render.

```
NOTE:
This also explain why nothing is displayed during this scanline - after the CPU updated the data, the PPU needs to make some internal fetches for the next scanline - the first visible scanline.
```

I know this isn't really tangible yet and hard to understand, but things will hopefully make sense later on. Just keep what I said in mind, and if you need a reminder, you can always jump back and read this section again.

### In conclusion

PPU starts at scanline 261, goes over 340 cycles (pixels).
Then switches to scanline 0, goes over 340 cycles (pixels),
Then switches to scanline 1, goes over 340 cycles (pixels),
switches over to scanline 2, etc etc..

It does this forever (well, mostly).

Hope you have some simple mental image on how rendering works at the surface level.
I highly recommend checking out [this diagram](https://www.nesdev.org/wiki/File:Ppu.svg) at the NESdev wiki(Don't get overwhelmed from the amount of information, soon enough everything will be clear. Keep this mental image of scanline and cycles for later. In the PPU, everything is measured in terms of frames, scanlines and cycles).

## Part 2 - Nametables, Pattern tables, Attribute tables, the Palette table and the palette

Okay, this section will be a bit longer.
So last section we talked about how the PPU renders each pixel at a basic level. As we all know, the NES can display a pretty nice range of colors. So how does the PPU know which color it needs to render for each pixel?

Glad you asked!

The NES's way of storing this data is a bit complicated, but after a couple of reads it's easier to understand :)

### Pattern tables

<!-- sprite (just a reminder: Again, this is exclusive to foreground sprite, this is both background _and_ foreground). -->

In short, each entry in pattern tables define the _shape_ of each tile in the screen.
What do I mean by that?

Well, The NES has 2 pattern tables, both of which are stored on the cartridge ROM. (sometimes also referred to as CHR ROM)

The 2 pattern tables reside in $0000-$0fff and $1000-$1fff respectively in the PPU address range.
From that we can infer the size of each pattern table is `$1000 = 4096 bytes`
The size of each entry is 16 bytes.

How does it represent the tile though?
Here's a great diagram I ~~stole~~ borrowed from the NESdev wiki. Down below I will explain what it shows.
This diagram shows how the data for a tile displaying '½' is saved:

```psuedo
(a . represents a 0)

Bit Planes            Pixel Pattern
$0xx0=$41  01000001
$0xx1=$C2  11000010
$0xx2=$44  01000100
$0xx3=$48  01001000
$0xx4=$10  00010000
$0xx5=$20  00100000         .1.....3
$0xx6=$40  01000000         11....3.
$0xx7=$80  10000000  =====  .1...3..
                            .1..3...
$0xx8=$01  00000001  =====  ...3.22.
$0xx9=$02  00000010         ..3....2
$0xxA=$04  00000100         .3....2.
$0xxB=$08  00001000         3....222
$0xxC=$16  00010110
$0xxD=$21  00100001
$0xxE=$42  01000010
$0xxF=$87  10000111

```

Now for the explanation:
We take the first 8 bytes to represent the MSB, and the other 8 bytes to represent the LSB.
We put the bytes "on top of eachother" (byte 0 on byte 8, byte 1 on byte 9, ... byte 7 goes on top of byte 16), and thus we get a matrix of 8x8, with a 2 bit value for each cell.

Let's take byte 0 and byte 8 for example:

```psuedo
byte 0 = 01000001
byte 9 = 00000001
```

When we place byte 9 on 0 we get:

```psuedo
(00)(01)(00)(00)(00)(00)(00)(11)
```

=> we get a 2 bit value!
2 bit value means: `2^2 = 4` so 4 possible values: `0`,`1`,`2` and `3`.

And remember, a tile is 8x8 pixels, so that means we get a 2 bit value for each pixel in the tile!
This value is later used to select the color for that specific pixel. (But its not the only one! the NES uses 4 bits to select the final color, so we get the 2 bits from some where else - we will see later from where)

### Nametables

A question that begs to be asked is: "Okay, so we now know what each 8x8 pixel look like. But how do we order them together? How do we know where each tile is located on the screen?"

The answer is nametables!

If each pattern table entry defines the _structure_ of a _tile_, then the nametables define the _structure_ of the _screen_.
What do I mean by that?
Well, remember how I said each entry in the pattern table is used to represent a tile?

So a nametable is a table of dimensions 32x30, where each each entry size is 1 byte. (NOTE: just sure this is clear: when I say 32x30, 8x8, etc this is a **logical** way we look at the memory. In reality all computer memory is just a contiguous space of bits.)
That means we have `1x32x30 = 920 bytes` in each nametable.
And the PPU has a memory map of 4 nametables, but in reality only 2 of them are real. The others are mirrors. (possibly 4 with some advanced mappers, but this is outside todays discussion)
Each nametable entry represents a tile. How? Well, that 1 byte entry is used to index into the pattern table. So the pattern table essentially associates each tile on the screen with a specific sprite entry in the pattern table. (more on how exactly this 1 byte is used to index into the pattern table, you can find [on this page](https://www.nesdev.org/wiki/PPU_nametables) of the NESdev wiki)

Both nametables reside in VRAM. This is because unlike the pattern table (which are constant), the nametables are dynamic. The CPU read and write from them (and other VRAM memory) using a set of "ports" (which we will look at later). That way the CPU can change whats on the screen for each frame.

That also explains why the resolution is of the NES screen is 256x240. The nametables are a able of 32x30 entries - each entry represents a tile, so we get `32x8x30x8 = 256x240` pixels in total!

### Attribute table

Okay, so now we have the structure of each tile, and the position of each tile on the screen, so theoretically we have all the information required to draw a black-white image. But the NES has a pretty nice range of colors. How do this fit into frame with all we explained before? (pun intended :)

The answer is attribute table!
This is a section that comes right after each nametable, and goes hand-in-hand with the nametable - so much so that often times they are referred to as one unit (which may cause some confusion)
Remember we need another 2 bits to get the final 4 bit color for each pixel? The attribute table is the next component in the process of getting that value.
Each attribute table size is 64 bytes total. Each byte is mapped to a _quadrant_ (which is 2x2 = 4 _blocks_) on the screen.
That byte is divided into 4 parts, giving us `8/4 = 2 bits` for each _block_ (which is 2x2 = 4 _tiles_).
So essentially each group of 4 tiles share the 2 bits.
So now we have 4 bits total for each tile! 2 bits from the pattern table, and 2 bits from the _block_ group it belongs to.

<!-- I'll give an example.
Suppose the attribute byte for some quadran on the screen is `01001001`.
That means that block0 -->

### The Palette and the Palette table

Now, how do we get from a sequence of 4 bits to a full color?
That 4 bit value we created for each tile is used as an index into _yet another_ memory section called the palette table (or palette RAM - they mean the same thing).
The palette table is pretty straight forward - it contains indexes into the NES palette, which is where the actual final colors to be displayed are stored.
The palette table has a size of $20 (32) bytes (Its memory map looks bigger: $3f00 - $3fff, but only the first $1f are real. The rest are mirrors)

That palette table is divided into 2 parts, first 16 bytes, and last 16 bytes. The first 16 bytes are used by the background sprites, and the other 16 for the foreground sprites.
Right now we are only talking about background rendering, so we will ignore the last 16 bytes.

The pattern table is also divided into 4 sub sections, called _palette group_.
We have 16 bytes for the background, so we have `16/4 = 4` colors groups.
The 2 bits we got from the attribute table is used to get the color group, and the 2 bits from pattern table is used to index a specific color from that group. 2 bits means we have `2^2 = 4` options, so 4 options for the color group, 4 options for the specific color.
(in reality we have 3 colors per group, since the first color of each group are the same, but that is for later. You can read more about this [here](https://www.nesdev.org/wiki/PPU_palettes))

Okay. So now we have a byte. What do we do with it?
We use that byte to index into the palette.
The palette has 64 colors, and each color is 24 bits (3 bytes) in an RGB format (byte for red, byte for green and a byte for blue) (I suppose for CRT that is a bit different, but that's out of scope for today)

Thats it!

It takes some time to wrap the head over all of this, so don't be frustrated if it takes some time.
I highly recommend you go read [this page](https://austinmorlan.com/posts/nes_rendering_overview/) written by Austin Morlan. His explanation include great images that help picture this mechanism.

## Part 3 - Rendering mechanism

Here we will dive more in detail on how rendering works.
Remember from before, how the PPU goes over each scanline, and each cycle renders a pixel?
So the position the PPU is currently at on the screen during rendering could be described using coarse x scroll , coarse y scroll, fine y scoll, fine x scroll, and a nametable.

**nametable** - the nametable we want to render from. Remember there are 4 namestables we can address (which in reality only 2 of them are actually present on the NES, the other 2 are mirrors or could be added using mappers - again, we will explain later) so we need `4 = 2^2 => 2 bits` to index a nametable.

**coarse x** - the x coordinate of a tile inside the nametable. Each nametable is 32x30, so we need `32=2^5 => 5 bits` to access a specific tiles x coordinate

**coarse y** - the y coordinate of a tile inside the nametable. Each nametable is 32x30, so we need `32=2^5 => 5 bits` to access a specific tiles y coordinate (NOTE: for the y component, we need to be able to reach values between 0 and 30 so we will need to use 5 bits for the y as well. (less than that and we will get 16, 8, etc. which is less than what we need))

**fine x** - the x coordinate of a pixel inside a tile. Each tile is 8x8 pixels, so we need `8 = 2^3 => 3 bits`

**fine y** - the y coordinate of a pixel inside a tile. Each tile is 8x8 pixels, so we need `8 = 2^3 => 3 bits`

The PPU has 2 internal registers used to keep track of which position the PPU is currently at: `v` and `t`.
`v` is the actual register the NES uses, `t` is only used to change `v`\`s when needed. (We will soon see how and why)

We just explained why we need 15 bits to index a certain pixel, and so both `v` and `t` are 15 bits.
Here is a diagram taken from the NESdev wiki to explain how their bits are divided:

```psuedo
yyy NN YYYYY XXXXX
||| || ||||| +++++-- coarse X scroll
||| || +++++-------- coarse Y scroll
||| ++-------------- nametable select
+++----------------- fine Y scroll
```

And so if you recall from before how the PPU rendering mechanism works, we can infer we increment coarse x every 8 cycles, and increment fine y every scanline, and when fine y overflows (ie we reached 8 pixels) - we increment coarse y.

But where is fine x you ask?
Well fine x is used a bit differently. It is set once, and is used to select a certain pixel from the shift regs. It's value is stored at a register called `x`

What are the shift regs though?
They are a set of registers to which the PPU loads the data to be rendered, and every cycle it shifts them by one, getting the bits it needs.
There are 4 shift regs:

pattern table shift lsb
pattern table shift msb
attribute table shift lsb
attribute table shift msb

each cycle you extract one bit from each (either 00000001 or 10000000, depending on how you choose to shift them) and then are shifted by one.

so using the bits from `pattern table shift lsb` and pattern `table shift msb` a 2 bit value formed and used as the pattern table data (the same value we discussed earlier) and similarly using `attribute table shift lsb` and `attribute table shift msb` a 2 bit value is formed, and used to index the attribute table, which are used to get the other 2 bits need.

So we have 4 bits now, which are used to index into the pattern table, from which you get the final color!

How do the shift registers contain that data? How are they filled?
Each cycle of the visible and prerender scanline the PPU renders a pixel, and does one of the following:

1. fetch a tile index from the nametables
2. fetch an attribute table byte.
3. fetch a pattern table msb byte using the tile index from fetch 1
4. fetch a pattern table lsb byte using the tile index from fetch 1

He places each of these fetches in designated latches, which every 8 cycles are used to feed the shift register.

## Part 4 - PPU registers

How does the PPU communicate with the CPU?
Well, the PPU has MMIO (memory mapped io) ports which it exposes to the CPU, which the CPU and the PPU can read and write from. That way they can communicate with eachother.
That means that the CPU just performs something like:

```assembly
lda #$somedata
sta someport
```

Now, these ports are mirrored by 8. That means that for example a read/write to `$2000` is the same as a read/write to `$2008`, and the same as a read/write to `$2016`, etc

(NOTE: The title is a little misleading, as they are not "registers". They are as I said ports - meaning that writes to a certain port doesnt necessarily just copies the data into a register.)

## PPU CONTROL ($2000)

This port is mapped to an internal register.
This register holds data that affects how the PPU is rendering stuff.
here's a diagram taken from the NESdev wiki:

```psuedo
7  bit  0
---- ----
VPHB SINN
|||| ||||
|||| ||++- Base nametable address
|||| ||    (0 = $2000; 1 = $2400; 2 = $2800; 3 = $2C00)
|||| |+--- VRAM address increment per CPU read/write of PPUDATA
|||| |     (0: add 1, going across; 1: add 32, going down)
|||| +---- Sprite pattern table address for 8x8 sprites
||||       (0: $0000; 1: $1000; ignored in 8x16 mode)
|||+------ Background pattern table address (0: $0000; 1: $1000)
||+------- Sprite size (0: 8x8 pixels; 1: 8x16 pixels – see PPU OAM#Byte 1)
|+-------- PPU master/slave select
|          (0: read backdrop from EXT pins; 1: output color on EXT pins)
+--------- Generate an NMI at the start of the
           vertical blanking interval (0: off; 1: on)
```

Bits `0,1` - the "Base nametable address" specify (as their title implies) the nametable from which we start rendering. They set the 2 bits on the `t` register (which also then sets `v`, but we'll get to that later)
Most of these other flags require further knowledge so we will not dwell too much on them, we will see the need for them later on. For now, just know this register exists. We will see how it is used, and how to read/write from/to it later

## PPU MASK ($2001)

This port is mapped to an internal register.
This register _also_ holds data that affects how the PPU is rendering stuff.
Here's a diagram taken from the NESdev wiki:

```psuedo
7  bit  0
---- ----
BGRs bMmG
|||| ||||
|||| |||+- Greyscale (0: normal color, 1: produce a greyscale display)
|||| ||+-- 1: Show background in leftmost 8 pixels of screen, 0: Hide
|||| |+--- 1: Show sprites in leftmost 8 pixels of screen, 0: Hide
|||| +---- 1: Show background
|||+------ 1: Show sprites
||+------- Emphasize red (green on PAL/Dendy)
|+-------- Emphasize green (red on PAL/Dendy)
+--------- Emphasize blue
```

_Show background_ and _Show sprites_ - each specify to the PPU whether it should render background data and whether it should render foreground data.

_Show background in leftmost 8 pixels of screen_ and _Show sprites in leftmost 8 pixels of screen_ - Same as the former 2, each specify to the PPU whether it should render background data and foreground data on pixels `0,1,2,3,4,5,6,7`.

Similar to PPUCTRL, ignore the rest of the flags until later.

## PPU STATUS ($2002)

This port is mapped to a register.
This register is used to display information regarding the status of the PPU.
Here's a diagram taken from the NESdev wiki:

```psuedo
7  bit  0
---- ----
VSO. ....
|||| ||||
|||+-++++- PPU open bus. Returns stale PPU bus contents.
||+------- Sprite overflow. The intent was for this flag to be set
||         whenever more than eight sprites appear on a scanline, but a
||         hardware bug causes the actual behavior to be more complicated
||         and generate false positives as well as false negatives; see
||         PPU sprite evaluation. This flag is set during sprite
||         evaluation and cleared at dot 1 (the second dot) of the
||         pre-render line.
|+-------- Sprite 0 Hit.  Set when a nonzero pixel of sprite 0 overlaps
|          a nonzero background pixel; cleared at dot 1 of the pre-render
|          line.  Used for raster timing.
+--------- Vertical blank has started (0: not in vblank; 1: in vblank).
           Set at dot 1 of line 241 (the line *after* the post-render
           line); cleared after reading $2002 and at dot 1 of the
           pre-render line.
```

Same as the previous regs, most of these flags require further knowledge about the PPU, but there is one flag that seems familiar - bit 7 - which tells the CPU if the PPU is in the vblank scanlines.

## OAM DATA

used for PPU foreground rendering, discussed at the [foreground rendering writeup](https://roeegg2.github.io/ppu_fg)


## OAM ADDR

used for PPU foreground rendering, discussed at the [foreground rendering writeup](https://roeegg2.github.io/ppu_fg)

## PPU SCROLL ($2005) & PPU ADDR ($2006) & PPU DATA($2007)

These 3 ports go hand in hand with each other.
In order to understand their use, let's dive a bit deeper onto the rendering mechanism:

Have you ever wondered, how do the nametables change?
I mean, we said each entry in the nametables represents a tile, which is associated with a pattern table entry - which is what the game wants to show on each tile.
So that means the nametables should be set by the programmer (since the programmer chooses what he wants to display on screen)
But the programmer just writes code - which gets run on the CPU.
And the CPU doesn't have the nametables on its memory map.
So how does he set their value?

The answer is PPU ADDR and PPU DATA. To put it simply, the programmer loads up an address to PPU ADDR to access on the **PPU's** address space, then places the data he wants to write to that address on PPU DATA, and then ~~(edit here) X~~ writes the data to the specified address.

So the game code will look something like this:

```assembly
; load A with the most significant byte of the address
lda #$2A
; write contents of A to the PPUADDR port
sta $2006
; load A with the least significant byte of the address
lda #$01
; write contents of A to the PPUADDR port
sta $2006

; now we finished writing the address.
; load A with the data to be written:
lda #$30
; store the contents of A to PPUDATA port, so they are written to the address in the PPU's memory space we specified before
sta $2005
```

Now, remember the registers `v, t, x`? I'll now explain more about them, and about their friend - `w`

`x` - not much more to explain about this one, it just stored the fine x value.
`w` - a 1 bit register used to track write sequences. I'll explain what I mean down below:

Remember when I said `v` keeps track of the PPU's current position, and `t` is just used to fill `v` with some new data from time to time? Well they have another use.
The address written to `PPUADDR`/`PPUSCROLL` is placed in `t`. Each write to these ports, swaps the value of `w` (from 1 to 0 or from 0 to 1). As I said earlier, it is used to keep track of a write sequence. Because an address is 16 bits, and because we can only write 8 bits at a time, by using `w` we can track if we are on the first write (when `w = 0`. This is when we set the most significant 8 bits) or the second write (when `w = 1`. This is when we set the least significant 8 bits).
On the second write, we also transfer the data from `t` to `v`.

If you want to toggle between the first and second write, you can perform a **read** on `PPUSTATUS`, which will set `w` to `0`. It's good practice to do that any time you want to write a new address using `PPUADDR`/`PPUSCROLL`, to make sure `w` is indeed 0.

Now you probably wonder: "What is the difference between `PPUADDR` and `PPUSCROLL`"?
Remember I said `t` and `v` have 2 uses? (using as an address to access VRAM, and as a counter the PPU uses to keep track of what pixel it renders right now). That is exactly the difference between the 2.

`PPUSCROLL` - is meant to be used to set `t` (which sets `v` on the second write) when the CPU wants to specify to the PPU what exact pixel it should start rendering from. (in that case, `v` and `t` are viewed from the first perspective I [explained before](#part-3---rendering-mechanism))
This is a reminder on how `t` and `v` look from a **scrolling** perspective:

```
(15 bits total)

edc ba 98765 43210
yyy NN YYYYY XXXXX
||| || ||||| +++++-- coarse X scroll
||| || +++++-------- coarse Y scroll
||| ++-------------- nametable select
+++----------------- fine Y scroll
```

`PPUADDR` - is meant to be used to set `t` (which again, also sets `v` on the second write) when the CPU wants to write to VRAM. (which in this case `t` and `v` are viewed as addresses)
This is how `t` and `v` look from an **address** perspective:

```
(14 bits total)

dcba9876543210

(No special division of bits like in scrolling view, just bits used as an address)
```

And each port sets `t` and `v` differently. `PPUADDR` sets `t` and `v` the way I explained at the start of the section, and `PPUSCROLL` sets them in the following manner:

first write sets the `coarse x` component of `t`, and the `x` register (which is as I explained before "`fine x`").
second write sets the `coarse y` and `fine y` component of `t`.
(As I said before, the `base nametable` is set using a write to `PPUCTRL`)

Here is a simple example for demonstration:
```
; 01111 will be placed in coarse x component of t, 101 will be placed in x register
lda #$7D (%01111101)
; writing the data to PPUSCROLL
sta $2005

; 010 will be placed in fine y, and 11110 will be placed in coarse y
lda #$5E (%01011110)
; writing the data to PPUSCROLL
sta $2005
```

That is probably one of most complicated aspects of the PPU, and a crucial one so make sure you
understand it well.
