+++
title = "Channels are not versions"
date = 2025-03-23
+++

# Channels are not versions

Have you ever asked someone "what version are you running?" and gotten an answer like "latest" or "beta"? Was that answer annoyingly unhelpful, but you couldn't quite explain why? Or were you the person giving that unhelpful answer? Either way, this post is for you!

## Software versions matter

Software tends to change over time; only very rarely is a piece of software ever really "done". Being easy to change is, in some sense, the entire _point_ of software. So in a lot of cases, we want to refer to not just a software project (e.g. Ubuntu), library (Log4j), or application (Firefox) but to a specific _version_ of that software. If you're trying to explain to your mum how to connect her laptop to a different WiFi network, it makes a big difference to the instructions you'll give if she's using Windows 11 or Windows XP.

## Precision matters

If you're a developer trying to understand a problem that someone trying to use your software has reported, you likely care _exactly_ what version they had the problem with. After all, the problem might only occur in some versions. It's even possible that you already fixed it without realizing, or reworked things such that it appears in a totally different way.

If you're responsible for a service that lots of people rely on, you should care about precisely what version every component of your system is running in your production environment. Not in the sense that you should know them all by heart - or have opinions on the relative merits of different versions the way wine snobs compare vintages of the same wine - but you do occasionally need to precisely describe, reason about, and modify what versions you're running. Without that precision it's impossible to make statements like "we fully tested this before deploying to production", or even "here's all the things that changed since the last time this was known to work".

## So what are we talking about here?

Versions are strings, that often look something like "2.0.541982.0" or "15.3.1", where the same string always refers to the same code. Git commit hashes are versions. Rust crate versions are versions (at least for crates that have been published on [crates.io](https://crates.io/), which disallows overwriting an existing version).

Channels are strings, that often look something like "alpha", "stable", or "latest", where the string refers to somewhere to get code. What code you actually get changes over time. Git branch names, RPM repositories, and docker image tags are all examples of channels.

Note that because what version you get from a channel changes over time, it's impossible to reason correctly about having tested a particular version if all you know is the channel the code you tested came from. It's also impossible to actually roll back a deployment that pulls software from a channel at deployment time, at least whenever the software vended via that channel changes between deployments - your rollback deployment will _also_ grab the latest (broken) version from that channel.

Building a deployment system which - in whole, or in part - fails to nail down a component's version in favor of just pulling from a channel isn't just sloppy or lazy. It's _dangerous_. It's leaving a landmine for your future colleagues (or future you). "Software Engineering" isn't a real profession with liability and such, but if it was I'd argue that this is a clear example of software engineering malpractice. Don't reference a channel when you really need a version.

## Special mentions

I can't write about version numbers without at least mentioning [TeX](https://en.wikipedia.org/wiki/TeX) (Donald Knuth's famous typesetting system, perhaps most commonly used as part of [LaTeX](https://www.latex-project.org/)), which has a version number of π. Or nearly π, anyway. Each release of TeX is given a version number that's a decimal approximation of π, with every subsequent release having one more digit (and so being a little bit closer to the actual value of π). Technically this works because π has an infinite number of digits in its decimal representation, but it's only practical because TeX is very rarely updated - as of this writing, the latest version is 3.141592653. Knuth has [expressed his intention](https://www.ntg.nl/maps/05/34.pdf) that upon his death, the version number of TeX be updated to π (`$\pi$` in TeX source code) and no further changes be made.

I should also say that channels aren't inherently bad. Since software changes all the time, "give me the latest and greatest" is a totally valid intention that our tools should allow us to express. It's just that when it comes time to deploying or releasing software, we need to be able to capture - and if necessary, restore - the exact version of everything involved. One approach to solving this problem is lockfiles, as used by [NPM](https://docs.npmjs.com/cli/v9/configuring-npm/package-lock-json) and Rust's [Cargo](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html). 