+++
title = "Parsing Hearthstone Deck Codes"
date = 2024-11-14
updated = 2024-11-26
+++

In the last few days I have been mucking around with [`Mimiron`](@/projects/Mimiron%20Discord%20Bot.md), my Hearthstone API library and Discord bot. One of the main functions of the bot, and the one it is used for the most, is parsing Hearhstone deck codes and presenting them in a pretty fashion.

I had a working parser for a while, but I wanted to experiment with it so I decided to rewrite it in [nom](https://docs.rs/nom/latest/nom/). Here are both parsers.

## Deck Code Format

The deck code fromat is explained [by HearthSim](https://hearthsim.info/docs/deckstrings/), the company behind [HSReplay](https://hsreplay.net) As far as I know, there is no official specification of deck codes encoding, so HearthSim's description is what everyone relies on. The page is actually outdated, but their example implementations (in [C#](https://github.com/HearthSim/HearthDb/blob/5258360a9c0411334e4f5a6dcb876b085d89cd3a/HearthDb/Deckstrings/DeckSerializer.cs#L169)m [Python](https://github.com/HearthSim/python-hearthstone/blob/82e7c71ebdb6c8521b168d6f0ca418ff56e0fd5d/hearthstone/deckstrings.py#L109)m and [TypeScript](https://github.com/HearthSim/hearthstone-deckstrings/blob/6fa4d9cb896841a6246130a59b52a7cf752ca69f/src/index.ts#L123)) explain it clearly enough.

- The deck code is essentially a `base64` encoded series of integers.
- These integers are encoded as unsigned `varint`s, [variable width integers](https://en.wikipedia.org/wiki/Variable-length_quantity)[^wikipedia].
- The first two integers are always `0` and `1`. They happen to be a byte each, simplifying matters.
- The third integer is the Format. `1` is Wild, `2` is Standard, `3` is Classic, and `4` is Twist.

[^wikipedia]: This Wikipedia page gives a terrible and obtuse explanation. The implementation turned out to very simple.

After `Format`, every block of integers starts with a count, then a listing of items, usually Card IDs (often called DBF IDs). Card IDs are unique numerical IDs that uniquely[^cardid] identify each card in the game. The blocks are as follows:

[^cardid]: IDs can be deleted from the game, leading to outdated deck codes out there, but that's out of scope of this article. In `Mimiron`, I work around that by looking them up in the third party database everyone else uses: [HearthSim's](https://hearthstonejson.com), and "canonilizing" the IDs. My bot is somewhat unique for relying first on [Blizzard's official API](https://develop.battle.net/documentation).

1. Hero. Conveniently, this block only ever has one element.
2. Single-copy cards.
3. Double-copy cards.
4. N-copy cards. Each item in this block is a Card ID followed a by a count. This is only used for weird one-off formats or Arena codes.
5. A block of blocks? Unclear.

The block of blocks is not clearly documented. Sussing it out from existing parsers, it only has one item, which is the Sideboard block. The Sideboard block starts with a count, as usual, but each item is a pair of Card IDs. The first is the card itself, and the second is the Card ID of the Card it is *under*. There are, as of this time, only two Sideboard cards in the game.

After that is whatever. It apparently does not matter. It seems to usually be two `0`s? hitherto undefined blocks? WHo knows.

### Example

Take this deck code for example:

```
AAECAdrJBgbHpAaopQbUpQaX4Qat4Qaq6gYM5p4GpKcGqKcG66kGw74GzsAG0MAG0dAGmOEGmOIG5OoGjfgGAAED9bMGx6QG97MGx6QG6N4Gx6QGAAA=
```

This is the list of numbers it represents, with comments.

```sh
0
1
2       # Format
1
107738  # Hero ID
6       # Single Copy IDs
102983  # <-- Note this ID is the same as the Sidebaord Card later.
103080
103124
110743
110765
111914
12      # Double Copy IDs
102246
103332
103336
103659
106307
106574
106576
108625
110744
110872
111972
113677
0       # N-Copy IDs. This is only used in weird formats.
1       # no idea really
3       # Sideboard IDs
104949
102983
104951  # first part of pair: Card IN Sideboard
102983  # The Sideboard Card
110440
102983
0       # who knows? Future proofing maybe?
0
```

It is a very simple format really.

## Desired Output and Shared Elements

This is the Object which I need to fill up.

```rust
struct Data {
    format: Format,
    hero: usize,
    cards: Vec<usize>,
    sideboard_cards: Vec<(usize, usize)>,
}
```

And this is the `Format` type with its conversion from a `usize`.

```rust
#[derive(Default)]
enum Format {
    Standard,
    #[default]
    Wild,
    Classic,
    Twist,
}
impl TryFrom<usize> for Format {
    type Error = (); // Petend it is something useful.
    fn try_from(value: usize) -> Result<Self, Self::Error> {
        Ok(match value {
            1 => Self::Wild,
            2 => Self::Standard,
            3 => Self::Classic,
            4 => Self::Twist,
            _ => return Err(()),
        })
    }
}
```

To be able to implement and test the different parsers without much of the surrounding code interfering with it, I decided to wrap the whole thing into a tiny executable to be able to pass the codes directly on the command line. I'd also need the `base64` crate.

This is the venerable `Cargo.toml`:

```toml
[package]
name = "deck_parser"
version = "0.1.0"
edition = "2021"

[dependencies]
base64 = "0.22.1"
pico-args = "0.5.0"
# more to xome
```

And this is `main.rs`, without the types defined above:

```rust
use base64::{prelude::BASE64_STANDARD, Engine};
use pico_args::Arguments;

fn main() {
    let mut args = Arguments::from_env();
    let raw = args.contains("--raw");
    let Ok(decoded) = args.free_from_fn(|s| BASE64_STANDARD.decode(s)) else {
        eprintln!("Not base64 encoded");
        std::process::exit(1);
    };

    if raw {
        print_nums(&decoded);
    } else {
        if print_data(&decoded).is_err() {
            eprintln!("Invalid Deck Code");
            std::process::exit(1);
        }
    }
}

// Print the List of Numbers
fn print_nums(decoded: &[u8]) {
    todo!()
}

// Print the Raw Data obejct.
fn print_data(decoded: &[u8]) -> Result<(), Box<dyn Error>> {
    todo!()
}
```

The `--raw` flag is mostly to test the `varint` reader, and check it always gets the same output.

## Straightforward Parser

This is the first parser that I implemented. I used the [`integer_encoding` crate](https://docs.rs/integer-encoding/latest/integer_encoding/) because I dod not want to think about how `varint` encoding works, and it presents a nice API. It required wrapping the buffer in a `std::io::Cursor`, but that was it. It reads right from there.

Consdiering the `varint` parsing is relegated to a crate, `print_nums` is fairly straightforward.

```rust
use integer_encoding::VarIntReader; // way at the top of the file.
use std::io::Cursor;

fn print_nums(decoded: &[u8]) {
    let mut buffer = Cursor::new(decoded);

    while let Ok(n) = buffer.read_varint::<usize>() {
        println!("{n}");
    }
}
```

And one can quickly test that it works with the following command. Should you run it with the deck code earlier, you will find it lists the same numbers.

```sh
cargo run -- --raw <insert-deck-code-here>
```

Excellent. This means that `integer_encoding` does its job splendidly.

The second one is a bit more involved, but also fairly straightforward. `read_varint` advances the `Cursor`, so all that is needed is to assign the values in the correct order. There is a small footgun hiding in the API, however, which is that unsigned `varint`s have a different encoding for signed `varint`s. Considering literals are, by default, `i32`, Rust's type inference works against you here.

```rust
fn print_data(decoded: &[u8]) -> Result<(), Box<dyn Error>> {
    let mut buffer = Cursor::new(&decoded);

    // Format is the third number.
    buffer.set_position(2);
    let format = buffer
        .read_varint::<usize>()?
        .try_into()
        .unwrap_or_default();

    // Hero ID is the fifth number.
    buffer.set_position(4);
    let hero = buffer.read_varint()?;

    let mut cards = Vec::new();

    // Single copy cards
    let count = buffer.read_varint()?;
    for _ in 0_usize..count { // <-- Type Inference footgun!!
        let id = buffer.read_varint()?;
        cards.push(id);
    }

    // Double copy cards
    let count = buffer.read_varint()?;
    for _ in 0_usize..count {
        let id = buffer.read_varint()?;

        cards.push(id);
        cards.push(id); // twice
    }

    // N-copy cards
    let count = buffer.read_varint()?;
    for _ in 0_usize..count {
        let id = buffer.read_varint()?;
        let n = buffer.read_varint()?;

        for _ in 0_usize..n {
            cards.push(id);
        }
    }

    // Sideboard cards. Are they always available?
    let mut sideboard_cards = Vec::new();
    if buffer.read_varint::<usize>().is_ok_and(|i| i == 1) {
        let count = buffer.read_varint()?;
        for _ in 0_usize..count {
            let id = buffer.read_varint()?;
            let sb_id = buffer.read_varint()?;

            sideboard_cards.push((id, sb_id));
        }
    }

    let result = Data {
        format,
        hero,
        cards,
        sideboard_cards,
    };

    println!("{:#?}", result); // Don't forget to dervie Debug

    Ok(())
}
```

Not a very pretty printing. But it works. Good base implementation.

## Unsigned Variable Integers

When I started trying to rewrite the parser in `nom`, I looked for an extension `nom` crate that can read `varint`s. I found two! Reading through their source, however, I realized that it is really simple, and decided to bring it in the code base myself.

Unsigned `varint`s' encoding is, as it turned out, fairly simple. Honestly it is actually easier to just show the code than explain how the encoding works.

```rust
fn parse_varint(input: &[u8]) -> Option<usize> {
    // a usize can only be as big as 9 bytes in varint encoding
    const MAX_BYTES: usize = 9;

    let mut num = 0;

    for (idx, &byte) in input.iter().take(MAX_BYTES).enumerate() {  
        // 7 bits from each byte are used
        let byte = byte as usize & 0x7F;
        let offset = idx * 7; 
        num |= byte << offset;

        // The last byte's most significant bit is 0.
        if byte & 0x80 == 0 {
            return Some(num);
        }
    }

    None // Insufficient data or overflow
}
```

Signed `varint`s are a whole other beast, and they do not concern this article outside of the type inference footgun from the previous section.

## `nom` Parser

This is what actually prompted writing this article.[^nom] I already used `nom` elsewhere in `Mimiron`, and I wanted to exercise more with it.

[^nom]: If you are somehow this far into this post and do not know what `nom` is, it is a Rust parser combinator library, probably the first and most famous. *Combinators* are small higher order functions (read: functions that can take and/or output other functions: function functions, if you will), that can be composed together. Parser Combinator Libraries are, in a way, their own Domain Specific Language. [`nom`'s documentation has a nice intro](https://docs.rs/nom/latest/nom/#parser-combinators).

The `nom` way is all about small, composable functions. The clearest small composable function to do is `parse_varint`. I am not going to use the imperative version shown above, I am writing one using `nom`'s parser combinators and the standard library's `Iterator` combinators.

```rust
// no nom codebase is complete without a giant use statement on top
// this includes combinators for all the functions below
use nom::{
    bytes::complete::{
        take,      // takes a number of bytes unconditionally
        take_till, // takes until a condition is met and stops before
    },
    combinator::{
        cond,      // calls a parser only if a condition is met
        map,       // maps the result of a parser
        map_opt,   // maps the result of a parser to an Option
        opt,       // optional parser, for a value. returns Option
        verify,    // verifies the result satisfies a condition
    },
    multi::{
        length_count, // gets a number from first parser,
                      // then applies second parser that many times
        many0,        // repeats a parser until it fails. returns Vec
    },
    sequence::{
        pair,       // call first parser, then second parser.
        preceded,   // discards result of first parser
        tuple,      // same as pair, but up to 21 separate parsers
    },

    IResult,        // nom's Result type.
                    // Returns remaining input and successful results
                    // or an error with the input inside the Error.
};

fn parse_varint(input: &[u8]) -> IResult<&[u8], usize> {
    let is_last = |b| b & 0x80 == 0;
    let is_in_bounds = |p: &[u8]| p.len() < 9;
    let try_varint = |(p, lb): (&[u8], &[u8])| {
        // this is the same imperative code up above with some error checking
        p.iter()
            .chain(lb)
            .enumerate()
            .try_fold(0, |acc, (idx, byte)| {
                ((*byte as usize) & 0x7F)
                    .checked_shl(idx as u32 * 7)
                    .map(|n| acc | n)
            })
    };

    map_opt(
        pair(verify(take_till(is_last), is_in_bounds), take(1u8)),
        try_varint,
    )(input)
}
```

Same, but slightly more unreadable (I am going to have so much fun with this):

```rust
fn parse_varint(input: &[u8]) -> IResult<&[u8], usize> {
    map_opt(
        pair(
            verify(take_till(|b| b & 0x80 == 0), |p: &[u8]| p.len() < 9),
            take(1u8),
        ),
        |(p, lb): (&[u8], &[u8])| {
            p.iter()
                .chain(lb)
                .enumerate()
                .try_fold(0, |acc, (idx, byte)| {
                    ((*byte as usize) & 0x7F)
                        .checked_shl(idx as u32 * 7)
                        .map(|n| acc | n)
                })
        },
    )(input)
}
```

Using this small function to generate the list of numbers, going straight for the unreadable verion:

```rust
pub fn print_nums(decoded: &[u8]) {
    _ = many0(map(parse_varint, |n| println!("{n}")))(decoded);
}
```

The second function is a bit more involved, but it generally follows the structure of the previous section's solution. I think it is easy enough to follow.

```rust
pub fn print_data(decoded: &[u8]) -> Result<(), Box<dyn Error + '_>> {
    // starting from 2 as Format is the third number.
    let (rem, format) = map(
        parse_varint,
        |f| f.try_into().unwrap_or_default()
    )(&decoded[2..])?;

    // skip 1 byte for Hero ID.
    let (rem, _) = parse_varint(rem)?;
    let (rem, hero) = parse_varint(rem)?;
    let (rem, cards) = map(
        tuple((
            // single cards
            length_count(parse_varint, parse_varint),

            // double cards
            length_count(parse_varint, map(parse_varint, |id| [id; 2])),

            // n-count cards
            length_count(
                parse_varint,
                map(
                    pair(parse_varint, parse_varint), 
                    (id, n)| [id].repeat(n)
                ),
            ),
        )),
        |(v1, v2, vn)| {
            v1.into_iter()
                .chain(v2.into_iter().flatten())
                .chain(vn.into_iter().flatten())
                .collect()
        },
    )(rem)?;

    let (rem, c) = opt(parse_varint)(rem)?;
    let (_rem, sideboard_cards) = map(
        cond(
            c.is_some_and(|c| c == 1),
            length_count(parse_varint, pair(parse_varint, parse_varint)),
        ),
        Option::unwrap_or_default,
    )(rem)?;

    let result = super::Data {
        format,
        hero,
        cards,
        sideboard_cards,
    };

    println!("{result:#?}");

    Ok(())
}
```

But this can be made worse: the whole thing can be collapsed into one function. `rustfmt` is pulling a lot of weight here, to be honest.[^bug]

```rust
pub fn print_data(decoded: &[u8]) -> Result<(), Box<dyn Error + '_>> {
    let result = map(
        tuple((
            map(parse_varint, |f| f.try_into().unwrap_or_default()),
            preceded(parse_varint, parse_varint),
            map(
                tuple((
                    length_count(parse_varint, parse_varint),
                    length_count(
                        parse_varint,
                        map(parse_varint, |id| [id; 2])
                    ),
                    length_count(
                        parse_varint,
                        map(
                            pair(parse_varint, parse_varint),
                            |(id, n)| [id].repeat(n)
                        ),
                    ),
                )),
                |(v1, v2, vn)| {
                    v1.into_iter()
                        .chain(v2.into_iter().flatten())
                        .chain(vn.into_iter().flatten())
                        .collect()
                },
            ),
            preceded(
                verify(parse_varint, |i| *i == 1),
                length_count(parse_varint, pair(parse_varint, parse_varint)),
            ),
        )),
        |(format, hero, cards, sideboard_cards)| Data {
            format,
            hero,
            cards,
            sideboard_cards,
        },
    )(&decoded[2..])?
    .1;

    println!("{:#?}", result);

    Ok(())
}
```

[^bug]: 2024-11-26 update: I found out that this code has a bug: it rejects valid decks without a sideboard. How would one fix that?

One tiny thing before the end. Note the `+'_` bound on the return type. Since `nom`'s errors contain the input, which is a reference, converting the errors can sometimes cause Rust's type inference to declare that your functions expect a `'static` lifetime. For example, here, when the `+'_` bound is removed, or [when converting `nom`'s errors to `anyhow`'s errors using the `?` operator.](https://www.reddit.com/r/rust/comments/1goc61r/nom_implemented_parser_is_demanding_i_use_a/). It is not a footgun, per se, as the compiler yells at you. But the error is frankly inscrutable unless you know what you are looking for.

## Final Thoughts

Rewriting in nom was a bit easier than I thought it would be. And even the compact "one-liner" is more readable than I thought. The hardest part is probably how to translate the problem into `nom`'s API. (Which I struggled with when I considered using `nom` for [Advent of Code](https://adventofcode.com)). Who knows? Maybe this year.

If you would like to do this with another Rust parser library (`chumsky` or `winnow` or `combine` or `pest`), please do share, and I will happily link it here.
