---
layout: post
title: "Procedural Dungeon Generation"
---
## Overview
When I was writing my game ArduDemon for the Arduboy, a Diablo I demake, I wanted to generate random dungeons for the player to explore. Unfortunately, many tutorials described techniques either too complicated for an 8-bit processor or too confusing for me to tackle. Accordingly, I wrote my own mostly from scratch, and it is described in this article. 

## A Critical Problem
One major limitation of the Arduboy is its 2.5k of RAM. While that alone is small, the frame buffer takes up 1k leaving you just 1.5k of RAM to work with! Since procedurally generated dungeons have to live in RAM, to have anything other than tiny dungeons I needed to somehow compress everything to fit.

I decided to just use a bit to represent each tile. These could be packed, 8 into a byte. Then, when I was drawing, I would use the tile's neighbors to determine which image to draw. This meant I could have 8 times as large dungeons!

## Step One: Rooms

I first randomly generated a array of 20 rooms. After each room was generated, it would check if it overlapped with any other generated room. I increased the size of each room by one to ensure no room could be flush. If it did, it would repeatedly generate a new room until it was unobstructed. 
```cpp

static bool rooms_overlap(Rect *room, uint8_t rooms_len) {
    // prevent flush rooms
    Rect larger_room = Rect(room->x - 2, room->y - 2, room->width + 4, room->height + 4);

    for (uint8_t i = 0; i < rooms_len; i++) {
        if (Arduboy2::collide(larger_room, rooms[i])) {
            return true;
        }
    }

    return false;
}
// ...

for (uint8_t i = 0; i < ROOMS_MAX; i++) {
    Rect new_room = gen_room();

    if (rooms_overlap(&new_room, i)) {
        for (uint8_t h = 0; h < 20; h++) {
            new_room = gen_room();
            if (!rooms_overlap(&new_room, i))
                break;
        }
    }
    rooms[i] = new_room;
}

```
<img src="https://sub1inear.github.io/assets/images/dungeon-generation-ardudemon/rooms.gif" alt="Rooms" width="150" height="150" />

## Step Two: Corridors

Next, I would generate corridors to connect each room with the following one in the array. This was the trickiest part of the whole algorithm, and took me days to implement correctly.

If the two rooms overlapped horizontally, I would randomly generate a horizontal corridor between where they overlap.

```cpp
if (room1->y < (room2->y + room2->height) && room2->y < (room1->y + room1->height)) {
    corridor.start.x = (room1->x < room2->x) ? room1->x + room1->width : room1->x;
    corridor.start.y = random(max(room1->y + 1, room2->y),
                              min(room2->y + room2->height, room1->y + room1->height - 1));

    corridor.end.x = (room1->x < room2->x) ? room2->x : room2->x + room2->width;
    corridor.end.y = corridor.start.y;

}
```

If the two rooms overlapped vertically, I would randomly generate a vertical corridor between where they overlap.

```cpp
else if (room1->x < (room2->x + room2->width) && room2->x < (room1->x + room1->width)) {
    corridor.start.x = random(max(room1->x + 1, room2->x),
                              min(room2->x + room2->width, room1->x + room1->width - 1));
    corridor.start.y = (room1->y < room2->y) ? room1->y + room1->height : room1->y;

    corridor.end.x = corridor.start.x;
    corridor.end.y = (room1->y < room2->y) ? room2->y : room2->y + room2->height;

}
```

Otherwise, I would create an L-shaped corridor starting anywhere on the first room and ending anywhere on the second.
The `middle` property notes it is a L-shaped corridor.
```cpp
else { // L-joins
    corridor.start.x = random(room1->x + 1, room1->x + room1->width - 1);
    corridor.start.y = (room1->y < room2->y) ? room1->y + room1->height : room1->y;
    
    corridor.end.x = (room1->x < room2->x) ? room2->x : room2->x + room2->width;
    corridor.end.y = random(room2->y + 1, room2->y + room2->height - 1);
    
    corridor.middle = true;
}
```
<img src="https://sub1inear.github.io/assets/images/dungeon-generation-ardudemon/corridors.gif" alt="Corridors" width="150" height="150" />

## Step Three: Blitting

Now that I had arrays of rooms and corridors, I would draw each room to the final dungeon array.
```cpp
static void draw_vline_to_array(uint8_t x, uint8_t y, uint8_t h) {   
    for (int i = y; i <= y + h; i++) {
        bitClear(dungeon_array[x + (i / 8) * DUNGEON_WIDTH], i & 0x7);
    }
}
static void draw_room_to_array(Rect *room) {
    for (uint8_t i = room->x; i < room->x + room->width; i++) {
        draw_vline_to_array(i, room->y, room->height);
    }
}
```
The corridors were a little trickier. I used an auxiliary function to decide which function, `draw_hline_to_array` or `draw_vline_to_array` and whether to switch the endpoints.
```cpp
static void draw_straight_line_to_array(Point *start, Point *end) {
    if (start->y == end->y) { // horizontal line
        if (start->x < end->x) {
            draw_hline_to_array(start->x, start->y, end->x - start->x);
        } else {
            draw_hline_to_array(end->x, end->y, start->x - end->x);
        }
    } else { // vertical line
        if (start->y < end->y) {
            draw_vline_to_array(start->x, start->y, end->y - start->y);
        } else {
            draw_vline_to_array(end->x, end->y, start->y - end->y);
        }        
    }
}
```
Using that, the only thing left was to check if it was an L-shaped corridor using the `middle` property. Then, I would just create a point in the middle and draw two lines.
```cpp
static void draw_corridor_to_array(Corridor *corridor) {
    if (corridor->middle) {
        Point middle = {corridor->start.x, corridor->end.y};
        draw_straight_line_to_array(&corridor->start, &middle);
        draw_straight_line_to_array(&middle, &corridor->end);

    } else {
        draw_straight_line_to_array(&corridor->start, &corridor->end);
    }
}
```
<img src="https://sub1inear.github.io/assets/images/dungeon-generation-ardudemon/final.gif" alt="Rooms and Corridors" width="150" height="150" />

