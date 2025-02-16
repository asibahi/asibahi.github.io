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

## Ideas for syntax

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

## The starting implementation.

To start work on this, it requires a starting minimal implementation. I opted to start with a simple document that containts only headings and paragraphs, and none of the other fancy features.

This would be the example document to render (I am using Markdown content tag to work within the constraints of the format, which is also sadly rendering left-to-right.)

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

## Starting Parser

