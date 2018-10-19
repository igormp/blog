---
layout: post
title: "Intel's UXA, SNA and weird screenshots"
date: 2018-10-19 11:00:00 -0300
bg: no
comments: true
---

For a while I've faced a weird problem with my screenshots where they would glitch, creating a weird image containing part of my desktop background with a black area on top instead of the actual window that I wanted to capture, as seen below.

| ![How the print should look like](/blog/images/regular_print.jpg "How the print should look like") | 
|:--:| 
| *How it should look* |

| ![The glitched print](/blog/images/glitch_print.jpg "The glitched print") | 
|:--:| 
| *The weird output* |

I thought that was maybe a bug with [maim](https://github.com/naelstrof/maim), the tool that I used to do screenshots along with [slop](https://github.com/naelstrof/slop), which made me turn to [shotgun](https://github.com/Streetwalrus/shotgun) to no avail.

Turns out it was due to the [acceleration method](https://jlk.fjfi.cvut.cz/arch/manpages/man/intel.4) used by my Intel driver, which defaults to SNA and was the culprit of this weird bug. By simply changing it to UXA actually solved my problem with no noticeable drawbacks.

To do so, I created a file in `/etc/X11/xorg.conf.d/` named `20-intel.conf` with the following content: 

```
Section "Device"
	Identifier  "Intel Graphics"
	Driver      "intel"
	Option      "AccelMethod"  "uxa"
EndSection
```