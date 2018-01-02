# Parallax Tiled Sprite Display

|     Item       |   Value    |   Comment
| -------------: | ---------- | ----------------
|    Vendor code | 0x8b43a166 | Parallax Graphical Engineering
|      Device ID | 0x73F747c2 | PTSD
|    Device type | 0x73F7     | Memory-mapped colour tile display
|        Version | 0x0001     |

The Parallax Tiled Sprite Display is a premium display device designed for
games. It supports 12-bit colours, a scrollable 256x256 pixel backgorund, and an
overlay of independently moveable sprites.

The visible area of the display is the same 128x96 pixels as the LEM and other
DCPU-16 displays.

Of course all this power comes at a cost, in money, complexity and RAM.


## Programmer's Model

This section summarizes the mental model of the device, with implementation
details.


### Colours and Palettes

The PTSD supports 16 palettes giving colour information. The background always
uses a single palette (which can be changed with an interrupt, see below), while
sprites each specify their palette, and can change it freely.

A palette consists of 2, 4, or 16 colours depending on the selected colour
depth. Each colour entry is a word in the same format as the LEM1802's palette
RAM: `0000rrrrggggbbbb`.

Palette RAM therefore has the format (using 4-bit colour):

```
p0c0 p0c1 ... p0cf
...
pfc0 pfc1 ... pfcf
```

Therefore the palette RAM region is 32, 64 or 256 words, depending on the colour
depth.


### Tiles

Tiles are the fundamental units of the PTSD. Tiles can be 4x4, 4x8 or 8x8
pixels, and can have 1, 2 or 4 bits of colour depth.

Therefore a tile ranges in size from a single word (4x4, 1-bit colour) to 8
words (8x8, 4-bit colour).

There are always 256 tiles in the system, so the tile data region (see below)
ranges from 256 to 2048 words.

#### Tile format

A tile consists of an index into its palette for each pixel, left to right, top
to bottom. Note that this **does not** correspond to the format of a LEM font!

Borrowing the same "capital F" example as the LEM1802 documentation, it looks
like this in 4x8 1-bit colour:

```
0: 1110100010001110
1: 1000100010000000
```

or more visually:

```
0: 1110 \
   1000 \
   1000 \
   1110
1: 1000 \
   1000 \
   1000 \
   0000
```

In 2-bit colour, each pixel is 2 bits wide. If we make the background colour 0,
the horizontal bars 1, and the vertical bar 2, we see:

```
Colour numbers    Bits
0: 2110 \         10010100 \
   2000           10000000
1: 2000 \         10000000 \
   2110 \         10010100 \
2: 2000 \         10000000 \
   2000           10000000
3: 2000 \         10000000 \
   0000           00000000
```

#### Tile RAM

There are always 256 tiles, and each tile can be 1, 2, 4, 8 or 16 words, so the
tile RAM ranges from 256 to 4096 words.

| Dimensions | Colour Depth | Tile Size | Tile RAM Size |
| :---       | :---         | :---      | :---          |
| 4x4        | 1            | 1         | 256           |
| 4x4        | 2            | 2         | 512           |
| 4x4        | 4            | 4         | 1024          |
| 4x8        | 1            | 2         | 512           |
| 4x8        | 2            | 4         | 1024          |
| 4x8        | 4            | 4         | 2048          |
| 8x8        | 1            | 4         | 1024          |
| 8x8        | 2            | 8         | 2048          |
| 8x8        | 4            | 16        | 4096          |


### Background Map

The background is 256x256 pixels, and composed of tiles. The background map is a
block of tile numbers, left to right, top to bottom. Since there are 256 tiles,
tile numbers are 8 bits. The left tile's number is in the high halfword, the
right tile in the low halfword.

When tiles are 4x4, the background map is 1024 words. When tiles are 4x8, it is
512 words. When tiles are 8x8, it is 256 words.

Note that as the tile size increases, each tile gets bigger but the tile map
gets smaller, as it takes fewer tiles to fill the same 256x256 background.

#### Colours and the Background

The single palette for the entire background is set by the `SET_BG_PALETTE`
interrupt.

#### Scrolling

The background can be scrolled horizontally and vertically with pixel
resolution.

The background map defines the complete 256x256 pixel area, which is fixed, and
the scrolling offsets define the position of the 128x96 visible area.

If the visible area extends off the right or bottom edge of the background, it
wraps around to the left or top edge. Put another way, the pixel on the screen
at index `(x, y)` is pixel `((x + scroll_x) mod 256, (y + scroll_y) mod 256)` in
the background.

Note that sprites give their coordinates relative to the visible area, and **are
not** subject to scrolling!

Scrolling is adjusted with the `SET_SCROLL` interrupt.


### Sprites

A "sprite" is a tile that sits above or below the background (see below), at a
specified position and with a specified palette.

There are 128 sprites in the PTSD. Each sprite has a 2-word entry in the sprite
RAM, which is always 256 words in size.

A sprite has the following format in memory:

```
SuVHpppp tttttttt
    S          Show bit, 1=show
    u          "Under" bit, 1=under background, 0=over background
    V          Vertical flip (1=invert Y)
    H          Horizontal flip (1=invert X)
    pppp       Palette number, 0-15
    tttttttt   Tile number, 0-255

yyyyyyyy xxxxxxxx
    yyyyyyyy   Y-coordinate of top-left corner, minus 7
    xxxxxxxx   X-coordinate of top-left corner, minus 7
```

