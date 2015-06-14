---
layout: post
title: "Building a Simple IoT Light Switch, Pt. 2"
date: 2015-06-14 15:13:07 -0400
comments: true
categories: hardware
---

This is a pretty late post, but I wanted to show off the hardware I designed
as a follow up to the first
["Building a Simple IoT Light Switch"](/blog/2015/01/18/a-simple-iot-light-switch/).
In that post I presented a prototype of a smart light switch I designed that
consisted of a button which triggered a small Python server running on a
Raspberry Pi to control my Phillips Hue lights over an HTTP API. It was an
extraordinarily simple design that worked well enough, how it was very
inconvenient to transport since I had to rewire the breadboard to the raspi
GPIO pins every time I moved. Since it seemed production ready
for my needs, I decided to solve this problem by using this as an opportunity
to learn PCB design and actually design a production board.

Special thanks go to [Nick Kubasti](http://nickkubasti.com/) for teaching me
how to use KiCad and to [John Sullivan](http://thejohnsullivan.com/) for helping
me with the final lab work.

The basic circuit I designed is below. It is merely a GPIO pin hooked up to
ground with a switch in between, and an LED and resistor for fun.

![](/images/iot2/switch1.png)

The code behind this configures the GPIO pin to use a pull up resistor, which
pulls the pin's voltage up to VCC when the button is not pressed, and down to
GND when it is. This lets the code poll for when the pin's value is `False`.
A more complete schematic including the pull up resistor is below. Note that
I won't have to implement the pull up part in my design because the Raspberry
Pi implements that internally.

![](/images/iot2/switch2.png)

My initial idea was to have a little board that would plug straight down into
the Pi's GPIO female headers with a little button and LED at the end. However,
that was really wasteful since that would block all of the pin even though
I was only using three of them (input, GND, VCC). Nick suggested a board design
that had two sets of headers: one set of male ones for plugging downwards as I
had originally thought, and one set of female ones facing upward that all the
unused pins would be forwarded to. This way, even though the downward facing
male pins are all plugged in, they can all still be accessed since the board
traces simply connect the unused male pins to the corresponding female ones.
The exception to this is GPIO pin 11, which is the one actually being used for
the switch. I just need to remember for future projects that pin 11 is
reserved. That was probably really hard to understand, so here's the schematic,
created with KiCad.

![](/images/iot2/schematic.png)

On the left you can see the two sets of 13 x 2 headers. The ones on the left
will be the male ones that plug downward into the Pi's female headers. The
ones on the right will be the female ones that wires can be plugged into.
All those wires going around the top and bottom are the forwarded connections.
The one pin exclusively in use on the left is pin 11, which is connected to
the circuit. All the rest are unused, and wires are used to connect them to
their corresponding female header. Pins 1 and 9 are technically in use, but
since they correspond to VCC and GND, it's fine to forward them too.

The actual circuit is in the bottom left. It's very similar to the above
circuit diagrams, pin 11 is connected to GND (pin 9) via a switch, and
3.3V VCC (pin 1) is connected to an LED, resistor, and the same GND (pin 9).

The next step after creating the schematic, and assigning actual parts to each
of the components, was to design the printed circuit board (PCB). This involves
laying out the components as they will appear on the physical
silicon wafer.

![](/images/iot2/pcb.png)

This was a completely new area to me and it was challenging and fun to
actually draw out the board's traces, taking care not to intersect them,
and using the different layers when necessary. You might notice that there's
a seemingly random set of 1 x 2 headers in the middle of the area where the
switch goes. I added those because I noticed that the button and LED were going
to actually extend off the end of the Pi. I was concerned about how pressing
down on the button would actually flip the Pi a little bit, so Nick had an
incredible idea and suggested adding space for a pair of dud headers that
could be used as legs to support the end of the PCB that hung off the side
of the Pi.

After this, I was then able to pretty easily use KiCad to generate some
Gerber files to send to the fab. I chose to use
[OshPark](https://oshpark.com/) based on Nick's recommendation, and they turned
out to be a great option. I was able to get three copies of my board, with
free shipping for <$8!

After waiting a couple of weeks for the boards to come in, John helped me
solder everything together to finish up this project. During this process
we noticed one design mistake: the layout of the switch was actually *way*
too big for the switch we had on hand, but that didn't turn out to be a serious
issue.

![](/images/iot2/mistake.jpg)

There are some pictures of the final product:

![](/images/iot2/final1.jpg)
![](/images/iot2/final2.jpg)
![](/images/iot2/final3.jpg)
![](/images/iot2/final4.jpg)

Here's a video of it in action:

<blockquote class="instagram-media" data-instgrm-version="4" style=" background:#FFF; border:0; border-radius:3px; box-shadow:0 0 1px 0 rgba(0,0,0,0.5),0 1px 10px 0 rgba(0,0,0,0.15); margin: 1px; max-width:658px; padding:0; width:99.375%; width:-webkit-calc(100% - 2px); width:calc(100% - 2px);"><div style="padding:8px;"> <div style=" background:#F8F8F8; line-height:0; margin-top:40px; padding:50% 0; text-align:center; width:100%;"> <div style=" background:url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACwAAAAsCAMAAAApWqozAAAAGFBMVEUiIiI9PT0eHh4gIB4hIBkcHBwcHBwcHBydr+JQAAAACHRSTlMABA4YHyQsM5jtaMwAAADfSURBVDjL7ZVBEgMhCAQBAf//42xcNbpAqakcM0ftUmFAAIBE81IqBJdS3lS6zs3bIpB9WED3YYXFPmHRfT8sgyrCP1x8uEUxLMzNWElFOYCV6mHWWwMzdPEKHlhLw7NWJqkHc4uIZphavDzA2JPzUDsBZziNae2S6owH8xPmX8G7zzgKEOPUoYHvGz1TBCxMkd3kwNVbU0gKHkx+iZILf77IofhrY1nYFnB/lQPb79drWOyJVa/DAvg9B/rLB4cC+Nqgdz/TvBbBnr6GBReqn/nRmDgaQEej7WhonozjF+Y2I/fZou/qAAAAAElFTkSuQmCC); display:block; height:44px; margin:0 auto -44px; position:relative; top:-22px; width:44px;"></div></div><p style=" color:#c9c8cd; font-family:Arial,sans-serif; font-size:14px; line-height:17px; margin-bottom:0; margin-top:8px; overflow:hidden; padding:8px 0 7px; text-align:center; text-overflow:ellipsis; white-space:nowrap;"><a href="https://instagram.com/p/37ZQ8_n86u/" style=" color:#c9c8cd; font-family:Arial,sans-serif; font-size:14px; font-style:normal; font-weight:normal; line-height:17px; text-decoration:none;" target="_top">A video posted by Mark Mossberg (@mssbrg)</a> on <time style=" font-family:Arial,sans-serif; font-size:14px; line-height:17px;" datetime="2015-06-14T23:29:39+00:00">Jun 14, 2015 at 4:29pm PDT</time></p></div></blockquote>
<script async defer src="//platform.instagram.com/en_US/embeds.js"></script>

Thanks for reading! I have two extra boards I'm not doing anything with, so
if you happen to have a Raspberry Pi and some Hue lights, I'd be happy to give
you the hardware and software for your own DIY light switch.
