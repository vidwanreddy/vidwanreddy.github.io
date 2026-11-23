---
layout: default
title: Linter Story
permalink: /linter-story/
---

## From "push, push, push" to 16 Rules: The Story of a Markdown Linter I Built for My Docs Team

Every technical writer knows the feeling. You've finished a page, the content is solid, and then the review comes back with fifteen comments, none of them about the content. "Add a blank line before this image." "This list needs three spaces after the asterisk, not one." "This note block isn't indented correctly, so it won't render." "This link is broken."

I spent a lot of time writing those comments, and receiving them. Eventually I decided a machine should do it instead. This is the story of how a weekend experiment became a VS Code extension that, by our estimate, saves a writer about 15 minutes of manual checking on every page.

There's one thing you should know up front: I'm a technical writer, and I don't know how to code. I've never written TypeScript, and I couldn't have written a single rule in this extension by hand. All of it was vibe coded. I described what I wanted in plain English, first to GitHub Copilot and later to Cursor, and the AI wrote the code. I tested it against real documents and kept asking until it was right.

A few years ago, a tool like this would have meant a ticket in a developer team's backlog, waiting behind product work. This time, the person who knew exactly what the tool should do was also the person who built it.

<!-- TODO: screenshot of the extension flagging issues in the Problems panel -->

## The gap that markdownlint didn't cover