#### Sprite Coordinates

Note that the X and Y coordinates are shifted over by 7; this allows them to be
showing just 1 pixel on all sides of the display.

For example, an X value of 0 means the top-left corner is at -7, leaving the
right edge of an 8x8 sprite just visible at column 0. (A 4x4 or 8x8 sprite would
be offscreen.) Similarly, setting the X value to 134 (127 + 7) puts the leftmost
pixel of the sprite visible at the right edge of the screen.

Note that the sprite's coordinates are relative to the visible area, and are not
subject to the background's scrolling.


#### Sprite Flipping

The `H` and `V` bits in a sprite do not change its position on the screen.

Setting `H` flips the sprite horizontally, putting its normally-rightmost pixels
at its left edge. Setting `V` flips the sprite vertically.


### Transparency

Note that colour 0 in a sprite's palette is always ignored. If the sprite's tile
has a pixel with colour 0, that pixel is treated as transparent instead.

There are effectively four layers to the display:

```
(Topmost)
Non-0 pixels of sprites with u=0 ("over" the background)
Non-0 pixels of the background
Non-0 pixels of sprites with u=1 ("under" the background)
Background colour 0
```



### Rendering Cycles

The PTSD renders 20 frames per second. When it is time for a frame to be drawn,
the PTSD takes a snapshot of memory at that point, and renders the frame
accordingly.

If the user program is in the middle of modifying the display when that snapshot
occurs, the picture can show "tearing", where a previous frame is mixed with the
current frame.

To enable programs to make their updates between the snapshots, the PTSD can
issue an interrupt right after each snapshot has been taken. Use the
`SET_INTERRUPT` interrupt to enable this feature.


## Interrupts

The PTSD has four memory mapped regions of variable sizes: tile data, background
map, sprite parameters, palettes.

If tile data or palettes are not mapped, the display is disabled and renders all
black. If the background map or sprite parameters are not mapped, that layer
will not be displayed.

The interrupt command is given in `A`, as follows:

- **0x0000**: `SET_CONFIG`

    `B` should contain the master configuration word, as follows:

    ```
    ---- ---- ---- ccss
    cc   Colour depth: 0=illegal, 1=1-bit, 2=2-bit, 3=4-bit
    ss   Tile size: 0=4x4, 1=4x8, 3=8x8. 2=(illegal)
    ```

    Note that it is legal to change these values dynamically, but that those
    changes will likely corrupt the display, since they radically change the
    interpretation of memory.

- **0x0001**: `SET_INTERRUPT`

    `B` gives the interrupt message to use for the post-snapshot interrupts. 0
    to disable interrupts.

- **0x0002**: `SET_BG_PALETTE`

    `B` gives the palette number (0-15) for the background. Defaults to 0.

- **0x0003**: `SET_SCROLL`

    `X` and `Y` give the scroll position for the background, 0-255.

- **0x0010**: `MAP_TILE_DATA`

    `B` gives the address of the tile data in memory. 0 unmaps the tile data
    (and disables the display, since tile data is required).

- **0x0011**: `MAP_PALETTES`

    `B` gives the address of the palettes in memory. 0 unmaps the palettes
    (and disables the display, since palettes are required).

- **0x0012**: `MAP_BACKGROUND`

    `B` gives the address of the background map in memory. 0 unmaps the
    background (disabling the background, but not the whole display).

    Note that a disabled background is considered to be filled with the
    background palette's colour 0, for transparency purposes.

- **0x0013**: `MAP_SPRITES`

    `B` gives the address of the sprites' parameters in memory. 0 unmaps the
    sprites, disabling sprites but not the background.

- **0xFFFF**: `RESET`

    Unmaps all memory, disables interrupts, and blanks the display.


## Memory Map Reference

For clarity and ease of reference, this table gives the sizes of each memory
region depending on the input parameters.

- Sprite RAM is always 256 words (128 sprites, 2 words each).
- Tile RAM is `w * h * c / 16`
- BG Map RAM is `(256 / w) * (256 / h)`
- Palette RAM is `2 ^ c * 16`

| Dimensions | Colour Depth | Tile RAM | BG Map RAM | Palette RAM | Total |
| :---       | :---         | :---     | :---       | :---        | :---  |
| 4x4        | 1            | 256      | 2048       | 32          | 2592  |
| 4x4        | 2            | 512      | 2048       | 64          | 2880  |
| 4x4        | 4            | 1024     | 2048       | 256         | 3584  |
| 4x8        | 1            | 512      | 1024       | 32          | 1824  |
| 4x8        | 2            | 1024     | 1024       | 64          | 2368  |
| 4x8        | 4            | 2048     | 1024       | 256         | 3584  |
| 8x8        | 1            | 1024     | 512        | 32          | 1824  |
| 8x8        | 2            | 2048     | 512        | 64          | 2880  |
| 8x8        | 4            | 4096     | 512        | 256         | 5120  |


## On Compatibility

The PTSD **is not** compatible with the LEM1802!

However, it can be used to simulate one fairly closely, if you have need to.

