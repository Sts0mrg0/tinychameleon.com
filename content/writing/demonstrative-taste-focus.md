---
title: "Demonstrative Taste: Focus"
date: "2020-01-18T21:08:14Z"
tags: ["programming", "python"]
---

I don't like the phrase "Code Smells" or the concepts behind it. It's a nebulous
phrase which changes definition by programming language, and is generally used
as a catch-all for subjective opinions about code structure. I believe taste is
a better concept for legibility and maintenance, because for written work it has
deep roots in linguistics. When using taste we don't need to identify what is
"best", we're looking for what is bad. For example: it is easy for a native
English language speaker to point out bad writing; people do this all the time
to non-native English speakers on the Internet.

There are many lower-level concepts we discuss in code review, like data clumps
or refactoring patterns, but all of these processes are directed at providing
tasteful results. Yet most of these concepts are overly concerned with code
structure; I believe they should be subordinate to the guiding concept of taste.

I would like to discuss one tasteful concept: focus. The idea of focus comes
from linguistics, and can be readily applied to programs; in fact, checking
cyclomatic complexity is an attempt to quantify it.

> *focus (plural: foci)*
>
> Linguistics --- an element of a sentence that is given prominence by intonational or other means.

While "Code Smells" are hazy, focus is explicit by relying exclusively on the
form of your code. Most importantly, I believe all developers are capable of
pointing to prominent sections within blocks of code. Focus has value because
prominence is easy to see and is language agnostic. This ease allows for quicker
agreements on prominent pieces within the scope of analyzed code, and preempts
subjective discussions in code review.

The examples I've chosen below come from real-world code written by junior
developers where I have provided the solutions we will discuss. These are short
by design to avoid overwhelming the discussion with large bodies of code, but
I believe, and hope, they adequately represent the described concepts.


## Nesting is for Birds

Let's start out with something that should be familiar to most developers:
eliminating duplication. Everyone likes to reduce code duplication, right?
Except sometimes it doesn't feel quite right once the code is cleaned up. I
think every developer will experience this feeling at some point, no matter the
programming language; points of focus are the reason behind this feeling.

When you create an abstraction it's easy to increase the number of prominent
pieces of program code within a given scope. Maybe you have additional required
parameters for a higher-order function now, or a brand new interface, or a more
complex weaving of sum types, but it can also manifest in simpler ways too.
Simple ways like this:

```
if princess_kidnapped:
    hero = find(mushroom_kingdom, 'Mario')
    hire(hero)
else:
    hero = find(mushroom_kingdom, 'Toad')
    hire(hero)
```

This kind of nesting and duplication is easy to spot. If we try to point out
prominent pieces of this code it should be clear each branch of the if-statement
is prominent. All if-statements produce this prominence because each branch
encodes a different procedure; this also applies equally to pattern matching
syntax.

What do we do once we've found the prominent pieces of code? Maybe nothing. What
you do should always take context into account to avoid that feeling of a
refactor gone wrong. In this case though, we're going to apply one of the
subordinate refactorings we know. Let's give it a shot:

```
name = 'Mario' if princess_kidnapped else 'Toad'
hero = find(mushroom_kingdom, name)
hire(hero)
```

We've removed the if-statement entirely and rely on an if-expression to isolate
the changing piece of information inside a new variable named `name`. It's clear
we've reduced the number of prominent pieces within this scope; it feels better
because reduced duplication and nesting is a side-effect of fewer foci. Every
level of nesting introduced creates a focus, but it is also unavoidable, so
use caution when introducing more of it.


## Three-Index Programming

There are times when, due to legacy decisions, and perhaps comically low ROI, you are stuck with code
that is horrid to read. In these situations, generally, there is little you can
do to fix the root problem. A refactor is out of the question due to the
negligible resultant value and implementing a secondary, better way of achieving
the same result for new code only compounds inconsistency over time.

Most of the time I see people hide the ugly details behind a pretty, local
interface. There just happens to be a lovely linguistics-turned-medical term
that covers this scenario:

> *focalize*
> 
> Medicine --- confine (a disease or infection) to a particular site in the body.

This is a good strategy because focalization of bad code also reduces prominence
within the scope using the new interface. You get to ignore the ugly code and
deal with less foci: a two-for-one deal! Many times this manifests in code
reviews like this:

```
KINGDOM_HERO = 'Luigi'

def save_princess(world):
    hero = find_hero(world)[0][0][0]
    if not hero:
        hero = KINGDOM_HERO
    guide(hero)
```

There are many foci in this small snippet of code:

- A `KINGDOM_HERO` module-level constant used in one location;
- A nesting level introduced by the if-statement assigning a default value;
- A triple indexer statement to obtain a value from some kind of lookup.

In this case, the triple-index expression cannot be removed, as much as I would
love it. We only have one option for improving this code: hide the code behind
a pretty interface:

```
def get_hero(world):
    hero = find_hero(world)[0][0][0]
    return hero or 'Luigi'

def save_princess(world):
    hero = get_hero(world)
    guide(hero)
```

We've reduced module-level foci by encapsulating the constant within the lone
function responsible for finding us a hero, and have reduced prominence within
the main `save_princess()` function. All related data and responsibilities are
cohesive; cohesion reduces foci, fewer foci provide less prominence, and less
prominence means greater legibility.


## Snug Idioms

Focusing on foci and focalization when reading code only gets you so far: you're
going to need to understand and apply programming language idioms. Taste will
help apply idioms in ways that avoid creating future maintenance nightmares for
yourself and others. Let's look at an example that combines two of the things
we've seen: duplication and if-statement nesting. This example loop was repeated
several times within a module:

```
for hero in heroes:
    for item in hero['inventory']:
        if item['name'] != 'Fire Flower':
            continue
        toss(item)
```

The cognitive load caused by foci these nested loops create is pernicious; our
cause isn't helped much by the if-statement guarded `continue` either. Newly
minted developers tend to take a first cut at solving this problem by entirely
eliminating the repetition, including loops:

```
def useless_items(heroes):
    yield from (
        item
        for hero in heroes
        for item in hero['inventory'] if item['name'] != 'Fire Flower'
    )

for item in useless_items(heroes):
    toss(item)
```

The original, repeated loop constructs are definitely focalized, but I wouldn't
consider this attempt successful. We've added a third loop, a generator, and a
piece of coupling: operating on `item` instances is now tied to the iteration
logic contained in `useless_items()`. This cross-cutting logic is the stuff
that piles up, making code hard to change in the future. Let's do a second pass
and roll back some of the abstraction:

```
def useless_items(inventory):
    yield from (item for item in inventory if item['name'] != 'Fire Flower')

for hero in heroes:
    for item in useless_items(hero['inventory']):
        toss(item)
```

If we look at prominence within this second example, the generator stands out,
but the nested loop has dropped a focus and gained legibility. What we've done
in this example is spread the total foci more evenly across scopes; in the real
codebase this loop construct was repeated and we achieved fewer foci. Most
importantly we've expressed ourselves similarly to Python by creating a function
like `enumerate()`. You can reduce prominence within scopes by following idioms;
the code will snap tightly together in ways your teammates expect.


## Thinking at Cruising Altitude

I believe the linguistics concepts of focus and focalization are a fantastic
model for thinking about written code. Instead of working at a lower-level
populated by mechanical transformations, thinking about prominence gives you a
wide-angle view of your code, which will help guide precision improvements to
scopes and serve as a warning of abstractions going wrong.

Next time you're reading linter output, or cyclomatic complexity warnings, think
about prominent pieces of your code, think about focus and focalization. This
type of analysis has been a wonderful tool for me over the years, and hopefully
it can help you in the future too.