We already used [markdownlint](https://github.com/DavidAnson/markdownlint), and it's excellent. But our documentation follows a house style built on MkDocs-style admonitions (`!!! note`, `!!! warning`), strict list spacing, and a `SUMMARY.md` file that acts as the table of contents for every docs site. markdownlint knows nothing about those conventions.

So the gaps added up:

*   It doesn't know that a list inside a `!!! note` block needs specific indentation, or the page renders as a mess.
*   It checks that same-page anchors exist, but not whether a link to *another* file actually points to a real heading.
*   It can't tell you that a page exists in the folder but nobody linked it from `SUMMARY.md`, which makes it invisible to readers.

Each of those rules was being enforced by humans squinting at whitespace. That's slow, and as anyone who's tried to tell two spaces from three by eye knows, unreliable.

## Day 1: a Friday night scaffold

The first commit landed on 18 July 2025. It was the stock VS Code extension scaffold: TypeScript, esbuild, a "Hello World" command, and a 461-line `test.md` I'd filled with every formatting mistake I could think of.

The first real rule came later that night, at 11:42 PM. It was a simple check that every paragraph has a blank line before and after it. It was about 50 lines of TypeScript that looped over each line of the document and pushed a warning when the neighbours weren't blank:

```typescript
if (line > 0) {
    const prevText = document.lineAt(line - 1).text.trim();
    if (prevText !== '') {
        diagnostics.push(
            new vscode.Diagnostic(
                new vscode.Range(line, 0, line, text.length),
                'Paragraph should have a blank line before it.',
                vscode.DiagnosticSeverity.Warning
            )
        );
    }
}
```

Seeing that first yellow squiggle appear in my editor was the hook. The tool was clearly *possible*, and it wasn't even hard.

## Day 2: "push", "push", "push", "so far so good"

If you read the commit history from the next day, you'll find three commits in a row with the message `push`, followed by one called `so far so good`.

I'm not proud of those commit messages, but I like what they capture. That was the day I stopped planning and just built. In a single Saturday I added rules for blank lines around images and around admonitions, and a 200-line rule for list marker spacing, the one I'd wanted most. Our style says unordered lists get three spaces after the `*` and ordered lists get two after `1.`. Now the editor said so too.

That first paragraph rule didn't survive, by the way. It turned out to be too broad and flagged things that were perfectly fine. A few weeks later I deleted it in favour of more targeted rules. It was the first of several lessons in *what not to lint*.

## The summer sprint

The next evening, Sunday 20 July, was the most productive few hours of the whole project. Between 7 PM and midnight I added:

1.  Internal link validation, including checking that `file.md#some-heading` actually matches a heading in `file.md`.
2.  External link validation, which sends a real HTTP request to every `http(s)://` link.
3.  A cache and a rate limiter for those external checks, so I wasn't hammering anyone's server every time I typed a character.
4.  An indentation rule for content inside admonitions.

Over the next two months the rule count kept growing: broken image links, table formatting, code blocks without a language tag, blank lines around code blocks inside admonitions. I even named the branch `9th-rule-and-so-on`, because by then I'd accepted it wasn't going to stop at nine.

In August I gave it a proper identity: a logo, an MIT license, a README, and two files I'd recommend to anyone building a linter, `EXAMPLES.md` and `BAD_EXAMPLES.md`. They show exactly what each rule accepts and rejects, and they became the tool's living specification.

## Real documents fight back

Testing a linter on your own carefully crafted `test.md` is one thing. Running it against real documentation is humbling.

Most of my commits in August and September weren't new rules. They were exceptions:

*   **Template variables.** Our docs use placeholders like `{~ var.product_name ~}` that get substituted at build time. My link checkers were dutifully reporting every one of them as broken.
*   **Special files.** `SUMMARY.md` uses its own list style on purpose, and `README.md` files follow different conventions. Both needed to be skipped by certain rules.
*   **Text next to images.** A short caption within a few spaces of an image isn't a missing blank line, it's a layout choice.
*   **Subjective rules.** On 15 September I removed a diagnostic entirely. It flagged indentation that was *unusual* but not *wrong*, and writers were rightly ignoring it.

That last one taught me the most. A linter that cries wolf trains people to ignore it. Every false positive costs you trust, and trust is the whole product.

## Someone else's pull request

On 8 October 2025, a teammate opened a pull request that improved how internal links were validated. It was a small change, one file, but it meant a lot to me. Until then this was my side project. After that it was *our* tool. Someone had cared enough to read the code, find the rough edges, and fix them. It was also quiet proof that AI-written code could hold up to a colleague's review.

## Thinking at the level of the whole site

By early 2026 the per-page rules were stable, and I started noticing a different class of problem, one that only shows up when you look *across* pages:

*   A page exists in the folder but isn't listed in `SUMMARY.md`, so no reader can navigate to it.
*   `SUMMARY.md` says "Getting Started" but the page's title says "Introduction".
*   A link says "Setup Instructions" but lands on a heading called "Installation Steps".

In March 2026 I added rules for all three. This was a shift in thinking: the linter stopped looking at a page in isolation and started understanding the docs site as a connected structure.

I also built, and on the same day deleted, a rule that checked for dead links in `SUMMARY.md`. The existing internal link rule already caught those. Two warnings for the same problem is noise, so it went.

## What vibe coding actually looks like for a writer

People sometimes assume "the AI wrote it" means I pressed a button and walked away. It didn't work like that. My process for every rule looked like this:

1.  Write a few examples of correct and incorrect markdown, exactly as I would for a style guide.
2.  Describe the rule to the AI in plain language, with those examples.
3.  Open my test files and see what gets flagged.
4.  When it flagged something it shouldn't, or missed something it should, explain why and ask for a fix.

That fourth step was most of the work, and it's where being a writer helped. I didn't know how a regular expression works, but I knew exactly what a correct page in our style looks like. The AI knew the code and I knew the domain. Neither of us could have built this alone.

In early 2026 I moved to Cursor, and I did something that felt very natural for a writer: I documented the project for the AI. I wrote down its conventions in rule files the AI reads automatically: one file per rule, always skip fenced code blocks, every message ends with `[amdocs-linter]`, every rule gets a test. I also wrote reusable "skills" for the two things I did most often, scaffolding a new lint rule and packaging the extension. After that, adding a rule went from a long back-and-forth to describing the behaviour and reviewing the result.

It turns out that clear, structured instructions are exactly what AI tools need, and writing them is what technical writers do for a living.

## Not waiting for the dev team

I want to be clear about what changed for me, because I think it applies to a lot of writers.

Before AI coding tools, every tool I wanted came with a dependency on someone else. I would file a request, explain the problem to a developer who didn't feel the pain, and wait behind product work that would always matter more. Most of those tools never got built.

With this extension, I was never blocked. When a writer reported a false positive on Monday, the fix could be in their editor the same week. I didn't need a sprint slot or a developer to translate my requirements. I was the person with the requirements, and now I could act on them.

That doesn't make developers less important. My teammate's pull request improved the tool, and I'd welcome more. But the gap between "I wish we had a tool that..." and actually having it is now much smaller than most technical writers think.

## Shipping it to people who aren't me

A tool only helps if people are actually running the latest version. For months, updating meant asking colleagues to download a `.vsix` file and install it by hand, and most of them didn't.

So in May 2026 I built auto-update. Once a day the extension quietly checks for a newer version. If it finds one, it shows the changelog and an **Update Now** button that downloads and installs the update for you. If anything goes wrong, such as no network or no VPN, it fails silently. Linting is never affected.

Versions 1.1.0, 1.2.0 and 1.3.0 all went out on 21 May. They added auto-update, empty-page detection, duplicate-entry detection in `SUMMARY.md`, and a setting to skip link checks for internal domains that sit behind single sign-on. Four days later, version 1.3.1 fixed a network edge case in the updater itself, which is exactly the kind of fix that auto-update made easy to roll out.

## What I'd tell anyone starting a tool like this

1.  **You don't need to know how to code.** If you can describe the problem precisely, with examples, an AI tool can write the code.
2.  **Start on a Friday night with one rule.** You don't need a design document to find out whether an idea works.
3.  **Keep a file of bad examples.** It's your test suite, your spec and your demo all at once.
4.  **Delete rules without regret.** A redundant or subjective rule makes every other rule less trustworthy.
5.  **Write down your conventions.** Future you, your teammates and your AI assistant all benefit.
6.  **Solve distribution early.** The best rule in the world doesn't help someone running last year's version.

It started with three commits called `push`. It's still growing, and I'm still writing commit messages I'm not proud of.
