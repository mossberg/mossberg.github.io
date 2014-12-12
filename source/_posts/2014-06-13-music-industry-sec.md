---
layout: post
date: 2014-06-13
title: Infosec, Meet the Music Industry
Slug: music_industry_sec
categories: security music
published: false
---

*Thoughts on infosec and the music industry*

For those that aren't familiar, I'm a huge fan of music, both listening to
and writing/recording it. I haven't kept up as much recently, but with my
additional free time during the summer I've been getting back into it and I had
a realization that intersected with my current interests in information security.

Frequently, recording musicians have very delicate recording setups that are
easily broken with software updates for OSes, DAWs
(Digital Audio Workstations), and plugins used inside DAWs. As a result, many
musicians are very wary of updates, because they can't afford to risk breaking
the tools their livelihood depends on, when they currently have a working setup
that does the job just fine. Go on any
[popular](http://www.gearslutz.com/board/)
[recording](http://www.soundonsound.com/forum)
[forum](http://forum.cockos.com/forumdisplay.php?f=20) and you'll find
numerous threads about troubleshooting software due to upgrades and the like.
Because of this, I wouldn't be surprised to find numerous artists
producing on computers running OS X Snow Leopard (10.6), despite that the
current version of OS X is Mavericks (10.9.3).

This reality presents quite a lush attack surface with potentially lots of aging,
vulnerable software sitting, just waiting to be exploited. Attackers could
potentially be able to steal and leak tracks before they've officially been
released or even fully produced.  A leak of raw, pre-processed, vocal
recordings could be very embarrassing for artists that rely heavily on pitch
correction software, not to mention could result in bad publicity which can
negatively affect record sales.

As far as I know, this isn't very much, if any, dialogue between the music
industry and the security community, however I think that at the very least,
there should be some basic education in the music industry about the
importance of using up to date software, especially in regards to security.
The deeper problem however is why is it so common for software updates to break
recording software, even in minor ways? One reason I can think of is the difficulty to conduct testing in a situation where nearly every user will be using a unique recording setup with different OSes, DAWs, plugins, and even recording hardware. I imagine it would be quite easy to introduce changes that break things, given the complexity going on behind the scenes.

It's probably true that the average producer probably doesn't have to worry about getting hacked, but something like this might be more relevant for the big time producers that work on Billboard Top 100 hits and the like, given the impact of some of the possibilities I mentioned earlier (leaks, mainly).[^1]

As always, thanks for reading!

---

[^1]: Although at that level of professionalism, I'd assume they'd make it a priority to have an up to date recording studio environment.
