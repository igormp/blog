---
layout: post
title: "Toying with a PS3 in 2019"
date: 2019-07-21 05:00:00 -0300
bg: no
comments: false
---

I recently bought a PS3 for the sake of it. I didn't even plan to play games on it, however, I wanted to see how hackeable it actually is. Since I got a FAT model (a CECHG to be exact), it is possible for it to suffer from the infamous YLOD, so the first thing I did was to open it up and replace the thermal paste.


| <img src="{{ site.baseurl }}/images/ps3/guts.jpeg" width="250"> | <img src="{{ site.baseurl }}/images/ps3/rsx-cell.jpeg" width="450"> |
|:--: | :--:| 
| *Motherboard* | *RSX and Cell chips* |

Second thing on the TODO list was to actually install a CFW into it. For that, the [PS3 Homebrew wiki on Reddit](https://www.reddit.com/r/ps3homebrew/wiki/index) was super useful and contains all you need to get your system up and running. I even installed a couple games of dubious legality just to see if everything was ok.

Third thing on my list was to install linux, because why not? It was pretty painless, apart from some minor details that I will explain bellow, and the fact that the PS3 is actually too hecking slow for modern standards (maybe the cheapo flash drive that I used didn't help either).

First thing to take notice of is that, in the [OtherOS++ wiki page](https://www.reddit.com/r/ps3homebrew/wiki/otheros), even though I have a NAND console, I had to keep the file as `dtbImage.ps3.bin.minimal`, instead of removing the `.minimal` from it, otherwise Rebug Toolbox would complain about a missing file. Also, having an USB hub is a good idea if your model only has 2 USB ports (and one of those will probably be used by a flash drive).

| <img src="{{ site.baseurl }}/images/ps3/live.jpg" width="450"> | <img src="{{ site.baseurl }}/images/ps3/hub.jpg" width="450"> |
|:--: | :--:| 
| *Red Ribbon live boot* | *Hub with keyboard, mouse and flash drive* |

Another thing that I noticed is that, if for some reason you try to install Red Ribbon with certain locale and a language other than the default one, e.g., Brazilian locale with en_US as the language, the install would fail. So keep that in mind and just accept whichever language it offers for your chosen locale.


| <img src="{{ site.baseurl }}/images/ps3/installed.jpg"> | 
|:--:| 
| *Neofetch of the system through SSH* |

Since Red Ribbon is based on an old version of Debian, and no releases have been released since 2014, some updates are needed to keep everything in order, as seen on [its website](https://redribbongnulinux.000webhostapp.com/).

After applying the recommended fixes and updates, I tried to upgrade it with a regular `apt-get upgrade`. Oh boy, do I regret that. It took over 2 hours to finish unpacking everything. 

Something that I want to look into is that there seems to be no I²C modules available, which means that I can't read the temperature sensors, hence I'm not able to control the fan, which seems to keep spinning at max speed whilst being annoyingly loud.

Other things that I want to look into are about [unlocking the 8th SPE](https://www.psdevwiki.com/ps3/Unlocking_the_8th_SPE), [upgrading to systemd](https://wiki.debian.org/systemd#Installation), managing to program something that makes use of the SPEs, and getting a modern distro installed, although I'm not sure if any distro still supports PowerPC BE, nor if the PS3's CELL is able to run in LE.