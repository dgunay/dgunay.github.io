---
layout: post
title: "Why you shouldn't specify someone's experimental fork as a dependency"
date: 2022-05-13 23:08:02 -0700
categories: git
---

I know, I know, it sounds stupid. But anyway, here's how I messed up and cost
some of my coworkers a couple hours.

## Why would I even do this

At work we use a code generator (mockery) to generate mocks of our interfaces
for testing.

Recently Go came out with 1.18, which adds support for generics. In my zeal for
the usefulness of generics, I started using it a lot in our codebase. And it did
cut down on a lot of repetitive code!

However, I eventually found out that mockery didn't support generics yet. This
would normally be fine because our interfaces aren't generic, but that wasn't
the problem. The presence of _any_ code calling generic functions in the
scanning path of mockery would cause it to crash. This lay latent for quite a
while because at first, I was only using generics at the very lowest layers of
our codebase, where the mocks were not being generated from.

I figured that the quickest way to get around this would be to see if there was
someone already working on adding support, and it turned out there was. It even
worked, in fact. Their PR was not yet merged, but it looked like it would be
merged "soon" and a new release cut for it. So I decided I would use their fork
to unblock us.

## Where I messed up

I chose possibly the worst way of using the forked code. I referenced a specific
commit in the downstream PR in our go.mod file.

Normally this would not be so bad. Thousands of projects rely on pulling code
over the internet during builds. The problem is that I did not think about the
practical difference between a stable, maintained release and someone's actively
developed branch from the standpoint of Git and how that interacts with build
tools.

Because the build tool has to be able to find the code, a tag or a commit hash
is usually the way it happens. Tags are fine and what are typically used to
reference stable releases. Commits though, can easily be overwritten. It's
something I typically avoid even on branches in progress, but other developers
can and do often force push or rebase their branches, destroying commits in the
process.

So naturally, the contributor force pushed to his branch and destroyed the
commit I was referencing. We could no longer generate mocks, which meant that we
could no longer change our interfaces or our tests would break and be unfixable.
Work ground to a halt until my team forked the project and got it working again.
It did not help that I made this change weeks ago and it worked fine until I was
in the middle of Disneyland on my day off, but hey I guess that's just Murphy's
Law.

## What I should have done

If something like that is important and you absolutely do not want it to change,
don't be an idiot like me. Do something to ensure it will always be available.
You can copy/fork the project, or even just check in binaries for all the
architectures you'll need. Whatever you do, just make sure that the code is
fully under your control, and don't just assume that it will stay online
forever.

Honestly, this is a best practice even for ostensibly stable dependencies. There
have been rather high profile instances of open source projects intentionally
breaking their releases, be it due to [opinions about release policy][uws],
[sabotage/activism][hacktivism], or to [get revenge against patent
lawyers][leftpad].

[uws]: https://alexhultman.medium.com/beware-of-tin-foil-hattery-f738b620468c
[hacktivism]: https://news.yahoo.com/developer-sabotaged-own-open-source-185931413.html
[leftpad]: https://qz.com/646467/how-one-programmer-broke-the-internet-by-deleting-a-tiny-piece-of-code/

The most ironclad solution is known as "vendoring", or making a copy of your 3rd
party dependencies. This ensures that if you have access to your code, you have
access to everything you need to build the project. It can be copied into the
repo or you can maintain a mirror of it and direct your build tools to pull from
there.

It is a bit more work to keep your dependencies up to date, but I have honestly
run into productivity-sapping problems several times now at different companies
stemming from internet-dependent builds. All it takes is for one momentary
outage to cost you 5+ minutes of build time. If it turns out to be a longer
outage, you can be blocked from deploying a new release or critical bugfix.
At a large enough scale, this will cost you much more money than it would to
just maintain mirror repositories for your dependencies or check them in.