## Step Four: Population
For the entrance to the dungeon, I picked a random spot in room 0.
For the exit, I just picked a random spot in room 1.
The creatures were randomly placed all around the dungeon.
If it was a boss level, just a boss would be generated and everything else filled up with dead monsters.

## Step Five: Drawing
The final step was to draw the bit-packed dungeon. To allow for multiple tiles and a isometric look to the scene, I used the 8 neighboring tiles to determine how to draw a specific tile.

This was rather tricky to implement because of the (literal) edge cases; you don't want the next tile in the array to an edge tile to affect it! Accordingly, every read had to be checked if it was on the edge. The left and right read were unaffected as I specifically ensured the right and left sides were always ones. Therefore, it didn't matter if it wrapped around; it would stay the same. To not bore you with the details, I've only included the cardinal directions in this snippet.
``` cpp
/*  7    2     6
       ------
    3  |Tile|  1
       ------
    4    0     5 */

// down
if (bit == 7) {
    if (byte + width < dungeon_size) {
        surrounding_bits[0] = bitRead(source[byte + width], 0);
    } else {
        surrounding_bits[0] = true;
    }
} else {
    surrounding_bits[0] = bitRead(source[byte], bit + 1);    
}

// right
surrounding_bits[1] = bitRead(source[min(byte + 1, dungeon_size)], bit);

// up
if (bit > 0) {
    surrounding_bits[2] = bitRead(source[byte], bit - 1); 
} else {
    surrounding_bits[2] = bitRead(source[max(byte - width, 0)], 7);
}

// left
surrounding_bits[3] = bitRead(source[max(byte - 1, 0)], bit); 
```
Then, I would use those to determine which tile would be drawn.
- If the tile is a side, draw either a left or right side.
- If the tile is a top corner, draw a normal tile and either a left or right side.
- Otherwise, draw a normal tile.
```cpp
if (!(surrounding_bits[0] & // use bitwise and instead of logical and for speed and smaller code
      surrounding_bits[1] &
      surrounding_bits[2] &
      surrounding_bits[3] &
      surrounding_bits[4] &
      surrounding_bits[5] &
      surrounding_bits[6] &
      surrounding_bits[7])) { // not fully surrounded and not on very edge
    
    if (surrounding_bits[2] &
        surrounding_bits[0]) { // side tile
        if (!surrounding_bits[3] & !surrounding_bits[1]) { // if both sides are clear, just make it a top-down tile
            Sprites::drawOverwrite(x - (player.scrollx & 0xf), y - (player.scrolly & 0xf), tiles, 6);
        } else if (!surrounding_bits[3] | !surrounding_bits[4] | !surrounding_bits[7]) { // left
            arduboy.fillRect(x - (player.scrollx & 0xf), y - (player.scrolly & 0xf), 3, 16, WHITE);
        } else if (!surrounding_bits[1] | !surrounding_bits[5] | !surrounding_bits[6]) { // right
            arduboy.fillRect(x - (player.scrollx & 0xf) + 16 - 3, y - (player.scrolly & 0xf), 3, 16, WHITE);
        } 
    
    } else if (surrounding_bits[0] & (!surrounding_bits[7] |
                !surrounding_bits[6]) & (!surrounding_bits[3] ^
                !surrounding_bits[1]) & (surrounding_bits[4] |
                surrounding_bits[5])) { // corner tile (top)
        Sprites::drawOverwrite(x - (player.scrollx & 0xf), y - (player.scrolly & 0xf), tiles, 6);
        if (surrounding_bits[3]) {
            arduboy.fillRect(x - (player.scrollx & 0xf) + 16 - 3, y - (player.scrolly & 0xf), 3, 16, WHITE);
        } else {
            arduboy.fillRect(x - (player.scrollx & 0xf), y - (player.scrolly & 0xf), 3, 16, WHITE);

        }
    } else { // top down tile
        Sprites::drawOverwrite(x - (player.scrollx & 0xf), y - (player.scrolly & 0xf), tiles, 6);
    }
}
```
Try it for yourself here:
<iframe src="https://tiberiusbrown.github.io/Ardens/player.html?blah=https://community.arduboy.com/uploads/short-url/loy2rj8nqf4ir20TOmuUjIhBZsu.hex&g=none&z=1&p=0&palette=highcontrast" title="Ardudemon" width=512vw height=256vh></iframe>


## Conclusion
And that's it! (Well, a very small portion.) I hope you've enjoyed this guide and semi-understood it. The full code is available [here](https://github.com/sub1inear/ArduDemon) in the `dungeon.cpp` (dungeon generation) and `map.cpp` (rendering) files, if you want to dig a little deeper. 