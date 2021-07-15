---
title: "Into The Crust"
date: "2021-07-21T22:40:32.169Z"
template: "post"
draft: true
slug: "forked-1"
category: "NeoFM"
tags:
  - "Code Dive"
  - "Boring"
  - "fm"
description: "A small insight into the code of hit game for weird perverts Fetish Master, or How I Fixed Performance With One Line"
socialImage: "/media/cover.png"
---
Official theme of this post:
<object
    data="https://www.youtube.com/watch?v=L5KVMnS9zgY">
</object>

So a few years ago, the developer of hit weird sex pervert game Fetish Master released the entire source.

And then seems to have disappeared, leaving it's development status forever in limbo. Shame.

Good thing it was released under a license that allows further development then. Ha.

I have a bit of a hobby of digging in to stanky code of open source games, and this one has some real stank. _Real_
stank.

# Disclaimer

I'm going to be pretty harsh on the code quality here, but I just want to be clear up front that this stuff happens. I
don't have any ill will towards and this isn't a personal attack on H.Coder, it's just a comedic look at some crusty
code.

Anyways, let's get going

# ...oddities

I did a bunch of work to get the build system updated and working in a way I wanted but tooling is insanely boring so I'm
not going to detail that here.

There's definitely a lot of strange parts but something immediately obvious was this one.

![what](/media/fmfork-1/error.png)

Attempting to compile with modern java versions yielded this baffling error. I thought it might be a zero width space
or something, and after some testing I realized the issue was the C. Or, rather that it wasn't a C at all but a С.

Cyrillic Capital Es that is.

Yeah, I'm at a loss as to how that even happened. Guessing some kind of strange locale crap given H.Coder is Japanese.

There's a lot of just really strange tiny details like this. Misspelled functions is a pretty amusing and common one.

By far the most obviously rank part though is the
# Comments

Oh jesus the comments.

<video autoplay loop muted playsinline>
  <source src="/media/fmfork-1/comments.mp4" type="video/mp4">
</video>

Comments in code are lines that are ignored when actually run or compiled. They're intended to be used for what their
name would imply, comments on how code works or why it's designed the way it is. Some people are very religious about
that everything needs to be thoroughly commented but I'm more in favor of comments are largely unnecessary and should
mostly just be for high level documentation and explanations of why possibly confusing decisions were made.

Another common usage for comments is to very quickly disable sections of code. If you turn a line into a comment, it
doesn't run, so it's a convenient way of temporarily disabling a small section, maybe for testing purposes or so you
can work on another section and then fix the commented section once that part is done. This isn't really the best way
of doing things but it's convenient and easy.

However, it's fairly universally considered bad practice to "leave in" commented code so to say. If you're actually
publishing or commiting to a repository or anything similar it just makes code more difficult to follow while not
actually doing anything.

Everything grey in that clip is a comment, and it's _all_ commented code. Yeah.

# Optimizing

There is one upside, however, is that when the code for the game clearly wasn't source controlled for very
long the comments almost act as a record of thoughts or work in progress things. For example, let's look at the
initialization for the scripting engine:

```java
        OptimizerFactory.setDefaultOptimizer("reflective");
        
        sysVars = (HashMap) fileXML.LoadXML(gameDataPath+"/texts/system.xml");
```

Hm. That doesn't really tell us much. That loose string literal is giving bad vibes but this only really tells us what
is happening and not why.

But wait! I actually removed a large section of commented code!

```java
//        if (System.getProperty("java.version").startsWith("1.6.") || System.getProperty("java.version").startsWith("1.7."))
//        {
//            Debug.print("Normal MVEL optimizer used");
//            OptimizerFactory.setDefaultOptimizer("ASM");
//        }
//        else
//        {
//            System.out.println("Java version is questionable, Backup MVEL optimizer used for maximum compatibility");
//            OptimizerFactory.setDefaultOptimizer("reflective");
//        }
        OptimizerFactory.setDefaultOptimizer("reflective");
//        OptimizerFactory.setDefaultOptimizer("ASM");
        
        sysVars = (HashMap) fileXML.LoadXML(gameDataPath+"/texts/system.xml");
```

Aha, now it makes more sense. Sort of. Previously, or perhaps it was planned to, it checked the java version and then picked
either the "ASM" or the "reflective" optimizer from MVEL. However, it doesn't do that now. It just always uses the
"reflective" optimizer.

Let's go consult the docs for MVEL and figure out what this means...

