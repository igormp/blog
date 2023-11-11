---
layout: post
title: "Hijacking Steam's lancache for faster CDN downloads"
date: 2023-11-11 14:50:00 -0300
bg: no
comments: true
---

As a long-time Steam user with a speedy 1Gbit internet connection, I recently decided to buy a new game and play it, as people often do. Clicked 'Install', got a pop-up for the 50GB download, big but apparently reasonable by today's standards. However, to my surprise, the download speed hovered around just 7MB/s — approximately 160Mbps, or merely 16% of my total bandwidth. This was significantly lower than what my actual internet connection can actually handle.

| <img src="{{ site.baseurl }}/images/steam-hijack/steam_download_rec.jpg"> |
| :-----------------------------------------------------------------------: |
| _Initial download speed at 7MB/s - That's awfully slow for a 1Gbit link_  |

The first thing I tried was switching download servers, which did net me a minor speed up, going from 7MB/s to almost 18MB/s, but still way less than what I was expecting.

|    <img src="{{ site.baseurl }}/images/steam-hijack/steam_download_sp.jpg">    |
| :----------------------------------------------------------------------------: |
| _After Server Switch: Speeds up to 18MB/s. That's better, but still not ideal_ |

After some googling online I ended up seeing that this is a somewhat common issue in some places, and [this reddit post](https://www.reddit.com/r/InternetBrasil/comments/urc1hv/para_quem_tiver_problemas_com_a_velocidade_de/) did some nice explaning on why it's somewhat slow here in Brazil, and it's basically a matter of Steam only using bad CDNs here, and their local servers not being that great. Another thing is that apparently Steam only makes use of 8 simultaneous connections to those download servers, whereas when you're using a lancache you can have up to 32 connections, which would teorically net you a 4x speed up.

For those that are not aware of what lancache is, it's basically a way to download files from a local computer instead of the internet. So if device A of yours already has a game, another device B in your local network can download it straight from A instead of having to download it from the internet. This is especially useful for people that have multiple devices in their network, or even for LAN parties, where you can have a single device download the game and then share it with everyone else, at much faster speeds than what your internet connection and Steam's servers can usually handle.

[This reply](https://www.reddit.com/r/InternetBrasil/comments/urc1hv/para_quem_tiver_problemas_com_a_velocidade_de/i8xphyr/) of theirs in special describes how they managed to trick Steam into thinking that it's downloading from a lancache, but instead redirects those calls to a specific CDN, which is what I'll be doing here.

Another source that gave me the certainty that this is viable was [this link](https://opensourcelan.com/blog/2016/07/01/steam-cdn/). It gives a nice explanation on how steam works with their CDN providers. They even have some JS code to hijack Steam's CDNs, but I'm not really a JS fan, so I totally ignored their implementation and decided to roll my own because why not?

## Their original idea

I'm going to leave a copy of the original post here for posterity, but I'll also give a quick explanation of what they did (in English):
<detais>

<summary>Original post (in Portuguese)</summary>
Sim, por padrão o pihole instala o lighttpd (da pra fazer nos outros servers eu acho mas n sei específicos).

No painel web em Local DNS você precisa apontar `lancache.steamcontent.com` pro ip do pihole (IPv4 e v6).

Depois por SSH adicionar essas linhas em `/etc/lighttpd/external.conf`

```
url.redirect = (
  "^/depot/(.*)" => "http://steampipe.akamaized.net/depot/$1"
)
server.max-keep-alive-requests=3000
$HTTP["url"] =~ "^/depot" { accesslog.filename = ""}
```

As opções de CDN são:

Akamai: `steampipe.akamaized.net` ou `akamai.cdn.steampipe.steamcontent.com`
Google: `google.cdn.steampipe.steamcontent.com` (tbm tem google2)
Level3/Lumen: `level3.cdn.steampipe.steamcontent.com`
Highwinds/Stackpath: `f3b7q2p3.ssl.hwcdn.net`
Depois só reiniciar o lighttpd ou o pihole inteiro e testar. Recomendo trocar a região do mesmo jeito pois o cliente pode abrir mais um monte de conexões se a lista de servidores tiver mais do que um.

EDIT: Ainda tô testando essa última linha de desligar o log.

</details>

Basically what they did was to redirect any call to `lancache.steamcontent.com` by overriding their local DNS to resolve the Steam lancache request to their PiHole's IP, and then they added some lines to their lighttpd config to redirect any request to `lancache.steamcontent.com` to a specific CDN.

## My first try

At first I thought "Hey, I do have a router running Linux and custom firmware, so I can just do the same thing, right?". So I wrote a really simple Go program that spun up a webserver, and whenever it received a request for a `/depot/` path, it would redirect it to the CDN that I wanted to use. I decided to go with Go since my idea was to run this on my router (an ARM based device), and I didn't want to bother with complicated cross-compiling (Go makes it really easy) or having to install a runtime/interpreter on it.

Sidenote: since it has been quite a while since I last did anything in Go, I was not aware that `go get` was deprecated in favor of `go install`, or that the latter was even a thing to begin with.

The first iteration of the code looked like so:

```go
package main

import (
	"net/http"
)

func main() {
	http.Handle("/depot/", http.StripPrefix("/depot/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		http.Redirect(w, r, "http://steampipe.akamaized.net/depot/"+r.URL.Path[1:], http.StatusFound)
	})))
	// Listen on port 8080
	http.ListenAndServe(":8080", nil)
}
```

Pretty simple, compile it with `GOOS=linux GOARCH=arm64 go build main.go`, `scp` it into my router and it's off to the races!

Now that the URL rewrite part is done, time to work on the DNS part. While the original idea made use of PiHole, and I do have a router running OpenWRT, I'm actually using a different software stack. After some more googling around I tried to add both a DNS override in the router's DNS config, and also a port forward for the service I was writting in parallel.

Of course those two ideas made no sense to begin with, since a) the DNS override would only work for the router itself and Steam would not detect a lancache available, and b) the port forward would only work for external requests, but that's what a tired mind at 2AM can do to you. Why do I even need to run this in my router to begin with? I can just run it in my PC and point my DNS to it, right? At first I thought that wasn't the case, since people mentioned that overriding your hosts file for the lancache with `127.0.0.1` would not work, but what about my own regular IP instead?

At this point, I had spent over an hour working on this and the download was still going on. I could have just waited for it to finish, but decided that I wanted my own solution to work, so I just stopped the download and spent way more time trying to solve the "issue" than I would have if I had just waited for it to finish. Oh, to be a programmer...

## Going simpler

So, instead of all that mess trying to run stuff in my router, messing with DNS settings, firewalls and whatnot, I simply added a new line into my `/etc/hosts` file containing the redirect to my PC's IP (`192.168.1.5 lancache.steamcontent.com`), spun up the service on port 80 in my PC, restarted Steam, and BAM! I do see requests coming in! But... why is the download so slow at Kb/s?

| <img src="{{ site.baseurl }}/images/steam-hijack/broken_downloads.jpg"> |
| :---------------------------------------------------------------------: |
|                                _Uhhhhhh_                                |

Opening the links in those logs only gave me 404s, what gives? Let's take a look into our actual rewrite code:

```go
http.Handle("/depot/", http.StripPrefix("/depot/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		http.Redirect(w, r, "http://steampipe.akamaized.net/depot/"+r.URL.Path[1:], http.StatusFound)
	})))
```

Let's zoom in a little bit:

```go
"http://steampipe.akamaized.net/depot/"+r.URL.Path[1:]
```

```go
r.URL.Path[1:]
```

Oh, why the heck am I removing the first character of the path? Anyhow, let's fix it and see if it works now:

| <img src="{{ site.baseurl }}/images/steam-hijack/working_500mbps.jpg"> |
| :--------------------------------------------------------------------: |
|                   _Finally managing faster speeds!_                    |

Yay! Finally managed to reach 50MB/s! That's only half of my actual available bandwidth, but I'll settle with that. Only took me two hours to speed up the download for the remaining 6GB of the game at faster speeds instead of the 10 minutes that it would have taken me to download it at full speed, but hey, where would be the fun in that?

The final code for the service can be found [here](https://github.com/igormp/steam-lancache-hijack). I plan on maybe adding some more features to it, like being able to specify which CDN to use (currently it randomly choses one for each request), automatically overwrite the hosts file, maybe even add a GUI, and make it available for Windows users, but no promises on that.
