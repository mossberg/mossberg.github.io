---
layout: post
date: 2013-10-11 22:50
title: From noise machine, to dedicated torrent box
published: false
---

*My adventures with a $5 server from the MIT Flea Market*

Not too long ago, I went to the MIT Flea Market which is an awesome flea market for all things hacker/maker. You can find all sorts of cool stuff ranging from tools to electronics to computers. I ended up finding a super old 2003 [Apple Xserve](http://en.wikipedia.org/wiki/Xserve) server that this one table was selling for only $5, so of course I had to buy it, despite having zero experience with working with an actual server before. 

In the time I've had it, I've done some of the basics, such as set up a basic web server which currently hosts a webpage for my apartment, but to be honest, this isn't useful. Lately I've been trying to think of other ways to make the server something more than a noisy box in our living room.

Something I thought of that would at least provide some small bit of utility was to use the server as a dedicated torrent box that anyone in the apartment could use to download and seed torrents. This offers a lot of advantages over just using your own computer, especially for a college student who only has a laptop which has to be carried around frequently. Instead of leaving my computer on all night, why not just give that work to the server, who isn't doing anything anyway?

Unfortunately the options weren't too great for torrent clients because the server is so old; it's running Mac 10.4, OS X Tiger! [^1] However, I was able to install a legacy version of [Transmission](http://transmissionbt.com) on it which works just fine. The key feature that I needed in order to make this work was this ability for the client to watch a certain directory for new torrent files and automatically add them to the queue. From that point, there were a couple of different ways that I thought of going about this. 

1. I could add a page to the website that would let a user upload a torrent file, which would then be put into the directory that the torrent client was watching. This would be very straightforward to implement.

2. I could take advantage of Dropbox. I could create a shared folder that anyone in the apartment could add torrent files to.[^2] From there, I could simply point Transmission to watch that shared directory and I'd be done!

With either of these methods, it would be nice for everyone to be able to watch the status of these torrents, so without even checking if my legacy version of Transmission offered a remote web interface (such as what uTorrent offers) I began looking for ways to hack this together myself. I found a couple of Python Transmission API's that I might have been able to use, but while digging through the Transmission preferences, I found a web interface option that made my life a lot easier.

Anyway, I went with option #2 above, just because it seemed easier and faster. I set up the shared folder in no time at all, but then I ran into a problem when I tried to test this out. My version of Transmission doesn't have an option to move the actual downloaded data file once it's completed downloading, which means that it would just get put into the same directory as the torrent files. That being a Dropbox directory, that wouldn't work because that (potentially huge) file would be synced to *everyone's* Dropbox accounts, taking up space, and letting everyone see what you downloaded.

The solution that I decided on using was changing the directory that Transmission was watching and then using cron and a python script to move all the torrent files from the shared dropbox directory into the actual watched directory. This is better because it eliminates any sort of Dropbox folder clog and offers slightly more privacy because after a certain (relatively short) time period, the file is moved out of Dropbox and no longer available for everyone to look at.

This setup works pretty well, the Dropbox sync works nearly instantly and I have cron setup to run the python script every 15 minutes, because I doubt anyone would need to torrent something so urgently that they couldn't wait, at most, 15 minutes for it to start. Once the script is run, the files are moved, Transmission notices, and gets to work. Anyone can check on the status of their download[^3] through the web interface, running on port 9091 of the web server.

At this point, the only thing left was to set up a way for people to actually get their files back from the server. It took some digging around in the settings of the server admin panel, but I was able to set up a public network share that allows anyone on the network to access the directory containing the data files.[^4]

And that’s it! It might have been a simple project, but this was my first experience with working with a server and it was pretty gratifying to set up something actually useful for my apartment.

---

[^1]: I don't really want to bother with updating the OS right now, I'd rather just make do with what I have.

[^2]: And delete, I suppose. But that would be a dick move.

[^3]: And all others, defeating the point of any sort of privacy mentioned earlier.

[^4]: Further eliminating any notion of privacy.
