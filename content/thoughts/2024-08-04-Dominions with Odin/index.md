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

```odin
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

```odin
// ^ is the pointer operator in Odin.
flip_tile:: proc(t: ^Tile) {
	t^ ~= Tile{.Controller_Is_Black}
}
```
