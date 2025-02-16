+++
title = "Warraq: Typesetting for Arabic Literature - Part 1"
description = "Work in progress."
date = "2025-02-15"
draft = true
+++

In an effort to try new and different things, I have decided to start working on a typsetting engine that is designed, specifically, for typesetting books of Arabic literature. Things like the Memories of Ali Al-Tantawi or Moontalk حديث القمر by Mustafa Sadiq Al-Rafi'i, or the Diwan of Al-Mutanabbi.

The feature set required is relatively small, but no existing typesetting system really supports it as well as desired. All existing typesetting systems that I know of are generalists: they aim to supprot everything, including math and scripting and invoices amd scholarly documents and chess books and architectural specifications and advertisement billboards. Meanwhile, things like [typesetting an Arabic poem](https://github.com/asibahi/simple-poem-typst), is an exercise in weird scripting gymnastics. Footnotes and margin notes are similarly cumbersome. Arabic justification in Typst does not, as of writing these words, use Kashida properly but instead the Latin-oriented method of changing spacing.

Not to diss on Typst and co. They are great projects for doing what they are trying to do. But they are not trying to do this specific thing. Besides, I need a bid project to waste my time on besides [Mimiron](../../projects/mimiron-discord-bot/). Hence, Warraq.

Warraq, in Arabic, is the "dealer in paper", the bookseller or the scribe. It is as fitting a name as any for a typesetting system.

## Implementation

This post is essentially a record of the thought process. Nothing is concrete. However, following the idea of [Writing a C Compiler](https://norasandler.com/2017/11/29/Write-a-Compiler.html) (which I plan to follow some time, perhaps even in parallel to writing these posts), I plan to start with a very minimal implementation, and add to it from there, instead of trying to implement everythng and the kitchen sink from the get go.

First, however, I have to, at least, put the goal posts. To work towards something.

## Features

In no particular order.

- Footnotes and/or Margin notes. What is a literature book without margin notes?
- Poetry rendering, allowing for both long-form poetry, stanzas, and modern poetry (think Sing of the Rain أنشودة المطر by Badr Shaker al-Sayyab).
- Column support. Text may be set in full page or in columns.
- Picture embedding. *Not* inline.
- Minimal color support. Only for pictures and, perhaps, text emphasis.
- Text emphasis uses bold, underline, and potentially different fonts and coloring.
- External configuration: Toml or something.
- PDF and PN G output. No HTML.

### Non-goals

- No scripting or math or code blocks.
- No English or LTR support (I do realize the irony of writing this post in English.)
- No HTML output
- No italics.
- No links and hyperlinks.

## Syntax Ideas

Syntax ix a dime a dozen. What is wrong with Markdown? I can always copy and evolve Markdown's syntax.

```md
# Chapter title
#> Subtitle

Normal text is just written as normal.
**This may be bold**.
__This may be underlined__
^^This may be differently colored^^ and ##this would be an in-line heading## (yes they exist). 

One new line is a soft wrap. Two new lines is a new paragraph.

## Heading Level 2

--- // horizontal separator, followed by a comment.
// does Markdown even have comments?

> block quote / epigraph
>> quoted person's name`(optional)

Footnotes[1]

[1:] Can also be rendered as margin notes.

// Comments maybe like this? What's below is poetry
// 1. for the first half. 1\ for the second half

1. Shall I compare thee to a summer's day
1\ Thou are more lovely and more temprate 
2. Rough winds do shake the darling buds of May
2\ And summer's lease hath all too short a date

Intermediary text goes here. Then afterwords the poemmay continue with the same justification
throughout. Carries on intellegently.

3. Sometime too hot the eye of heaven shines,
3\ And often is his gold complexion dimm'd.
4. And every fair from fair sometime declines,
4\ By chance of nature's changing course untrimm'd

```

---

## Starting Implementation.

To start work on this, it requires a starting minimal implementation. I opted to start with a simple document that containts only headings and paragraphs, and none of the other fancy features.

This would be the example document to render (I am using Markdown content tag to work within the constraints of the format, which is also sadly rendering left-to-right for the time being.)

```md
# عنوان

## عنوان فرعي
نص فقرة
ملتف يدويا

فقرة غير ملتفة يدويا

## عنوان آخر
فقرة أخرى
````

This is enough of a feature set to start working.

---

## Starting Parser

To start with a parser, I am using the venerable [`nom`](https://docs.rs/nom/latest/nom/). Version 8.0 was recently released with a different architecture, but it is generally the same `nom` I know and love.

As parsing immediately into a finished document makes no sense, I am going to parse the document into a list of `Element`s. These `Element`s can, perhaps, be rendered one at a time into the page as necessary.

```rust
#[derive(Debug)]
enum Element {
    Heading { title: String, level: usize },
    Paragraph (String),
    // والبقية تأتي
}
```

Now a bunch of type aliases to ease working with `nom`'s deeply generic code.

```rust
type Error<'s> = (&'s str, nom::error::ErrorKind);
type Emit = nom::OutputM<nom::Emit, nom::Emit, nom::Complete>;
type Check = nom::OutputM<nom::Check, nom::Emit, nom::Complete>;
```

The first one here, `Error`, is a simple type that implements the `ParseError` trait. It allows for easy debugging to find out where parsers would fail. Eventually it would be changed to another type, perhaps even `()`.

The second and third type aliases warrant more of an explanation. The central piece of the `nom8` architecture is the `Parser` trait, and its main method is `process`. `process` takes a genertic argument of the type `OutputM`, M standing for Mode. It has, itself, three generic arguments. The first one decides whether the *successful* result of the parser is genrated or omitted (`Emit` or `Check`, respectively.) The second one is the same but for the error type. These are useful for parsers which discard their results (like `recognize`) or ones which discard their errors (like `alt`), as it avoids generating the necessary code.[^complete]

[^complete]: The last one, `Complete`, is about error generation, to indicate whether errors should differentiate between a parser that failed because of potentially incomplete information, or failed because it just does not fit. I am not insterested in the streaming version, hence they are both `Complete`.

In fact, you will find that the implementation of the `parse_complete` method, is equivelant to `process::<Emit>`, the type alias I defined above. I could just use it, honestly, but for now I like this version better.

Oh, one more thing: any function that takes `&str` and returns a `nom::IResult` implements the `Parser` trait just fine. I am really shaving the yak overcomplicating things for myself right here.

```rust
fn parse_header<'s, E: ParseError<&'s str>>(i: &'s str) -> IResult<&'s str, Element, E> {
    // # Heading\n

    let (i, level) = many1_count(tag("#")).process::<Emit>(i)?;
    let (i, _) = space1(i)?;
    let (i, title) = terminated(not_line_ending, line_ending).process::<Emit>(i)?;
    let (i, _) = multispace0().process::<Check>(i)?;

    let elem = Element::Heading {
        title: title.trim().into(),
        level,
    };

    Ok((i, elem))
}
```

The parsing for headers is simple enough. It looks for any number of `#` characters (which indicate the level) that start a line, followed by one or more spaces, then followed by the actual header, truncated by the new line, then followed by whatever amount of whitespace until the beginning of the next element.

I like the `terminated(not_line_ending, line_ending)` bit. It is funny and self-explanatory.

```rust
fn parse_paragraph<'s, E: ParseError<&'s str>>(mut i: &'s str) -> IResult<&'s str, Element, E> {
    // yada yada\nyada\n\n
    // or
    // yada yada\r\nyada\r\n\r\n

    let normal_end = recognize(count(line_ending, 2));
    let last_end = recognize((line_ending, eof));

    let end = alt((normal_end, last_end));

    let (i, body) = recognize(many_till(anychar, end)).process::<Emit>(i)?;
    let (i, _) = multispace0().process::<Check>(i)?;

    let elem = Element::Paragraph(body.trim().to_string());
    Ok((i, elem))
}
```

In true Markdown fashion, a single new line does not a paragraph end. A paragraph ends with *two* new lines, or a new line followed by `eof`, end of file. As the `many` parser (used for `parse_document`) errors out on parsers that do not comsume any input (otherwise it would infinitely loop), it took a lot of trial and error to get it right. Also the two weird `recognize`s there are just so both `normal_end` and `last_end` would have the same `Output` type.

```rust
fn parse_document<'s, E: ParseError<&'s str>>(i: &'s str) -> IResult<&'s str, Vec<Element>, E> {
    many(1.., alt((parse_header, parse_paragraph))).process::<Emit>(i)
}

pub fn run(input: &str) {
    for elem in parse_document::<Error<'_>>(input).unwrap().1 {
        dbg!(elem);
    }
}
```

Yeah that's it. It just produces a list of elements found on the page., and prints them to console. I did not make much use of my type aliases so far, to be honest. I can always remove them later.

Running this over my sample document returns this output:

```
[src/lib.rs:71:9] elem = Heading {
    title: "عنوان",
    level: 1,
}
[src/lib.rs:71:9] elem = Heading {
    title: "عنوان فرعي",
    level: 2,
}
[src/lib.rs:71:9] elem = Paragraph(
    "نص فقرة\nملتف يدويا",
)
[src/lib.rs:71:9] elem = Paragraph(
    "فقرة غير ملتفة يدويا",
)
[src/lib.rs:71:9] elem = Heading {
    title: "عنوان آخر",
    level: 2,
}
[src/lib.rs:71:9] elem = Paragraph(
    "فقرة أخرى",
)
```

Excellent! I got 'em right where I want 'em. Now what?
