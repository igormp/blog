---
layout: post
title: "Using ADB to quickly simulate touch events"
date: 2018-02-23 00:00:00 -0300
bg: no
comments: true
---
*__Notice__: this requires that you have root access in your phone*


A while I go I started playing one of those non-stoppable clicker games on my android phone, in which all you have to do is tap on the screen to acquire resources and make your tapping even more profitable. Seeing how boring it is to keep tapping your screen the whole time, I wondered if there was some way do to that automatically without requiring me to install a 3rd party app into my phone, and the first thing that came to my mind was [ADB](https://developer.android.com/studio/command-line/adb.html). Here I'll show you how I did it.

## Connecting to your phone

First of all, you'll need to [download](https://developer.android.com/studio/releases/platform-tools.html) the binaries required to use ADB with your phone. Also, don't forget to actually enable ADB on your phone. A step-by-step guide can be found [here](https://www.howtogeek.com/125769/how-to-install-and-use-abd-the-android-debug-bridge-utility/).

One thing that I did was to enable ADB over the network, so as to free myself from having to have an USB cable plugged the whole time. Connecting to my phone wirelessly was easy as running `adb connect PHONE_IP_ADDRESS:5555` in the terminal (remember to replace PHONE_IP_ADDRESS with your phone's actual IP address or hostname).

Run `adb shell` and you'll be greeted with... yes, you've guessed it! Your phone's shell! Now acquire root privileges with `su` and you're good to go.

## Actually sending touch events

After making the best use of my Google-fu, I came to this neat [Stack Overflow question](https://stackoverflow.com/questions/3437686/how-to-use-adb-to-send-touch-events-to-device-using-sendevent-command). Now all I had to do was to find the coordinates on the screen where I wanted the touch event to happen; this can be done by running `getevent -l`, which will output something like:

```bash
add device 1: /dev/input/event6
  name:     "uinput-fpc"
add device 2: /dev/input/event5
  name:     "msm8920-sku7-snd-card Button Jack"
add device 3: /dev/input/event4
  name:     "msm8920-sku7-snd-card Headset Jack"
add device 4: /dev/input/event2
  name:     "gf3208"
add device 5: /dev/input/event0
  name:     "qpnp_pon"
could not get driver version for /dev/input/mice, Not a typewriter
add device 6: /dev/input/event3
  name:     "gpio-keys"
add device 7: /dev/input/event1
  name:     "ft5x06_720p"
```

Then I proceeded to tap on my phone's screen where I wanted the simulated taps to occur, and that's what I got:

```bash
/dev/input/event1: EV_ABS       ABS_MT_TRACKING_ID   00006943            
/dev/input/event1: EV_ABS       ABS_MT_POSITION_X    00000166            
/dev/input/event1: EV_ABS       ABS_MT_POSITION_Y    0000031f            
/dev/input/event1: EV_KEY       BTN_TOUCH            DOWN                
/dev/input/event1: EV_SYN       SYN_REPORT           00000000            
/dev/input/event1: EV_ABS       ABS_MT_TRACKING_ID   ffffffff            
/dev/input/event1: EV_KEY       BTN_TOUCH            UP                  
/dev/input/event1: EV_SYN       SYN_REPORT           00000000  
```

Don't forget to press Ctrl+C to stop it from listening to your touches.  
Take notice of the values taken from `ABS_MT_POSITION_X` and `ABS_MT_POSITION_Y`, those are the coordinates for your X and Y axis in hexadecimal. 0x168 and 0x320 in my case, which translate to 360 and 800 in decimal.

Now that you have the coordinates, you can simply run `input tap X Y`, replacing X and Y with the decimal values that you got in the last step. 

You should've noticed that it only simulates a single touch on your screen, to get around it I've made the following simple one liner:

```bash
while : ; do input tap 360 800; done
```

Don't forget to replace the coordinates with your own ones!

And hey, it works! __But it's too frecking slow__.

## Speeding it up

You should be wondering why it's slow, and this happens because `input tap` is a java application, and that delay is actually the time required to spawn then run it (at least according to the answer that I took from [here](https://stackoverflow.com/questions/34443112/faster-android-input-tap-command)).

So, how do we get around it?  
After a little bit more of googling around, I've come around [this](https://stackoverflow.com/a/12884604/4542015). Writing straight to the touch interface, why didn't I think of it before?  
And then I  blindly copy pasted the solution presented in that link and, to my surprise, it simply didn't work at all.

After scratching my head for a while, I noticed something: remember the output that you got to determine tap coordinates? Take a closer look at it, right in the beginning we have something along the lines of `/dev/input/event1`, and that was my actual touch input device location.

Since the current working directory that we're running at is probably the root of you device, which is read-only, we need to navigate to a read-write folder before continuing.
After getting into a read-write folder, you should be able to run the following (replacing `eventX` with your actual input device`:

```bash
dd if=/dev/input/eventX of=record1
```

Proceed to tap or swipe whatever you want there, it'll record everything and save it to a file. Press Ctrl+C after you're finished.

To test you recording, run:

```bash
dd if=./record1 of=/dev/input/eventX
```

If everything worked smoothly, you should be able to run it in an endless loop with:

```bash
while : ; do dd if=./record1 of=/dev/input/eventX ; done  
```

And that's it! Now you should be able to cheat in those clicker games or even automatize some tests and events on your phone.