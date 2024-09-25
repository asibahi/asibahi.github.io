+++
title = "My Ideal Code Editor"
date = 2024-09-24
+++

I have been doing some thinking about what my ideal code editor should be. I currently use VSCode, both to edit this site and to play around with code. However, VSCode is actually too powerful for my needs. And it lacks one crucial feature (that's pretty much lacking from almost every text/code editor outside of TextEdit and Notepad): proper Arabic language and Right-to-Left (RTL) support.[^rtl]

[^rtl]: There are *Markdown* editors that support Arabic decently enough, such as [Bear](https://bear.app) and Obsidian. But those do not handle HTML and Sass and what goes into editing a website, even ones as simple as this.

I tried several editors (honestly none more than 5 minutes) and I always go back to VSCode because of how comfortable it is. I tried [Helix](https://helix-editor.com), but could not live in the Terminal or modal editing. Tried Zed .. and it honestly just felt weird. Looked at [CodeEdit](https://www.codeedit.app), which is a really interesting project that brings the Xcode aesthetic down to code editor level, but it is still a work in progress, and it still doesn't deal with RTL properly.[^cot]

[^cot]: 2024-09-26: I also came across [CotEditor](https://coteditor.com). Looks nice and has good CJK support, with vertical text.

This post is more for my own records than anything. Whenever I start on the great My Ideal Code Editor (Code name: MICE, or KTB, or whatever), I have a nice feature list to stick to.

## Has Not

MICE lacks the following features:

- An integrated Terminal. My usual terminal works just fine.
- An integrated Git GUI. I can learn a few Git commands on the terminal.
- An integrated remote connection. I am a hobbyist who works solo.
- Built-in support for any language. Focus merely on the text features.
- Any sort of preview or built-in compiler. With a possible concession to Markdown. (I really like VSCode's Markdown preview pane)

## Has

However, MICE has these nice features:

### High Level Features:
- Native UI. I am on a Mac. I would like a native Mac UI.
- Arabic text support. If I have an Arabic paragraph inside of an English document, I would like said paragraph to be right-aligned.
- Tabs. Directory pane. 
- A command line tool to open directories, like VSCode's `code .` command. No idea currently on what else it needs to do.
- Rely on the the system's spell check in `txt` and `md` files.

### Coding Features
- LSP support: To provide IDE capabilities for any language. Intellisense, Go To Definition, Rename Symbol, Inlay Hints, the works.
- Tree-Sitter support: Syntax Highlighting (with perhaps LSP fallback), Multi-Cursor stuff, Code folding, Indentation. (Maybe through WASM plugins)
- Syntax Highlighting can have [multiple typefaces in it](https://monaspace.githubnext.com). 
- Smart editor features like auto-closing parentheses and select-and-type-a-brace-to-surround-block.
- While no language support is Built-In, the default installation should come with the web's defaults: HTML, CSS, JavaScript, and some useful additions like Sass, Markdown, and some Templating language like [Tera](https://keats.github.io/tera/), the one used for this site.
- Easy to install "packages" for additional languages.

What else? I don't know. I might add to this page once I think of anything or start any project.
