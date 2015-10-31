---
layout: post
title: "Bombing a Coding Interview: A Post-Mortem"
date: 2015-09-06 09:36:40 -0400
comments: true
categories: programming interview
published: false
---

Recently, I bombed a whiteboard coding interview question harder than
ever before. It was really bad. I had a bunch of miscellaneous thoughts
scattered all around the white board, but I *barely* had any code to show
when time elapsed, and the code I had wasn't even close to a solution. The
worst part was that in hindsight, it wasn't even a particularly
hard problem. In this
post I want to dump some thoughts on the experience so I can learn from it.

Before that, some meta thoughts. It can be hard to reflect on and document
failures like this, because it often requires accepting uncomfortable or
inconvenient truths about yourself -- in my case, that my algorithmic thinking
skills are much more poorly developed than other areas of my knowledge.
Conversely, it's much easier to make excuses and absolve yourself of
responsibility. The main excuses I felt myself thinking afterwards were:

- "This interview was towards the end of a long day of other challenging
  interviews. I didn't do well because I was burnt out."
- "My interviewers said very little the whole time. I didn't do well because
  they should have given me more hints/helped me work through it more/put me on
  the right track."

The first sentences of those were true, but the second were definitely
subconsciously made up to help me comfort myself. While there perhaps may
have been bits of truth in those second sentences, it's not healthy
to lean on them at crutches. Ultimately, they prevent me from
facing two realities.

1. I was solely responsible for my "less than desirable" performance.
2. Because of this, the only thing stopping me from improving is myself!

Ok, now let's look at the interview question.

I was essentially given a variant of the
[Skyline](https://uva.onlinejudge.org/index.php?option=com_onlinejudge&Itemid=8&category=3&page=show_problem&problem=41)
problem. This seems to be a fairly well known programming problem, but
I had never seen it before.  Paraphrasing, it was essentially:

> Given an unsorted array of box structures (defined below), write a function
> that prints out each vertex of the skyline created by the collective boxes.

```c
struct box {
    int width, height;
    int x; // x coordinate of bottom left corner of box
};
```

Referencing the image below, my function would need to print out the coordinates
of each of the "x"'s.

![](/images/skyline.png)

I didn't really know where to start, so I just began manually examining
the white board drawing of the boxes. I identified that my starting point would
be the `struct box` with the minimum `x` field and then started trying to
determine the set of conditions for detecting a "rise" or "drop". This became
complex and I was having a lot of troubling reasoning about how to deal
with all the possibilities related to boxes that were inside each other and how
certain configurations would affect the skyline, or not.

A big mistake here was spending too much time concerned with the details
related to the conditions of the rise/drop. I didn't even know how I was going
to process the unsorted array, but I just sorted of pushed that off for later.
I drafted up some measly code for locating the array element
with the smallest `x` field via iterative search, and ended up getting in the
mindset that somehow my function would need to fundamentally work by making
various passes along this array to locate elements of interest (or something
like that). That was a fundamentally flawed approach,
and unfortunately, I got really stuck going down this path and ran out
of time.

I remember trying to voice my thoughts to the interviewers, but
not really being able to say anything because all I had was vague, abstract
idea for a general approach, but nothing I could really put into words.
Since I wasn't saying much, it makes sense that they weren't really saying
much either.  Maybe it would've helped to even directly ask them if it
seemed like I was on the right track.

I also remember thinking at one point that there was *something* crazy about an
approach based on making passes over the input array. It just didn't seem
right. What I *should* have done was scrap the idea right there and taken a
step back to consider completely different strategies. It probably would have
helped if I had considered creating a secondary data structure to help
organize the box data in a more useful way. Brian Gordon wrote a great
[post](https://briangordon.github.io/2014/08/the-skyline-problem.html)
describing solutions to this problem, and started off with a naive solution
involving creating a heightmap that was populated by iterating over the boxes.
I'm pretty confident that if I was hinted at creating helper data structures, I
would've been able to produce at least a naive solution without much trouble,
but unfortunately that just never came to mind.

This isn't the first time I've struggled with an interview type problems that
involved creating extra structures beyond what's given, but this reflection is
still helpful in affirming that this type of problem is a focus
area for me. I also see now that directly asking the interviewers for a hint,
or even if I'm on the right track might not be a bad idea. Worst case, they
say nothing, and I'm right where I was before. Lastly, I can know going
forward that if I intuitively feel like there's something wrong with an
approach I'm exploring, it might be a good idea to completely drop it and
consider different approaches. Hopefully keeping these in mind will help
me improve for future interviews!
