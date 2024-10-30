---
title: "The NES Internals Series: chapter 3 - PPU Foreground Rendering"
series-name: "NES Internals"
chapter: 3
date: 2024-04-13 15:13:25 +0300
tags : ["nes", "emulation", "assembly"]
---

## Terminology

- **DMA** - a component in a computer system which can access the system RAM, and other storage areas directly and move data between them without needing the CPU.
As always, if you want to understand better, go look it up online

## OAM

The OAM (object attribute memory) is the place where sprites are stored. It has the size of 256 bytes, and can store 64 sprites.
Using a short calculation, we get that each sprite entry is the size of `256/64 = 4 Bytes`.

There is another memory area called secondary OAM, which can hold up to 8 sprites (32 bytes).
Data is written to OAM, and then transfered over to secondary OAM, from which the sprites are actually drawn

### Byte 0

This byte contains the Y position of the sprite, counted from the top of the screen.
The value here is actually subtracted by 1, to get the actual Y value of the sprite. I will explain the reasoning behind this later on.

### Byte 1

This byte contains the tile number to be used, to index into the pattern tables.
Its use differs between 8x16 and 8x8:

From the background graphics chapter, we already know we know the PPU can address 2 pattern tables (`$0000-$0fff` and `$1000-$1fff`).
For 8x8 sprites, the decision of which pattern table to use is given by bit 3 of PPUCTRL.
For 8x16 sprites, bit 0 of the this byte, is used to select the pattern table.

### Byte 2

This byte contains some important attribute data about the sprite: palette group used, sprite priority, and whether the sprite should be rendered flipped (both vertically or horizontally)

### Byte 3

The X position of the sprite, counted from the left of the sprite (ie the first pixel)

Thats it! That's how sprites are represented in the NES.


## DMA

Remember I said the PPU copies data from primary OAM into secondary OAM?
Well the PPU has 2 options to perform these copies:
1. Copying using OAMADDR and OAMDATA
2. Copying using DMA


> **_NOTE:_** In contrast to the usual way a the case of the NES, the CPU is actually halted while the DMA is copying data to secondary OAM, so it doesn't have that advantage, but the copy is actually way faster using DMA (it takes 2 cycles to copy a byte using DMA, and the CPU has overhead of resolving the addressing mode, fetching the next instruction, etc.)

In order to instruct the DMA to activate, and copy the contents of _primary OAM_ into _secondary OAM_, the programmer can write some value to the **OAMDMA** register.
After that register is written the CPU is halted and a copy takes place.

## Sprite priority & overlapping

*As we've seen, the _PPU background_ and _PPU foreground_ rendering mechanism operates independently of each other.
1. So what happens when a sprite pixel overlaps a background pixel?
2. And if we're already talking about overlapping, what happens if we have 2 sprites which overlap on some pixel?

### Question 2 answer:

Sprite priority is determined by the position of the sprites in OAM - the lower the sprite position is (0, 1, 2) the higher it's priority is.
So if Sprite A overlaps Sprite B at some pixel, and lets say Sprite A is at OAM index 4, and Sprite B is at OAM index 20, then Sprite A "wins" and gets to have the pixel!

### Question 1 answer:

Quoted from the NESdev wiki: _"if the frontmost opaque sprite's priority bit is true (1), an opaque background pixel is drawn in front of it."_

> **_NOTE:_** Yes, if a sprites priority bit is 1, it means it should be **behind** the background. If it's 0 then it should be **infront** of background.

These weird behaviors have let NES game developers to create some unique and creative visual effects.
