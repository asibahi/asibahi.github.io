+++
title = "Dominions with Odin"
draft = true
+++

In an escape from the worries of life, I have been reading recently into the [Odin programming language](https://odin-lang.org). I wanted to learn something that's closer to the metal than Rust, and Odin seems nice.

For a project to do with the language, I figured I would build a library of sorts for the abstract boqard game [Domnions, by Christian Freeling](https://mindsports.nl/index.php/the-pit/526-dominions). ([Sensei's Library link](https://senseis.xmp.net/?Dominions)). The game can be described as a Go variant with distinct pieces (as opposed to Go itself where every "piece" is identical.

The game is weird. It is much less known than Freeling's other games like Havannah (for which I wrote [an implementation for in Rust](https://github.com/asibahi/w9l)) and Grand Chess (which I built a physical board for). Even Freeling himself does not put much stock into it, and is more interested in the tile set itself (which he calls [the China Labyrinth](https://mindsports.nl/index.php/puzzles/tilings/china-labyrinth/), even though they have nothing to do with China) than the game. He made other games with the tile set which you can find by browsing his site.

I am writing this post first as a way to organize my thoughts on how to represent the game in code. I will not be explaining the rules of the game, see the above links for that, but I will be talking about to represent the game in code form.

## Tiles

The most intriguing feature of Dominions, for me, is the tiles. There are many tile sets and game piece sets in the world of games: chess, dominoes, mahjong, playing cards, tarot. Dominions tiles are such a set.

![Full set of Dominions Tiles](cl-small-set.png)

Each tile defines how it connects to its neighbors. They do not rotate, but they can be flipped to change controller (which is how capture happens in the game). A full set of tiles, pictured above, has 64 shapes.

A clever Id/numbering scheme for the tiles is described by Freeling.

![Full set of Dominions Tiles](numbering.png)

Each side is assigned a power of two. So each piece unique id is simply the sum of which sides it connects. Binary system! 

Each player starts the game with the full set of tiles, minus the Blank, the Zero-Tile, as it does not have a role in the game. So each tile has also, an owner.

During the game, control of tiles can be flipped from the owner to the opponent. After all you win the game by controlling more tiles.

All this neatly fits into a `u8`! And Odin has bitsets native into the language. When I was asking questions in Odin's discord, Ginger Bill (BDFL of Odin) suggested this structure.

```cpp
// 0b 0 0 _ 0 0 0 _ 0 0 0
//    | |   | | |   | | | 
//    | |   | | |   | | Top_Right
//    | |   | | |   | Right
//    | |   | | |   Btm_Right
//    | |   | | Btm_Left
//    | |   | Left
//    | |   Top_Left
//    | Owner_Is_Black
//    Controller_Is_Black
// 
// the two players are White and Black. White starts.

Tile_Flag :: enum u8 {
	Top_Rght,
	Right,
	Btm_Right,
	Btm_Left,
	Left,
	Top_Left,
	Owner_Is_Black,
	Controller_Is_Black,
}
Tile :: distinct bit_set[Tile_Flag;u8]
```

The game starts with 126 unique tiles. White starts with the tiles `0b00_000_001` to `0b00_111_111`. Black starts with the tiles `0b11_000_001` to `0b11_111_111`. The highest bit, can flip during the game.

```cpp
// ^ is the pointer operator in Odin.
// ~ is bitwise XOR.
flip_tile:: proc(t: ^Tile) {
	t^ ~= Tile{.Controller_Is_Black}
}
```

## Board representation

Here is where I go back and forth into reading Red Blob Games's [excellent guide to Hexagonal boards](https://www.redblobgames.com/grids/hexagons/).

The board of Dominions is a 9-sided hexagon. 217 cells. Let's defer tile placement restrictions for now. How do we actially place tiles onto the board? 

I oscillated (heh) between a few ideas, but eventually settled on a giant big array of `[217]Tile`, where an empty cell has the value `0`[^1]. To calculate offsets, I adapted the functions declared in the Red Blob article, and started with this neat loop (`N`, `CENTER`, and `CELL_COUNT` are compile-time constants based on the board's size):

```cpp
Board :: [CELL_COUNT]Tile

get_tile :: proc(back: ^[CELL_COUNT]Tile, hex: Hex) -> (^Tile, bool) {
	if hex_distance(hex, CENTER) > N {
		return nil, false
	}

	// From RedBlob on representing Hexagonal boards with an array of arrays:
	// Store Hex(q, r) at array[r][q - max(0, N-r)]. Row r size is 2*N+1 - abs(N-r).
	// this loop calculates r's offset.

	q, r := hex[0], hex[1]

	r_len := 0
	for r_idx in 0 ..< r {
		r_len += 2 * int(N) + 1 - abs(int(N - r_idx))
	}
	

	idx := r_len + int(q - max(0, N - r))
	return &back[idx], true
}
```

But having this loop run every time I make a look up is .. uncertain at best. So I decided to hardcode the row offsets, but kept the loop in case i am experimenting with different board sizes for debugging.

```cpp
get_tile :: proc(back: ^[CELL_COUNT]Tile, hex: Hex) -> (^Tile, bool) {
	if hex_distance(hex, CENTER) > N {
		return nil, false
	}

	q, r := hex[0], hex[1]

	r_len := 0
	// `when` is a compile-time `if`.
	when N == 8 {
		switch r {
		case 0:
		case 1:  r_len =   9
		case 2:  r_len =  19
		case 3:  r_len =  30
		case 4:  r_len =  42
		case 5:  r_len =  55
		case 6:  r_len =  69
		case 7:  r_len =  84
		case 8:  r_len = 100
		case 9:  r_len = 117
		case 10: r_len = 133
		case 11: r_len = 148
		case 12: r_len = 162
		case 13: r_len = 175
		case 14: r_len = 187
		case 15: r_len = 198
		case 16: r_len = 208
		}
	} else {
		// For when I am trying non-default board sizes.
		for r_idx in 0 ..< r {
			r_len += 2 * int(N) + 1 - abs(int(N - r_idx))
		}
	}

	idx := r_len + int(q - max(0, N - r))
	return &back[idx], true
}
```

This has the advantage of being "clean", with no wasted place in the array for unused coordinates. I should be careful not to index into `Board` directly, however. Cell lookup is O(1): when you're checking a cell you *need* to check its neighbors to verify legal moves.

We would need a separate data structure to track the tiles in player's hands (not in play yet).

An alternative idea is to represent `Tile` itself by a struct, that holds data whether the tile is in play, and if so its location on the board. However, how do you quickly query the tile's neighbors? You'd need to iterate over every tile at every check to know where it is, might as well stuff them all in an array.

---

[^1]: As mentioned earlier, the Blank tile is not used in the game. This permits using a sentinel value of `0` (or really any value with the smallest six bits set to `0`) to mark an empty cell. Since we are using a `u8` bitset anyway, why waste memory on pointers (which are wider), or `Maybe`, which is at least an extra byte in size? I am not thinking *too* hard about performance (I know nobody will use this), but it is an interesting constraint to keep in mind.