![ok](/media/fmfork-1/mvel_docs.png)

Oh great MVEL doesn't actually have library documentation, just the language guide. And the actual code isn't documented either.

Well let's just trial and error see what happens, I don't feel like digging through it's source right now.

So let's uncomment that ASM optimizer and...

![dangerous](/media/fmfork-1/fm_error.png)

The log floods with MVEL compiler errors as soon as you click start game, I see why it was disabled. According to that
log, something is broken with MVEL itself. Library bug, a classic. Well the easiest way to deal with a library bug is
to find a patched version. The version of MVEL used by Fetish master is `2.2.2.Final`

I assumed from this weird version name this was some end of life final release version of MVEL, but it turns out it's
still somewhat actively maintained. The most recent release is `2.4.12.Final`. Let's just plunk that in and...

![dangerous](/media/fmfork-1/fm_running.png)

Jackprot!

It runs with no visible errors, and runs way smoother and with no noticable lag no matter the complexity.

Boom, lag fixed.

# Whu Happun?

Wait a minute, what's going on here anyways? Why did I go around poking with a specific commented line in a fairly
large application? What is that actually doing?

So I, among others, noticed that a big slowdown seemed to be associated with the "scripts" in the game. Whenever the
game needed to do a not particularly large amount of not particularly complicated processing, it would grind to a halt
and take seconds to finish. Making the UI asynchronous so it doesn't hang every time this happens would be huge, but
also be fairly complicated. And on top of that, it doesn't really fix the issue it just makes the UI more responsive.
It would be better if there was some kind of easy win that could be made in terms of making the slow operations just
go faster.

Seeing the actual operations in question immediately made me suspect. I've worked with some very slow languages and
what's actually happening to cause the lag shouldn't really be a major processing issue in even interpreted languages
like MVEL.

Assuming that MVEL is just slow is almost a non starter and ends where I started so that's going to be a final
conclusion if other methods fail, so let's look at what I checked first.

### The scripts are being loaded incorrectly

Something is malformed about how the scripts are loaded and called. Instead of being cached in RAM they're reread from
the disk each time, they're called more times than they should, something like this.

Really cursory examination showed this wasn't the case. They're read from the disk, cached, and then on top of that
actually get "compiled" by MVEL into an AST as well, and this is done properly too.

This one's a bust.
### Some kind of overly expensive operation latches on to the rest of the processing

This didn't seem to be the case either. Nothing I could find seemed to be doing anything that could spiral in complexity
or runtime. On top of this, the performance is always a gradual creeping decay. An indicator of the whole thing running
slow rather than something specific suddenly running over.

### Some kind of performance feature is disabled.

This was my last resort, and had promise. Thinking maybe the scripts were being fully interpreted each time, I put in a
simple Google search, "MVEL JIT."

![google result of optimizer link](/media/fmfork-1/mvel_optimizer.png)

BINGO!

Second result was a code link to a file in the MVEL repo containing a comment about support for an ASM JIT compiler.
And it seems like there's two optimizers supported, a "reflective" optimizer and an "ASM" optimizer.

Searching for the two static strings in the fetish master codebase yielded no results, but the raw "ASM" lead me to the
commented block in question. And then I uncommented it, saw the error, and so on.

So the issue was due to a bug in MVEL, JIT had been disabled and so a major performance feature was gone.

## Just In Time

Generally programming languages can fall into three categories, interpreted, compiled, or ASM.

An interpreted language is read each time at run time and then executed like that. You can think of the computer as
actually going line by line and evaluating each line of code. Sometimes there's more complicated of transformations but
generally the source is the same as the actual program you run.

A compiled language is more of an intermediate language. It's fed into a "compiler" which turns the code into another type of code which can be more directly run. Normally this second code is an ASM language.

An ASM language is as low level as it gets. Short for assembler or assembly, they're as low level as it gets in the relevant environment. In some cases this can be it is literally what your actual processor is executing, and there's no lower level way to talk to it. While they typically are extremely terse, difficult to work with, and have a steep learning curve, they are fundamentally executed similar to an interpreted language. The computer goes line by line and runs each instruction.

# Let's GOOOOOOOOO

I'll be publishing a modified version of Fetish Master in the next few days that includes this fix. If you want to
incorporate this into your own branch or fork, feel free to. I wouldn't be able to stop you because of the license
anyways, but just making it explicit.

I might do (shorter) snippets into ongoing refactors and code horrors of Fetish Master if people find this interesting.
Who knows.
