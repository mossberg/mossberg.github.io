---
layout: post
title: "Building A Simple IoT Light Switch"
date: 2015-01-18 13:32:48 -0500
comments: true
categories: programming python hardware
---

I'm lucky enough to own a [Philips Hue](http://www.meethue.com) wireless
lighting unit (thanks [NUACM](http://acm.ccs.neu.edu)!) which essentially is
this really awesome Internet of Things (IoT) product that lets me replace
all my standard light bulbs with special RGB ones that can be controlled
wirelessly. The bulbs communicate via
[ZigBee](http://en.wikipedia.org/wiki/ZigBee) with a "Bridge" unit that
is connected to my local network and hosts an HTTP API for interfacing
with the lights. This API is used by the official Hue mobile app for controlling
the lights, but is also publicly documented and totally hacker friendly.
The lights are awesome, but it is a bit of a drag to have to use an app
to turn them all off rather than having some physical switch [^1], so I decided
to fully embrace the IoT trend and use my Raspberry Pi to build a simple
HTTP-fluent light switch for turning my lights on and off.

<!-- more -->

### Hardware

The circuitry itself is literally as simple as it gets for this kind of thing,
all I have is GPIO pin 11 on the board (BCM pin 17) connected to pin 6 (GND),
with a switch in between. In my code, I'll configure pin 17 to use an internal
pull up resistor which will bring the voltage up to 3.3V when the button is not
pressed and down to 0V when it is.

![](/images/rpi.jpg)

### Software

The Raspberry Pi Python library makes it really easy to control circuits. For
simplicity, my code uses a polling approach to detect when the button is pressed
but the RPi library also supports real callbacks using threading.

```python
def main():
    while True:
        inp = gpio.input(PIN)
        # pull up resistor will cause inp to be True when button is not
        # pressed and False when button is pressed
        if not inp:
            callback()
        time.sleep(BUTTON_SLEEP)
```

My `callback()` function consists of code that uses the Hue API to request
a diagnostic of the lights, which comes back as a JSON blob with each light
represented as an object. If no lights are on, it turns them on, otherwise
turning them all off. Turning the lights on and off is as simple as submitting
a PUT request to the API endpoint for each light with a JSON blob specifying
the state to turn to (On=True, Off=False).

```python
def callback():
    survey = requests.get(BASE + '/lights')
    survey = json.loads(survey.text)
    numlights = len(survey)
    # if any are on, turn all off.
    if any([survey[str(x)]['state']['on'] for x in range(1, numlights+1)]):
        for light in survey.keys():
            turn(light, False)
        print '[{}] Off'.format(datetime.datetime.now())
    else:  # else if all are off, then all on
        for light in survey.keys():
            turn(light, True)
        print '[{}] On'.format(datetime.datetime.now())


def turn(light, state):
    data = json.dumps({'on':state})
    requests.put(BASE + '/lights/{}/state'.format(light), data=data)
```

That's it! I leave the script running in a tmux pane on the Pi and I can
hit the button at any point to toggle the lights on and off. The full code
is available on [github](https://github.com/mossberg/raspi/blob/master/hue.py).

For future work,
it would be cool to integrate RF chips so the button wouldn't have to be
physically attached to the Pi and I could have a little remote control. I'll
leave that for another day. Thanks for reading, here it is in action!

<blockquote class="instagram-media" data-instgrm-version="4" style=" background:#FFF; border:0; border-radius:3px; box-shadow:0 0 1px 0 rgba(0,0,0,0.5),0 1px 10px 0 rgba(0,0,0,0.15); margin: 1px; max-width:658px; padding:0; width:99.375%; width:-webkit-calc(100% - 2px); width:calc(100% - 2px);"><div style="padding:8px;"> <div style=" background:#F8F8F8; line-height:0; margin-top:40px; padding:50% 0; text-align:center; width:100%;"> <div style=" background:url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACwAAAAsCAMAAAApWqozAAAAGFBMVEUiIiI9PT0eHh4gIB4hIBkcHBwcHBwcHBydr+JQAAAACHRSTlMABA4YHyQsM5jtaMwAAADfSURBVDjL7ZVBEgMhCAQBAf//42xcNbpAqakcM0ftUmFAAIBE81IqBJdS3lS6zs3bIpB9WED3YYXFPmHRfT8sgyrCP1x8uEUxLMzNWElFOYCV6mHWWwMzdPEKHlhLw7NWJqkHc4uIZphavDzA2JPzUDsBZziNae2S6owH8xPmX8G7zzgKEOPUoYHvGz1TBCxMkd3kwNVbU0gKHkx+iZILf77IofhrY1nYFnB/lQPb79drWOyJVa/DAvg9B/rLB4cC+Nqgdz/TvBbBnr6GBReqn/nRmDgaQEej7WhonozjF+Y2I/fZou/qAAAAAElFTkSuQmCC); display:block; height:44px; margin:0 auto -44px; position:relative; top:-22px; width:44px;"></div></div><p style=" color:#c9c8cd; font-family:Arial,sans-serif; font-size:14px; line-height:17px; margin-bottom:0; margin-top:8px; overflow:hidden; padding:8px 0 7px; text-align:center; text-overflow:ellipsis; white-space:nowrap;"><a href="https://instagram.com/p/yAhhmMn8yt/" style=" color:#c9c8cd; font-family:Arial,sans-serif; font-size:14px; font-style:normal; font-weight:normal; line-height:17px; text-decoration:none;" target="_top">A video posted by Mark Mossberg (@mssbrg)</a> on <time style=" font-family:Arial,sans-serif; font-size:14px; line-height:17px;" datetime="2015-01-18T20:09:40+00:00">Jan 18, 2015 at 12:09pm PST</time></p></div></blockquote>
<script async defer src="//platform.instagram.com/en_US/embeds.js"></script>

[^1]: Of course I could physically go to each light and flip the switch but manually turning off all three lights in my room is even more work than launching the app.
