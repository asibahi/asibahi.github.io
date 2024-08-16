+++
title = "The Garden of Odin"
description = "An Odin implementation of an abstract board game."
draft = true
+++

In an escape from the worries of life, I have been reading recently into the [Odin programming language](https://odin-lang.org). I wanted to learn something that is closer to the metal than Rust, and Odin seems nice.

For a project to do with the language, I figured I would build a library of sorts for the abstract board game [Domnions, by Christian Freeling](https://mindsports.nl/index.php/the-pit/526-dominions). ([Sensei's Library link](https://senseis.xmp.net/?Dominions)). The game can be described as a Go variant with distinct pieces (as opposed to Go itself where every "piece" is identical.

The game is *weird*. It is much less known than Freeling's other games like Havannah (for which I wrote [an implementation for in Rust](https://github.com/asibahi/w9l)) and Grand Chess (which I designed and built a physical set for). Even Freeling himself does not put much stock into it, and is more interested in the tile set itself (which he calls [the China Labyrinth](https://mindsports.nl/index.php/puzzles/tilings/china-labyrinth/), even though they have nothing to do with China) than the game. He made other games with the tile set which you can find by browsing his site.

![A game in progress of Dominions](dominions.png)

This is the sample game Freeling posted on his website.

I am writing this post as a way to organize my thoughts on how to represent the game in code. It is a semi organized, semi-chronologically-sorted brain dump: I am writing it as I iterate over the code. I will be talking mostly about how to represent the game in code form, leaving the rules for later.

## Tiles

The most intriguing feature of Dominions, for me, is the tiles. There are many tile sets and game piece sets in the world of games: chess, dominoes, mahjong, playing cards, tarot. Dominions tiles are such a set.

![Full set of Dominions Tiles](cl-small-set.png)

Each tile defines how it connects to its neighbors. They do not rotate, but they can be flipped to change controller (which is how capture happens in the game). A full set of tiles, pictured above, has 64 shapes.

A clever Id/numbering scheme for the tiles is described by Freeling.

![Full set of Dominions Tiles](numbering.png)

Each side is assigned a power of two. So each piece unique id is simply the sum of which sides it connects. Binary system! 

Each player starts the game with the full set of tiles, minus the Blank, the Zero-Tile, as it does not have a role in the game. So each tile has also, an owner.

During the game, control of tiles can be flipped from the owner to the opponent. After all one wins the game by controlling more tiles.

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
//    | Owner_Is_Host
//    Controller_Is_Host
// 
// the two players are Guest and Host. Guest starts.

Tile_Flag :: enum u8 {
	Top_Rght,
	Right,
	Btm_Right,
	Btm_Left,
	Left,
	Top_Left,
	Owner_Is_Host,
	Controller_Is_Host,
}
Tile :: distinct bit_set[Tile_Flag;u8]
```

The game starts with 126 unique tiles. Guest starts with the tiles `0b00_000_001` to `0b00_111_111`. Host starts with the tiles `0b11_000_001` to `0b11_111_111`. The highest bit, can flip during the game.

```odin
// ^ is the pointer operator in Odin.
// ~ is bitwise XOR.
tile_flip:: proc(t: ^Tile) {
	t^ ~= Tile{.Controller_Is_Host}
}
```

## Board representation

Here is where I go back and forth into reading Red Blob Games's [excellent guide to Hexagonal boards](https://www.redblobgames.com/grids/hexagons/).

The board of Dominions is a 9-sided hexagon. 217 cells. Ignoring tile placement restrictions for now, how to actially place tiles onto the board? 

I oscillated (heh) between a few ideas, but eventually settled on a giant big array of `[217]Tile`, where an empty cell has the value `0`[^1]. To calculate offsets, I adapted the functions declared in the Red Blob article, and started with this neat loop (`N`, `CENTER`, and `CELL_COUNT` are compile-time constants based on the board's size):

```odin
Board :: [CELL_COUNT]Tile

board_get_tile :: proc(back: ^[CELL_COUNT]Tile, hex: Hex) -> (^Tile, bool) {
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

But having this loop run every time I make a look up is .. uncertain at best. So I decided to hardcode the row offsets:

```odin
board_get_tile :: proc(back: ^[CELL_COUNT]Tile, hex: Hex) -> (^Tile, bool) {
	if hex_distance(hex, CENTER) > N {
		return nil, false
	}

	q, r := hex[0], hex[1]

	r_len := 0
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

	idx := r_len + int(q - max(0, N - r))
	return &back[idx], true
}
```

This has the advantage of being "clean", with no wasted place in the array for unused coordinates.[^2] Cell lookup is O(1): when checking a cell, it is imperative to check its neighbors quickly to verify legal moves.

A separate data structure would be needed to track the tiles in player's hands (not in play yet).

An alternative idea is to represent `Tile` itself by a struct, that holds data whether the tile is in play, and if so its location on the board. However, the only way to quickly query the tile's neighbors is to iterate over every tile at every check to know where it is. Might as well stuff them all in an array.

## Hand 

Following the same idea from representing the board, Hands are better implemented as a fixed array. Initializing a full set of hands for both players by transmuting a `u8` into a `Tile`. *That is* why I wanted `Tile`s to be `u8`s to begin with!! 

```odin
HAND_SIZE :: 63
Hand :: distinct [HAND_SIZE]Tile

hands_init :: proc() -> (guest: Hand, host: Hand) {
	for i in 0 ..< u8(HAND_SIZE) {
		guest[i] = transmute(Tile)(i + 0b00_000_001)
		host[i]  = transmute(Tile)(i + 0b11_000_001)
	}

	return
}
```

Also, to check the legality of move, the player has to have the tile available to play, so I defined a couple of helper ~~methods~~ procedures.

```odin
hand_has_tile :: proc(hand: Hand, id: u8) -> bool {
	assert(0 < id, "Blank Tile is not playable")
	assert(id <= HAND_SIZE, "Tile is impossible")

	return !tile_is_empty(hand[id - 1])
}

hand_get_tile :: proc(hand: ^Hand, id: u8) -> (Tile, bool) {
	if !hand_has_tile(hand^, id) do return nil, false

	ret := hand[id - 1]
	hand[id - 1] = {}
	return ret, true
}
```

About here, I thought I would like to test my logic. So I took advantage of Odin's testing framework with a couple of simple asserts. Note that I put these in the same file as the procedures they test, and they worked just fine. They are all prefixed with `test_*` not to pollute the autocompletion too much.

```odin
import "core:testing"

@(test)
test_hand_get_tile :: proc(t: ^testing.T) {
	w, _ := hands_init()

	tile: Tile
	ok: bool

	// should be successful
	tile, ok = hand_get_tile(&w, 63)
	testing.expect(t, !tile_is_empty(tile))
	testing.expect(t, ok)

	// should fail. tile was emptied
	tile, ok = hand_get_tile(&w, 63)
	testing.expect(t, tile_is_empty(tile))
	testing.expect(t, !ok)
}
// other tests go here
```

After that, I figured I would make sure for every weird logic I make I'd have to write a bunch of tests to .. test it, and sanity check my code.

## *The* Game Object - First Draft

Now that I have the basic, naïve framework of the game's representation, I started looking into it from the other side: top down.

Looking at different game libraries, to see which API they provide and how they allow their users to interact with the internal rules engine. The [shakmaty crate](https://docs.rs/shakmaty/latest/shakmaty/) has been of great help in the past, as well as the [goban crate](https://docs.rs/goban/0.18.1/goban/). JaniM from the Rust discord also shared with me their [Variant Go Server](https://github.com/JaniM/variant-go-server). I also looked back at my [Havannah implementation](https://github.com/asibahi/w9l) which I did last year to remember how I structured things. (Maybe I'd tool both into Wasm modules. How about that?)

Most of these structure their API around a specific object: the *Game* object, which is interacted with by querying legal moves, making moves, and, optionally, undoing moves. That's it. The Game object tracks the state of all elements in the game.

The most straightforward way to do that is a struct. I do not really know if this is the "optimal" arrangment of fields, but I am doing what makes sense to me.

```odin
Player :: enum u8 {
	Guest, // White. Goes first.
	Host,  // Black
}

Status :: enum u8 {
	Ongoing,
	Guest_Win,
	Host_Win,
	Tie,
}

// The Object of our attention
Game :: struct {
	board:                 Board,
	to_play:               Player,
	status:                Status,
	guest_hand, host_hand: Hand,
	// ... more to come
}
```

To inquire about moves, a `Move` struct is needed[^3]. A move is simply a placement of a `Tile` on `Hex`. Whose tile and whose turn are ideally tracked and verified by the Game object.

```odin
Move :: struct {
	hex:  Hex,
	tile: Tile,
}
```

This misses one big thing: Groups. Dominions is a game of territory, based on Go. Tiles together make Groups. Groups have libertiesm which they live and die of. Groups capture other Groups. The winner is the player with bigger Groups. Groups are importamt.

## Bitboards

Before talking about Groups, [Bitboards](https://en.wikipedia.org/wiki/Bitboard) warrant an introduction.

Bitboards are, at their core, based on a simple observations: a chessboard has 64 squaresm and a `u64` has 64 bits. By tying each bit address to a specific square, *any* binary value can be mapped.

In chess implementations: There is a bitboard (read: a `u64`) to mark where all the White pieces are. There is another bitboard to mark where all the squares the Queen sees are.

The other advantage of bitboards is that, since they are just bits, they have bit operations. With a bitboard showing the white pieces are and a bitboard showing where black's pieces can move next turn: just `AND` them together and there is now a new bitboard of which pieces of white are under attack. They are small, simple integers, and operating on them is as easy as integers. 

Unfortunately, however, the Dominions board is decidedly *not* 64 cells, or any such convenient number. It is 217 cells. It would be possible to represent the whole board with a 217bit integer, should it exist, but the largest bit set Odin provides is 128 bits. But have no fear! An array of 7 bit sets can solve the problem, as 7 `u32` integers can fit our need 217 bits and more (namely 224 bits). This is as small as can be. Thanks to Odin's array programming (which is also taken advantage of for Hex math), using bitwise operations on these bitboards is as easy as they are on usual bitsets. The additional 7 bits can be used for metadata, as well.

```odin
Bitboard :: distinct [7]bit_set[0 ..< 32;u32] // 7 * 32 = 224
PLAY_AREA: Bitboard : {~{}, ~{}, ~{}, ~{}, ~{}, ~{}, ~{25, 26, 27, 28, 29, 30, 31}}
DATA_AREA: Bitboard : {{}, {}, {}, {}, {}, {}, {25, 26, 27, 28, 29, 30, 31}} // To use the extra bits for metadata

// helper procedure for common maths 
@(private = "file")
bit_to_col_row :: proc(bit: int) -> (col, row: int) {
	col = bit % 32
	row = bit / 32
	return
}

bb_set_bit :: proc(bb: ^Bitboard, bit: int) {
	assert(0 <= bit && bit < CELL_COUNT) // assert here to allow an unchecked version for meta data
	col, row := bit_to_col_row(bit)
	bb^[row] |= {col}
}

bb_get_bit :: proc(bb: Bitboard, bit: int) -> bool {
	col, row := bit_to_col_row(bit)
	return col in bb[row]
}
```

Not to get ahead of the topic, but merging two of these bitboards (bitwise `OR`) is now as simple as this:

```odin
group_capture :: proc(winner, loser: ^Group) {
	// ----- snipped
	winner.tiles |= loser.tiles // the `tiles` field is a Bitboard.
	// ----- snipped
}
```

As an addition, I implemented an iterator over the set bits in `Bitboard`. Odin has nice syntax to iterate over the set flags in a bitset, but an array of bitsets presents a logistical challenge. I hacked at it for a couple of hours and came up with the following. Odin's implementation of iterators is, all considered, fairly easy.

```odin
// The State machine
Bitboard_Iterator :: struct { 
	bb:   Bitboard,
	next: int,
}
bb_make_iter :: proc(bb: Bitboard) -> (it: Bitboard_Iterator) {
	it.bb = bb & PLAY_AREA
	return
}

// The function
bb_iter :: proc(it: ^Bitboard_Iterator) -> (item: Hex, idx: int, ok: bool) {
	for i in it.next ..< CELL_COUNT {
		if bb_get_bit(it.bb, i) {
			item = hex_from_index(i)
			idx = i
			it.next = i + 1
			ok = true
			return
		}
	}

	return
}

// And it is used such:
bbi := bb_make_iter(bitboard_from_somewhere)
for hex, idx in bb_hexes(&bbi) {
	// do something with this hex or its corresponding index
}
```

With the help of the Hex to Index math mentioned earlier in `Board` (and I have by now extracted it to another procedure), each bit in the bitboard can be quickly mapped to its corresponding Hex.

Which would bring the topic back to Groups, but there is another detour.

## Slotmaps

In my implementation of Havannah (linked earlier), I used the `slotmap` Rust crate to track groups. Odin does not have a slotmap[^4] in its core library. There is [a sample showcase implementation by Ginger Bill](https://gist.github.com/gingerBill/7282ff54744838c52cc80c559f697051), but I wanted to try my hand at this FFI thing. With help from the Rust and Odin discords to navigate the FFI of both languages, I did the following, and it works! Almost statically typechecked from both sides, too.

```rust
use slotmap::{DefaultKey, Key, KeyData, SlotMap};

// The Rust code does not need to manipulate the object in anyway.
// So a type erased `*mut c_void` is all Rust knows. It is just a mutable pointer.
type SmItem = *mut core::ffi::c_void;
type SmPtr = *mut SlotMap<DefaultKey, SmItem>;

#[no_mangle]
pub extern "C" fn slotmap_init() -> SmPtr {
    let sm = Box::new(SlotMap::<_, SmItem>::new());
    Box::into_raw(sm)
}

#[no_mangle]
pub unsafe extern "C" fn slotmap_destroy(sm: SmPtr) {
    _ = unsafe { Box::from_raw(sm) };
}

#[no_mangle]
pub unsafe extern "C" fn slotmap_insert(sm: SmPtr, item: SmItem) -> u64 {
    let Some(sm) = (unsafe { sm.as_mut() }) else {
        return 0;
    };
    let handle = sm.insert(item);
    handle.data().as_ffi()
}

#[no_mangle]
pub unsafe extern "C" fn slotmap_contains_key(sm: SmPtr, key: u64) -> bool {
    let Some(sm) = (unsafe { sm.as_mut() }) else {
        return false;
    };
    let key = DefaultKey::from(KeyData::from_ffi(key));
    sm.contains_key(key)
}

#[no_mangle]
pub unsafe extern "C" fn slotmap_get(sm: SmPtr, key: u64) -> SmItem {
    let Some(sm) = (unsafe { sm.as_mut() }) else {
        return core::ptr::null_mut();
    };
    let key = DefaultKey::from(KeyData::from_ffi(key));
    let ret = sm.get(key);
    *ret.unwrap_or(&core::ptr::null_mut())
}

#[no_mangle]
pub unsafe extern "C" fn slotmap_remove(sm: SmPtr, key: u64) -> SmItem {
    let Some(sm) = (unsafe { sm.as_mut() }) else {
        return core::ptr::null_mut();
    };
    let key = DefaultKey::from(KeyData::from_ffi(key));
    sm.remove(key).unwrap_or(core::ptr::null_mut())
}
```

And after compiling this to a `staticlib`, do this from the Odin side:

```odin
foreign import slotmap "deps/slotmap/libslotmap.a"
import "core:c"

Slot_Map :: distinct rawptr
Sm_Key :: distinct c.uint64_t
Sm_Item :: ^Group

foreign slotmap {
	slotmap_init	:: proc() -> Slot_Map ---
	slotmap_destroy :: proc(sm: Slot_Map) ---
	slotmap_insert	:: proc(sm: Slot_Map, item: Sm_Item) -> Sm_Key ---
	slotmap_contains_key :: proc(sm: Slot_Map, key: Sm_Key) -> c.bool ---
	slotmap_get	:: proc(sm: Slot_Map, key: Sm_Key) -> Sm_Item ---
	slotmap_remove 	:: proc(sm: Slot_Map, key: Sm_Key) -> SmItem ---
}
```

Odin's type system allows a `rawptr` to really be cast to .. any pointer. So here all the types I *know* are the same, regardless of C's type erasure, are marked with the same alias. Even though, as far as the C ABI is concerned, both `Slot_Map` and `Sm_Item` are `rawptr`s, but *I* know the difference. Conveniently, `Sm_Item` can now be easily derefrenced to get the underlying Group without casting.

Having done that, and as much as I am pleased with myself for getting this to work, I do not like that the keys are `u64`s. They are *large*: much larger than any number of groups/indices required. Even in the unlikely event of each tile placed forming its own group, there would be a maximum of 126 groups (2 x 63). Eight bits would be enough to have a unique key for every possible group in the game. Something to optimize later, perhaps?

## Groups, and Game Object - Second Draft

A group is, simply, two bitboards:

```odin
Group :: struct {
	tiles:     Bitboard,
	liberties: Bitboard, 	
}
```

Each group keeps track of its location (which hexes it occupies) and its liberties. A group's size and liberty count is calculated by calculating the cardinality of the respective bitboards.

```odin
bb_card :: proc(bb: Bitboard) -> (ret: int) {
	temp := bb & PLAY_AREA
	#unroll for i in 0 ..< 7 { // premature optimization, perhaps?
		ret += card(temp[i])
	}
	return
}

group_size :: proc(grp: ^Group) -> int {
	return bb_card(grp.tiles)
}


group_life :: proc(grp: ^Group) -> int {
	return bb_card(grp.liberties)
}
```

To track groups, two fields are added to the Game Object: one for each player. Additionally, a "Group Map" of sorts might be needed to quickly look up the group any cell belongs to. (This is the initial idea, but keeping all these things in sync seems daunting. There could be a better design that is only revealed with a conrete implementation.)

Revisiting the `Game` struct from before:

```odin
Game :: struct {
	board:                 Board,
	to_play:               Player,
	status:                Status,
	last_move:             Maybe(Move), // <-- I will get to that
	guest_hand, host_hand: Hand,
	groups_map:            [CELL_COUNT]Sm_Key,
	guest_grps, host_grps: Slot_Map,
}
``` 

Note that the score is not recorded here. As with many aspects of this design, I am currently unsure if it something should be tracked and updated individually with each move, or something calculated from the game state at any given moment.

There is also no history tracking, which needs lists of `Move`s and past  `Boards` and `Hand`s. The slotmap, being currently the only heap allocation, complicates undoing moves trivially, so a new `Game` object would be constructed from past raw data of boards and games. I decided to ignore history tracking for now.

```odin
// A player's territory consists of the number of their pieces on the board minus the number of pieces they didn't place.
game_get_score :: proc(game: ^Game) -> (guest, host: int) {
	for tile in game.board {
		(!tile_is_empty(tile)) or_continue
		if .Controller_Is_Host in tile {
			host += 1
		} else {
			guest += 1
		}
	}
	for tile in game.guest_hand {
		if !tile_is_empty(tile) do guest -= 1
	}
	for tile in game.host_hand {
		if !tile_is_empty(tile) do host -= 1
	}
	return
}
```

Now the game state and its metadata at any given point are neatly encoded, it is time to start thinking about how to make moves, and how to check their legality.

## The Game Rules

So far, I made no mention of the game's rules, only talking about its physical properties: the tiles, the board, the game zones. But players cannot place Tiles where they choose: here are the rules summarized for your convenience:

![Full set of Dominions Tiles](cl-small-set.png)

### Tiles

- Each **Tile** has a specific set of sides it can connect to, encoded, visually, on the Tile itself.
- Each Tile *must* match its neighboring Tiles in connections. **Connected** sides must match and **Separated** sides must match.
- Board edges are cosidered neutral Separateds.

### Groups

- A **Group** is one or more Tiles of the same color (Controller) that are connected together. Adjacencies on Separated sides do not count.
- The **Liberties** of a Group are its Connected sides border empty cells.
- If a Group loses its last Liberty, it is captured, and all its tiles are flipped (switched Controller).

### The Structure

- The **Structure** is the collective name of all the connections together, in the whole board, *regardless of Controller*.
- Disconnected parts of the Structure are called **Sections**, created as Tiles are placed on Separated adjacencies. A single Section can include multiple Groups (but a Group cannot belong to multiple Sections).

### Placement

*Finally*

- Bears repeating: A Tile can only be placed where it matches its neighboars in Connected and Separated sides.
- Firstly, the Guest, first player, starts by placing any Tile anywhere.
- Afterwards, a Tile can only be placed *adjacent to an enemy Tile*, with one exception.
- A Tile can be placed where it is not adjacent to an enemy Tile *if and only if* it is extending a Group that is a whole Section.
- Suicide is legal. Player may place a Tile so it has no Liberties (by matching its Liberties to enemy Tiles), and is therefore immediately captured.
- Lastly, it is illegal to cause **Oscillation**: a Section which has no Liberties. [^5]
- A Player may, instead of placing a Tile, pass the turn instead. Or if they have no legal moves, they *must*.

### Scoring

- The Game ends when both players pass consecutively.
- The winner is the Player with the highest score.
- The score is the amount of Tiles controlled on the boad *minus* the Tiles still in Hand.

## Moves, and Game Object - Third Draft

Back to code. As a refresher, here is `Move`:

```odin
Move :: struct {
	hex:  Hex,
	tile: Tile,
}
```

That is all a `Move` is: the player whose turn it is (as tracked by `Game`) places a `Tile` from their hand into a `Hex`, and from there follows that groups get updated and tiles get flipped and game progresses.

But as a player may (or may have to) pass the turn, and as the game ends with two successive passes, `last_move` is registered in `Game` to track whether the game is about to end.[^6]

To play the game, `Game` needs to have the following:

- A way to make the move
- A way to query if a move is legal
- A way to query the list of legal moves

Perhaps the most straightforward way to do it is to have `game_make_move` do *all* the work. It updates the game state, naturally, but also (based on said game state) update an embedded list of legal moves in `Game` itself. And then to check if a move is legal, simply query the embedded list of moves. Here is what `Game`, and the skeleton of `game_make_move`, look like now:

```odin
Game :: struct {
	board:                 Board,
	to_play:               Player,
	status:                Status,
	last_move:             Maybe(Move),
	//
	guest_hand, host_hand: Hand,
	//
	groups_map:            [CELL_COUNT]SmKey,
	guest_grps, host_grps: SlotMap,
	// 
	legal_moves:           [dynamic]Move,
}

game_make_move :: proc(game: ^Game, candidate: Maybe(Move)) -> bool {
	(game.status == .Ongoing) or_return // Game is over. What are you doing?
	move, not_pass := candidate.?

	// legal move check
	(!not_pass ||
		slice.contains(game.legal_moves[:], move) ||
		board_is_empty(&game.board)) or_return

	defer if game.status == .Ongoing {
		switch game.to_play {
		case .Guest:
			game.to_play = .Host
		case .Host:
			game.to_play = .Guest
		}
		game.last_move = move
		game_regen_legal_moves(game)
	}

	if !not_pass { 	// Pass
		if game.last_move == nil { 	// Game ends
			guest, host := game_get_score(game)
			if guest > host do game.status = .Guest_Win
			else if guest < host do game.status = .Host_Win
			else do game.status = .Tie
		}
		return true
	}

	active_hand: ^Hand

	switch game.to_play {
	case .Guest:
		active_hand = &game.guest_hand
	case .Host:
		active_hand = &game.host_hand
	}

	// Make move. Already known to be legal!!
	hand_tile, removed := hand_remove_tile(active_hand, move.tile)
	assert(removed) // but verify
	game.board[hex_to_index(move.hex)] = hand_tile
	move.tile = hand_tile // might be superfluous, but just to ascertain the Owner flags are set correctly

	// TODO: update game state

	return true
}

@(private)
game_regen_legal_moves :: proc(game: ^Game) {
	clear(&game.legal_moves)

	// TODO: build them again
}
```

The two `TODO`s left are probably the meat of this whole engine. It is clearer to write them out in steps, in English, before encoding them into code.

## Updating the Game State

The key observation, I think, is that the game state change starts from the where the last move is played. No need for a global search process, but only the exact Hex being played and the surrounding groups.

### `group_section_init`

The simplest, and shortest, game state change that is not a Pass, is a Tile that starts its own Section. This happens in two occasions: the first move, and whenever a Tile is only adjacent to other tiles via its Separated sides. This Tile by definition also starts a new Group. This function assumes it is only called when a move is legal.

```odin
@(private)
group_section_init :: proc(move: Move, game: ^Game) -> (ret: Group, ok: bool = true) {
	// Check that all Connected sides connect to empty tiles.
	// AND find Liberties
	for flag in move.tile & CONNECTION_FLAGS {
		neighbor := move.hex + flag_dir(flag)

		t, in_bounds := board_get_tile(&game.board, neighbor)
		if in_bounds {
			tile_is_empty(t^) or_return
			bb_set_bit(&ret.liberties, hex_to_index(neighbor))
		}
	}

	bb_set_bit(&ret.tiles, hex_to_index(move.hex))
	bb_set_bit_unchecked(&ret.liberties, CELL_COUNT) // Special location for whether it is own section

	return
}
```

And in `game_make_move` :

```odin
if grp, ok := group_section_init(move, game); ok {
	grp := new_clone(grp)
	key := slotmap_insert(game.host_grps, grp)
	game.groups_map[hex_to_index(move.hex)] = key
} 
```

### `group_attach_to_friendlies`

The second easiest to deal with is a Tile that only connects to Groups of its own color. But there are a few variations

- *Extension*: The Tile may only extend one Group
- *Merge*: The Tile may merge two friendly Groups into one.
- *Suicide*: The Tile may consume the remaining Liberties of the Group(s) it connects to, Suiciding the whole Group. In which case the newly formed Group is captured (flipped) as a whole, and merged with the surrounding enemy Group(s).

The last case is interesting. Note that Move legality is checked earlier, so on principle, it is *known* that the newly formed enemy Group has at least one Liberty left.

A funny observation here is that it is impossible for a Tile that *only connects to friendlies* to Capture an enemy Group, but it is entirely possible that is causes its own Group to be immediately captured.

An easy check to whether this move is a Suicide is to check whether the new Tile has any Liberties of its own. For example: if it has 2 Connections and is connected to a friendly Group from only one side of the two, and the other Connection is to an empty cell, then it is obviously *not* a Suicide.

Ergo, the procedure for extension or merging is as follows. It turned *way* longer than I expected.

```odin
@(private)
group_extend_or_merge :: proc(move: Move, game: ^Game) -> (ok: bool = true) {
	// Bug tracker
	tile_liberties_count := card(move.tile & CONNECTION_FLAGS)

	// Friendliness tracker
	tile_control := move.tile & {.Controller_Is_Host}

	// Scratchpad: Found Groups
	nbr_friend_grps: [6]Sm_Key
	nfg_cursor: int

	// Scratchpad: New Liberties
	new_libs: [6]Hex
	libs_cursor: int

	for flag in move.tile & CONNECTION_FLAGS {
		neighbor := move.hex + flag_dir(flag)
		nbr_tile := board_get_tile(&game.board, neighbor) or_continue

		if tile_is_empty(nbr_tile^) {
			new_libs[libs_cursor] = neighbor
			libs_cursor += 1
		} else if nbr_tile^ & {.Controller_Is_Host} == tile_control {
			// Same Controller
			tile_liberties_count -= 1

			// record Group of neighbor tile.
			key := game.groups_map[hex_to_index(neighbor)]
			if !slice.contains(nbr_friend_grps[:], key) {
				nbr_friend_grps[nfg_cursor] = key
				nfg_cursor += 1
			}
		} else {
			// Different Controller
			return false
		}
	}
	assert(tile_liberties_count >= 0, "if this is broken there is a legality bug")
	assert(nfg_cursor > 0, "This proc should not be called with no friendly neighbors")
	(tile_liberties_count > 0) or_return 	// if this is false then this might be a Suicide

	// == Are we the Baddies?
	friendly_grps: Slot_Map
	if tile_control == {} {
		friendly_grps = game.guest_grps
	} else {
		friendly_grps = game.host_grps
	}

	// == Identify first group
	blessed_key := nbr_friend_grps[0]
	assert(slotmap_contains_key(friendly_grps, blessed_key))
	blessed_grp := slotmap_get(friendly_grps, blessed_key)
	bb_set_bit(&blessed_grp.tiles, hex_to_index(move.hex))

	// == Merge other groups with first group
	for i in 1 ..< nfg_cursor {
		assert(slotmap_contains_key(friendly_grps, nbr_friend_grps[i]))
		grp := slotmap_remove(friendly_grps, nbr_friend_grps[i])
		defer free(grp)

		blessed_grp.tiles |= grp.tiles

		// Only if both Groups are in their own Section is the new Group in its own section
		play_area := PLAY_AREA & (blessed_grp.liberties | grp.liberties)
		data_area := DATA_AREA & (blessed_grp.liberties & grp.liberties)
		blessed_grp.liberties = play_area | data_area
	}

	// == Update liberties
	bb_unset_bit(&blessed_grp.liberties, hex_to_index(move.hex))
	for l in 0 ..< libs_cursor {
		bb_set_bit(&blessed_grp.liberties, hex_to_index(new_libs[l]))
	}

	// == Update the groupmap
	bbi := bb_make_iter(blessed_grp.tiles)
	for _, idx in bb_iter(&bbi) {
		game.groups_map[idx] = blessed_key
	}

	// this should be all.

	return
}
```

Now, what to do if it *is* a potential suicide? First of all, it is not possible to know whether it is a suicide or not without doing everything already done in `group_extend_or_merge` anyway. If it were a Suicide, the final, collasced Group would have no Liberties, and therefore it would be an easy decision to simply flip its Controller and swap its allegiance.

The problem lies in how to merge it with its capturing Group(s). It needs to be merged to maintain an accurate count of Liberties, as the count of Liberties is clearly being used to test whether it needs to be captured or not!! (Not in *this* move. but in subsequent ones.)

The naïve approach is to iterate over every Tile member of the Group and every Connection of said Tile, and compile a list of opponent-controlled neighbors, then check the Group membership of *those* Tiles, (similarly to how `nbr_friend_grps` is used,) then do the merging routine again, but this time for the opponent.

The *other* naïve approach is to add a third `Bitboard` to `Group` to track its surrounding enemies and just iterate over those.

Both solutions bother me for different reasons. The first seems like a time and CPU cycle waste (as I am already iterating over the same Tile's flags multiple times ..), and the second seems adds an extra knob to keep track of.

## Rethinking Groups

One idea to explore is to change `Group`'s representation entirely, away from Bitboards. (Yes I spent a lot of time talking about Bitboards, and they're really cool, but hear me out). Throughout the program, the state of the game has been tracked through a number of `[CELL_COUNT]` arrays, of variable things, with trivial conversion from a human-readable `Hex` to an index within these arrays. So what's another `[CELL_COUNT]` array? But what would it be an array *of*?

Currently, as far as a Group is concerned, a location can be one of three states: Either a member Tile, a Liberty, or an enemy Connection. That is an enum! More things can be added to it later as needed.

```odin
Hex_State :: enum u8 {
	Empty            = 0x00, // numbers chosen for a reason
	Liberty          = 0x01,
	Enemy_Connection = 0x11,
	Member_Tile      = 0x13,
}
```

And this how the `Group` using this enum would look like:

```odin
Rethought_Group :: struct {
	state:      [CELL_COUNT]Hex_State,
	extendable: bool, // sadly no clean niche to hide that
}
```

Much cleaner! Surprisingly, Odin allows bitwise OR over enumerations.[^7] If the resulting value has no tag assigned, it becomes a `BAD_ENUM_VALUE` and may potentially wreck the program. But if the numbers are assigned appropraitely, it can be made to always have a valid value.

Thinking through this, it is clear that `.Empty` with any other tag should be, well, that other tag. `.Liberty`, being essentially an empty cell as well, with any of the other two tags should produce the other tag. `.Member_Tile` and `.Enemy_Connection` overlap when capturing groups, so Enemies should be converted to Members. Here is the printed `OR` table:

```
        Empty   Liberty   Enemy   Member

Empty   Empty   Liberty   Enemy   Member
Liberty	        Liberty   Enemy   Member
Enemy                     Enemy   Member
Member                            Member
```
Great. Looks good to me. Let's roll with it. This is how `Group` works now. Should I need more states I shall think of other clever numbers to use. Then follow the compiler's erros about missing fields and correct those as needed.

Compiler driven development !!

## Back to Updating Game State

### Back to `group_attach_to_friendlies`

This above change makes the merging process much simpler. It also allowed me to delete the entire `bitboard.odin` file! Simply following the compiler's errors leads me to this version of `group_extend_or_merge`:

```odin
@(private)
group_extend_or_merge :: proc(move: Move, game: ^Game) -> (ok: bool = true) {
	// ----- snip: same as before, for now.

	// == Identify first group
	blessed_key := nbr_friend_grps[0]
	assert(slotmap_contains_key(friendly_grps, blessed_key))
	blessed_grp := slotmap_get(friendly_grps, blessed_key)
	blessed_grp.state[hex_to_index(move.hex)] = .Member_Tile

	// == Merge other groups with first group
	for i in 1 ..< nfg_cursor {
		assert(slotmap_contains_key(friendly_grps, nbr_friend_grps[i]))
		grp := slotmap_remove(friendly_grps, nbr_friend_grps[i])
		defer free(grp)

		blessed_grp.state |= grp.state
		blessed_grp.extendable &= grp.extendable
	}

	// == Update liberties
	for l in 0 ..< libs_cursor {
		blessed_grp.state[hex_to_index(new_libs[l])] |= .Liberty
	}

	// == Update the groupmap
	for slot, idx in blessed_grp.state {
		if slot == .Member_Tile do game.groups_map[idx] = blessed_key
	}

	return
}
```

Now, merging groups membership and liberties also merges their enemy connections as well. No additional bookkeeping! The check for potential Suicide earlier can now be removed, and writing this (larger and larger) procedure can continue:

```odin
	// ----- snip: same as before + declaring pointers to enemy groups

	// == Update liberties
	for l in 0 ..< libs_cursor {
		blessed_grp.state[hex_to_index(new_libs[l])] |= .Liberty
	}

	if tile_liberties_count == 0      // Potential Suicide
	   && group_life(blessed_grp) == 0 // Definite Suicide
	{
		// no insertion into the enemy slotmap .. there is merging to be done!
		cursed_grp := slotmap_remove(friendly_grps, blessed_key)
		defer free(cursed_grp)

		// Scratchpad 
		nbr_enemy_grps := make([dynamic]Sm_Key)
		defer delete(nbr_enemy_grps)

		for loc, idx in cursed_grp.state {
			#partial switch loc {
			case .Member_Tile:
				tile_flip(&game.board[idx])
			case .Enemy_Connection:
				key := game.groups_map[idx]
				if !slice.contains(nbr_enemy_grps[:], key) {
					append(&nbr_enemy_grps, key)
				}
			}
		}
		// == same steps as before
		assert(len(nbr_enemy_grps) > 0) // or there is Oscillation
		blessed_key = nbr_enemy_grps[0]

		assert(slotmap_contains_key(enemy_grps, blessed_key))
		blessed_grp = slotmap_get(enemy_grps, blessed_key)

		blessed_grp.state |= cursed_grp.state

		for i in 1 ..< len(nbr_enemy_grps) {
			assert(slotmap_contains_key(enemy_grps, nbr_enemy_grps[i]))
			grp := slotmap_remove(enemy_grps, nbr_enemy_grps[i])
			defer free(grp)

			blessed_grp.state |= grp.state
		}

		// check if blessed_grp is extendable
		extendable := true
		for loc in blessed_grp.state {
			if loc == .Enemy_Connection {
				extendable = false
				break
			}
		}
		blessed_grp.extendable = extendable
	}

	// == Update the groupmap
	for slot, idx in blessed_grp.state {
		if slot == .Member_Tile do game.groups_map[idx] = blessed_key
	}

	return
}
```

Change the procedure's name to `group_attach_to_friendlies`, and *now* it is done. All this merging logic should really be refactored and `DRY`'d, but I will let it be for now. This one procedure end up about 130 lines of code.

### `group_attach_to_enemies`

Similarly to attaching to friendlies, this move can either be a nothing, a capture, or a suicide. The logic of the `friendlies` procedure might need to be repeated in this one. And the checks done and the data collected would also need to be repeated. 

So why separate it at all? The idea was it would simplify handling, but it does not seem to do that. So, rethinking the move handling, allow me to try summarising the logic that *actually* needs to be done (this was revised in tandem with writing the code in the next section):

1. Create these trackers:
	1. Liberties of the newly placed tile,
	2. Neighboring, connected Friendlies (tracking Groups), and
	3. Neighboring, connected Enemies (tracking locations).
2. Iterate over all Connected Sides of the newly placed tile, and fill in the trackers as needed,
3. For every Friendly connected Group, merge them together. (If none are connected, the Tile starts its own Group with marked Liberties and Enemy neighbors.) Mark this group as "Blessed".
4. Iterate over neighboring Enemy Groups, if any have a Liberty count of 0: [^8]
	1. They are flipped and merged with the Blessed Group.
	2. Iterate over connections of the Blessed Group, and merge with it any friendly Groups found.
	3. Move is over. (This is because we already filtered for legal moves, or a check for Oscillation would be needed.)
5. The Blessed Group is checked for Liberty count. If it is 0, it is captured (flipped) and merged with its surrounding Enemy Groups.
6. Done

Note that I am assuming here that this is not a recursive operation. Here is the assumption: A new Tile placement that has no liberties *but* takes away the last liberty of an enemy Group captures it. There is no need to check if the surrounding friendly Groups (that surrounded the surrounding Enemy Groups) would also have no Liberties, because if they had no Liberties they would not exist! A lot of weight is placed right now on the correctness of `game_regen_legal_moves`, which is still delayed for later.

### `game_update_state_inner` - Second Draft

A monstrous 230-ish lines of code which could really use some refactoring. This drags on but I made my best to comment my thoughts throughout.

```odin
@(private)
game_update_state_inner :: proc(move: Move, game: ^Game) {
	// Bug tracker
	tile_liberties := card(move.tile & CONNECTION_FLAGS)
	tile_liberties_countdown := tile_liberties

	// Friendliness tracker
	tile_control := move.tile & {.Controller_Is_Host}

	// Scratchpad: Found friendly Groups
	nbr_friend_grps: [6]Sm_Key
	nfg_counter: uint

	// Scratchpad: Found Enemy Groups
	nbr_enemy_tiles: [6]Hex
	net_counter: uint

	// Scratchpad: New Liberties
	new_libs: [6]Hex
	libs_counter: uint

	for flag in move.tile & CONNECTION_FLAGS {
		neighbor := move.hex + flag_dir(flag)
		nbr_tile := board_get_tile(&game.board, neighbor) or_continue

		if tile_is_empty(nbr_tile^) {
			new_libs[libs_counter] = neighbor
			libs_counter += 1
		} else if nbr_tile^ & {.Controller_Is_Host} == tile_control {
			// Same Controller
			tile_liberties_countdown -= 1

			// record Group of neighbor tile.
			key := game.groups_map[hex_to_index(neighbor)]
			if !slice.contains(nbr_friend_grps[:], key) {
				nbr_friend_grps[nfg_counter] = key
				nfg_counter += 1
			}
		} else {
			// Different Controller
			tile_liberties_countdown -= 1

			nbr_enemy_tiles[net_counter] = neighbor
			net_counter += 1
		}
	}

	assert(tile_liberties_countdown >= 0, "if this is broken there is a legality bug")

	// == Are we the Baddies?
	friendly_grps: Slot_Map
	enemy_grps: Slot_Map
	if tile_control == {} {
		// Guest Controller
		friendly_grps = game.guest_grps
		enemy_grps = game.host_grps
	} else {
		// Host Controller
		friendly_grps = game.host_grps
		enemy_grps = game.guest_grps
	}

	// The placed Tile's Group
	blessed_key: Sm_Key
	blessed_grp: Sm_Item
	if nfg_counter == 0 {
		blessed_grp = new(Group)
		blessed_key = slotmap_insert(friendly_grps, blessed_grp)

		if net_counter == 0 {
			blessed_grp.extendable = true
		}
	} else {
		blessed_key = nbr_friend_grps[0]
		assert(
			slotmap_contains_key(friendly_grps, blessed_key),
			"Friendly slotmap does not have friendly Key",
		)
		blessed_grp = slotmap_get(friendly_grps, blessed_key)

		// == Merge other groups with blessed group
		for i in 1 ..< nfg_counter {
			assert(
				slotmap_contains_key(friendly_grps, nbr_friend_grps[i]),
				"Friendly slotmap does not have friendly Key",
			)
			temp_grp := slotmap_remove(friendly_grps, nbr_friend_grps[i])
			defer free(temp_grp)

			blessed_grp.state |= temp_grp.state
			blessed_grp.extendable &= temp_grp.extendable
		}
	}
	blessed_grp.state[hex_to_index(move.hex)] |= .Member_Tile

	defer {
		// == Update the groupmap
		for slot, idx in blessed_grp.state {
			if slot == .Member_Tile do game.groups_map[idx] = blessed_key
		}
	}

	// == Update liberties
	for i in 0 ..< libs_counter {
		blessed_grp.state[hex_to_index(new_libs[i])] |= .Liberty
	}
	// == Update Enemy neighbors for blessed group
	for i in 0 ..< net_counter {
		blessed_grp.state[hex_to_index(nbr_enemy_tiles[i])] |= .Enemy_Connection
	}

	// == register surrounding Enemy Groups of blessed Group
	surrounding_enemy_grps := make([dynamic]Sm_Key)
	defer delete(surrounding_enemy_grps)

	for slot, idx in blessed_grp.state {
		(slot == .Enemy_Connection) or_continue
		key := game.groups_map[idx]
		if !slice.contains(surrounding_enemy_grps[:], key) {
			append(&surrounding_enemy_grps, key)
		}
	}

	// == if there are no surrounding enemy groups there is nothing more to do
	if len(surrounding_enemy_grps) == 0 {
		assert(
			group_life(blessed_grp) > 0,
			"newly formed groups must have liberites or enemy connections",
		)
		blessed_grp.extendable = true
		return
	}

	// == these are the friendly groups that surround the dead enemy groups.
	level_2_surrounding_friendlies := make([dynamic]Sm_Key)
	defer delete(level_2_surrounding_friendlies)

	// == go over surrounding enemy groups to see if they're dead.
	capture_occurance := false
	for key in surrounding_enemy_grps {
		assert(slotmap_contains_key(enemy_grps, key), "Enemy slotmap does not have enemy Key")
		temp_grp := slotmap_get(enemy_grps, key)
		temp_grp.state[hex_to_index(move.hex)] |= .Enemy_Connection // this is probably correct

		// Enemy Group is dead
		(group_life(temp_grp) == 0) or_continue
		capture_occurance = true

		cursed_grp := slotmap_remove(enemy_grps, key)
		defer free(cursed_grp)

		for slot, idx in cursed_grp.state {
			#partial switch slot {
			case .Member_Tile:
				tile_flip(&game.board[idx])
			case .Enemy_Connection:
				key := game.groups_map[idx]
				if !slice.contains(level_2_surrounding_friendlies[:], key) {
					append(&level_2_surrounding_friendlies, key)
				}
			}
		}

		// CAPTURE
		blessed_grp.state |= cursed_grp.state
		blessed_grp.extendable &= cursed_grp.extendable
	}

	// == merge level 2 surrounding friendlies into blessed group
	for key in level_2_surrounding_friendlies {
		assert(slotmap_contains_key(friendly_grps, key))
		temp_grp := slotmap_remove(friendly_grps, key)
		defer free(temp_grp)

		blessed_grp.state |= temp_grp.state
		blessed_grp.extendable &= temp_grp.extendable
	}

	// == if there is a capture, it is done.
	if capture_occurance do return

	// == if blessed group's liberties larger than 0, it is done capturing
	if group_life(blessed_grp) > 0 do return

	// == Now the blessed group has converted.

	cursed_grp := slotmap_remove(friendly_grps, blessed_key)
	defer free(cursed_grp)

	new_family := make([dynamic]Sm_Key)
	defer delete(new_family)

	for loc, idx in blessed_grp.state {
		#partial switch loc {
		case .Member_Tile:
			tile_flip(&game.board[idx])
		case .Enemy_Connection:
			key := game.groups_map[idx]
			if !slice.contains(new_family[:], key) {
				append(&new_family, key)
			}
		}
	}

	assert(len(new_family) > 0, "Oscillation")

	blessed_key = new_family[0]
	assert(slotmap_contains_key(enemy_grps, blessed_key), "Enemy key is not in enemy map")

	blessed_grp = slotmap_get(enemy_grps, blessed_key)
	blessed_grp.state |= cursed_grp.state

	for i in 1 ..< len(new_family) {
		assert(slotmap_contains_key(enemy_grps, new_family[i]))
		temp_grp := slotmap_remove(enemy_grps, new_family[i])
		defer free(temp_grp)

		blessed_grp.state |= temp_grp.state
	}

	// check if new blessed group is extendable
	extendable := true
	for loc in blessed_grp.state {
		if loc == .Enemy_Connection {
			extendable = false
			break
		}
	}
	blessed_grp.extendable = extendable

	return
}
```

## `game_regen_legal_moves`

This is the current state of this function, which a lot is riding on:

```odin
@(private)
game_regen_legal_moves :: proc(game: ^Game) {
	clear(&game.legal_moves)

	// todo: build them again
}
```




---

[^1]: As mentioned earlier, the Blank tile is not used in the game. This permits using a sentinel value of `0` (or really any value with the smallest six bits set to `0`) to mark an empty cell. Since Tiles are a `u8` bitset anyway, why waste memory on pointers (which are wider), or `Maybe`, which is at least an extra byte in size? I am not thinking *too* hard about performance (I know nobody will use this), but it is an interesting constraint to keep in mind.

[^2]: I should be careful not to index into `Board` directly, however.

[^3]: I love that `move` is not a keyword here, which is really annoying in Rust.

[^4]: Generatinal arena, generational handles, handle-based map, a rose by any other name.

[^5]: It is Oscillation because the resulting Group has no Liberties, and therefore has no clear Controller, so it *oscillates* between both colors. This is way the Blank tile has no role in the game: it automatically oscillates. 

[^6]: Technically, only a record of whether the last move *was* a pass is needed, but `last_move` is semantically clearer than a `last_move_was_a_pass` (or `pass_ends_the_game` or `the_end_is_nigh`) boolean or a `player_who_last_made_a_move` enum field. It may also be useful to highlight the last move in a GUI.

[^7]: Rust would *totally* yell at me. Then tell me to implement the trait to define the behavior myself.

[^8]: Freeling does not specify in which order captures are processed. I am assuming here the order is the same as Go. Anyway, all this needs to be verified later once (if?) the engine is implemented.
