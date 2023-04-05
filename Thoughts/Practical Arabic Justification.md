# Practical Arabic Justification

Here are some links that you might want to look at:
- [Deficiencies of Handling Arabic Script in OpenType. Aida Sakkal and Mamoun Sakkal. Tech Talks 2021](https://www.youtube.com/watch?v=Ai4dgLpFMx8). Mamoun Sakkal is a rather well known Arabic type designer. The video goes over the practical problems of justification in detail.
- [On Arabic justitifcation part 2 by Titus Nemeth](https://research.reading.ac.uk/typoarabic/on-arabic-justification-part-2-software-implementations/). (Look at parts 1 and 3 as well).
- [About Digital Khatt.](https://digitalkhatt.org/about) Real world (as it gets) examples of some of the ideas of Arabic justification. This is a public github project you can find [here](https://github.com/DigitalKhatt).
- [Making JSTF better by Simon Cozens on TypeDrawers](https://typedrawers.com/discussion/3465/making-jstf-better)
- [Discussion at HarfBuzz Github - Support for jalt feature and target length](https://github.com/harfbuzz/harfbuzz/discussions/3252) I will save you a click: "HarfBuzz currently does not implement justification, because it's a hard problem, and is under-specified in OpenType." Here is a [follow up discussion: API for buffer target length](https://github.com/harfbuzz/harfbuzz/pull/3259) Useful context by all parties.
- More general and high level: [Approaches to Line Breaking, W3C](https://www.w3.org/International/articles/typography/linebreak.en) and [Approaches to full justification](https://www.w3.org/International/articles/typography/justification).

## Intro

Justification is a nasty and difficult problem. When justifying over sufficiently long lines, the [Knuth-Plass algorithm](Linebreaking.md) is more than enough. However, when the lines are narrow, for example in a newspaper column, there just arent enough words in a line to make small changes to the space between them unnoticeable, and you need to take more drastic measures.

For English, and Latin (and Cyrillic?) script-based languages in general, the usual solution is [hyphenation](https://en.wikipedia.org/wiki/Hyphen#Justification_and_line-wrapping). This does not apply to every script. For example, the Japanese script have their own [justification algorithm](https://www.w3.org/TR/jlreq/?lang=en) that does not "just" add spaces between characters. Other scripts have their own rules going on.

## Arabic Hyphenation?

In Arabic, they *used to* hyphenate words around 1400 years ago. They, sadly, no longer do that, as it would make things so much simpler. In the following image, you can see the letter ا at the end (left) of the first line, and the word fragment لله at the start (right) of the second. And again at the end of the second and start of third. (Consecutive hyphenated lines - oh no). The break is not across syllabic boundaries, as an English speakers would expect, but between connected clusters.

![A page from the Quran](IMG_1070.jpg)

If you know anything about the Arabic script, you know two things: 1. It is written from right to left, and 2. in words, the letters are connected, but not all of them! (Letters are almost never connected *across* words.) So for example in the word ديمقراطية, meaning Democracy, well it is one word, but there are four "clusters". Like I said, we do not "hyphenate" (I am the term loosely here as there no actual hyphens) between those any more, but it is an important concept to keep in mind.

Which brings us back to the original problem statemnt: if there line width is so small that merely adjusting the spaces between words would be unseemly, what do you do? I am going to abuse my version of Microsoft Word a bit for this article. Here is an image of line of poetry justified using space stretching only, (Considering it is poetry, breaking words across lines is not even an option).

![Space Justification](just-calibri-spaces.png)

The lines are by [Ahmed Shawqi](https://en.wikipedia.org/wiki/Ahmed_Shawqi):

صوني جمالك عنا إننا بشر     
خلق التراب وهذا الحسن روحاني        
أو فابتغي فلكا تأوينه ملكا      
لم يتخذ شركا في العالم الفاني.

I have a few unhinged ideas. But let's talk about the usual approach first.

## Kashidas

If you have watched Mamoun Sakkal's video. linked above, on the technical limitations of OpenType when it comes to Arabic script, you would have noticed he spends a lot of the video talking about "Tatweel", or what is sometimes called by its Persian name, Kashida.

Kashia is a character on all Arabic keyboard. It does not have a meaning, by itself or by others, and it is merely decorartive. It is a typographinc (typesetting?) trick, that takes advantage of Arabic's connected nature. For an Arabic reader (and this applies to literally every language that uses the Arabic script), there is literally no difference between جمل and جـــــــمــــــــل. The second one simply looks longer. You can probably see where this is going.

The kashidas method basically inserts these characters betwen the individual letters of clusters.

![Kashida Justification](just-calibri-kashida.png)

The main problem with this is that it is *dumb*. So dumb it is useless. You can see how it adds some kashidas after بشر on the first line. (Reminder: "after" meand on the left.) This can be fixed, and this is probably a MS Word problem, but the other problem is a bit more subtle, and it is not clear with Calibri (as it was probably designed with this behavior in mind). Let us try with a different font by the same designer, Sakkal Majalla.

![Kashida Justification](just-majalla-kashida.png)

You can see it in the word التراب in the second line. The font uses (and abuses) an OpenType feature called Contextual Alternates, which basically means some letters look different when followed by specific letters, and it is all over the place in every half-decent Arabic font online.

The software, for whatever reason, *does not* call the shaper (HarfBuzz and the like) again after inserting the kashidas, and in fonts that use cursive shaping and do not sit on the baseline, you end up with this: (Noto Nastaliq Urdu)

![Kashida "Justification"](just-noto-kashida.png)

This is not a display typeface being misused by the way. It is meant for texts. 

The way type designers protect their fonts from this fate is by making the Kashida unicode character be 0-width (aka does nothing). The font [Sakkal Kitab](https://fonts.ilovetypography.com/fonts/sakkal-design/sakkal-kitab) (which costs $120 I do not have), does not even permit you to put Kashidas manually between any two letters. Only in specific places, and only manually (or by use of stylistic sets). You can try it yourself on the webpage by copying parts of the poem above and inserting the kashida in random places. The [Sakkal Kitab PDF Specimen](https://d3qx8f8l5aa3yc.cloudfront.net/images/sakkal_kitab_features.pdf) is a very nice brief read regarding justification.

Speaking of which:

## OpenType Features

OpenType has a feature *specifically* for justification. It is not very well supported, and in fact in mainstream software only InDesign makes use of it. It is called Justification Alternates, or `jalt` for short.`jalt`s, in theory, can make the word longer *or* shorter, tho I have not come across examples that make them shorter. I do not have InDesign, but Sakkal Majalla helpfully has a stylistic set that does what `jalt` does. This how the font looks normally:

![Space Justification](just-majalla-spaces.png)

This is how it looks with Stylistic Set 3 (or for this exercise, `jalt`s):

![jalt Justification](just-majalla-jalt.png)

You might reasonably point out that this does not solve the problem, and the text still has a lot of ugly spaces between the words. You would be forgetting an important fact here: I am intentionally abusing the software by forcing fully justified short lines. Poetry is a legit use case for justification, but it is an extreme case nonetheless. You can see a more real world example in Sakkal Kitab's Specimen, linked above. Here is a handy screenshot for you:

![jalt Justification](sakkal-kitab-specimen.png)

The real issue with `jalt` and font-driven justification, if you would call it that, is that apparently it is very underspecified, and the support, if it exists at all, is inconsistent.

You can see in the previous screenshot how in the paragraph with `jalt`, InDesign turns it on for *every word*, and not only when needeed. This, honestly, defeats the purpose. The text looks way too loose. The manual justification sample, on the right, is more measured when using the justification alternates. (Although because InDesign does not actually let you access `jalt` directly, it uses stylistic sets instead.) The following page is perhaps more illustrative (the middle and right columns are identical by the way):

![jalt Justification](sakkal-kitab-specimen2.png)

You can see here that the justification alternate, while not used *sparingly*, they are only used when the spaces would otherwise be too loose. You might also notice that the typesetter (who I can only assume is Sakkal himself) never applied it for two consecutive words, and never more than two times in a line. The choice of *which* words to stretch is probably arbitrary, based on personal taste.

I believe this process can reasonably be automated.

### Using `jalt`s reasonably

First of all you need whether the font has `jalt`s or not. You could even ask the user if they would like to use any other font features for justification purposes, and if they'd have a priority.

For example, if the user specifies using `ss16`, `ss17`, `ss18`, `ss19`, and `ss20` for justification, in this order, use those. If they don't, use `jalt` if available.

Pseuorustcode time! I will be adjusting on the `build_line()` function from the [linebreaking](Linebreaking.md) article.

```rust
fn build_line(input: ShapedText,
              start_bp: u64,
              end_bp: u64,
              desired_width: f64) -> Result<Line, Error> {
    /*
    
    .. boxes and glues and min spaces and max spaces,
    same as before.

    */

    if sum_boxes + (count_glue * max_space) < desired_width {
        return Error::LineTooLoose;
    }
    if sum_boxes + (count_glue * min_space) > desired_width {
        return Error::LineTooTight;
    }

    let true_width = sum_boxes + (count_glue * pft_space);

    // someone doublecheck this next line please
    let actual_space_width =
        (desired_width - true_width)/count_glue + pft_space;

    // raw cost, before changes for hyphenation and the like
    let line_cost = Math::square(desired_width - true_width);
    
    //...
    // Here is where you place a bunch of code laying out the
    // glyphs (sorry boxes) into a Line object, with the 
    // actual_space_width you found, and return it!
    //...

    return Ok(final_line);
}

```