---
title: "Mikrotik: A new adventure in the networking world"
date: 2024-05-19T14:20:00-07:00
---

Recently, my HP ProCurve switch decided to die. It was one of those situations
where it didn't make sense to try to repair it. However, this caused my entire
home network to go down, so I had to plug the things that I could directly into
my Linux router/lab box instead. Now normally, I'd be able to get another
ProCurve from my father, as he used to have access to pallets of them. However,
this is no longer the case, so I had to figure out what I wanted to do instead.

I ended up deciding on trying out Mikrotik. My reasons came down to the
following:

* I've heard some good things, so it shouldn't be too bad
* The price was substantially lower than an equal-ish ProCurve switch online[^1]
* It has SFP ports, so it should be more "open" on what it accepts
* Firmware updates are simple and easy to get to

Those were the main points, especially the penultimate one. Being unable to use
"third-party transceivers" in my ProCurve was absolutely frustrating. Despite
the fact there is command to "allow" them, this never worked. And, even if it
did work, my switch was too old to even do 10G.

As for the last one, I should point out, HP has effectively made it impossible
to actually download firmware for your switches.[^2]

### Racking and Trying It Out

So I got the switch, great! I racked it up and immediately tried to start
working with it. There was one thing that caught me off guard: The CLI is
completely different from what I'm used to on ProCurve and Cisco. Literally
*everything* starts with a `/`. Thankfully `<TAB>` works! I like when I can use
that on a command line. Also thankfully, Neil (a good friend of mine on my team
at the Rocky Linux project) gave me a good cheat sheet!

#### Management Interfaces

I should point out that I much prefer a CLI interface whenever possible.
Learning this new CLI was weird. I first asked "why can't they just use `conf`
and `show` like normal managed switches?" Regardless, I got used to how this
worked and being able to tab efficiently made all the difference.

I learned WinBox, their dedicated management program for Windows, happens
to able to connect to it via MAC address, in case I didn't want to use the
serial port. But I didn't try it out until I needed some visuals when the
console port was getting frustrating at the beginning. I will say that WinBox
offers the ability to open a terminal. I think that's a neat feature.

I learned a bit later that it comes with a web interface called "WebFig", but
that requires a routable/reachable IP in the network. The default 192.168.88.1
did not fit this mold for my network, so WinBox it is.

#### Weird Concepts

Coming from an HP ProCurve, a lot of things in Mikrotik's RouterOS already made
sense. In between all the things that made sense, there were some things that
*didn't* make sense, at least at first. The primary one was bridges. I ended up
understanding the "why", at least for my model of switch.

I noticed too that since RouterOS is the default choice, that means my switch
was capable of basically being a router all by itself. This feature I had no
plans of using; I essentially just wanted a managed switch, since I already have
routing taken care of elsewhere.

#### Documentation

There is a *lot* of documentation out there for Mikrotik. Some may be outdated,
some may still work with what's currently out there. Whatever it may be, I found
very quickly that the documentation lacked some specific explanations but
overall, helped lead to the "right" thing to do. They even provide explanations
of some networking concepts. Granted, not a deep dive, but it may be enough for
those who already "understand" some of those things.

I'll give it an "A" for having documentation to begin with. I'll also give it an
"A" for having links to other pages with more information on a particular
concept. The only difficulty I had was understanding some of the concepts, not
finding answers to questions in the documentation, or seeing "real world"
examples there. I believe this was mostly a "me" problem, and not a
documentation problem. There were a couple forum posts that provided answers to
questions that came to mind.

### Configuration

It's at this point that I was configuring my switch. I basically needed all my
VLAN's, dedicated ports to particular VLAN's, LACP where needed, and "trunked"
ports to begin with. What I found fairly quickly was how weird it was to set it
all up and have it *work*.

To summarize what I did, I had to:

* Leave the starting bridge alone (it's fine as it is)
* Create VLAN's on the bridge for each VLAN I have, and set appropriate "tagged"
  and "untagged" interfaces
* Create VLAN "interfaces" and assign them to the bridge
* Assign an IP address to my management VLAN
* Enable VLAN Filtering on the bridge

#### Commands

This post wasn't supposed to be a guide. However, something told me I was going
to end up with comments or emails about how I got my configuration to work. It
seemed only appropriate to provide what I did to set my switch up, because I
know there will be others over time who are looking for the "right" information
or a "correct" way of setting up a CRS3XX switch.

The first thought that came to mind: I need to create an LACP (until I get SFP
card and cables). So I needed to choose four (4) ethernet ports.

```
# Remove ether{1..4} from the bridge
[admin@npsw00n1] > /interface/bridge/port remove numbers=0,1,2,3

# Add the bonding interface
[admin@npsw00n1] > /interface/bonding add lacp-rate=1sec mode=802.3ad name=bond0 \
\... slaves=ether1,ether2,ether3,ether4 transmit-hash-policy=layer-2-and-3

# Add the bonding interface into the bridge
[admin@npsw00n1] > /interface/bridge/port add comment=lacp0 interface=bonding1 bridge=bridge
```

With that out of the way, now I just have to get all of my VLANs in there,
making sure that the bridge is tagged for every VLAN.[^4]

```
[admin@npsw00n1] > /interface/bridge/vlan
# Management VLAN. I want one interface "untagged" (aka management port) and
# then the LACP take care of it.
[admin@npsw00n1] /interface/bridge/vlan> add bridge=bridge comment="Management" \
\... tagged=bond0,bridge untagged=ether24 vlan-ids=1000

# The rest of the VLAN's I just added and I would get to their dedicated ports
# later. Only a few I show here.
[admin@npsw00n1] /interface/bridge/vlan> add bridge=bridge comment="Isolated" \
\... tagged=bond0,bridge vlan-ids=1001
[admin@npsw00n1] /interface/bridge/vlan> add bridge=bridge comment="Builder" \
\... tagged=bond0,bridge vlan-ids=1005
[admin@npsw00n1] /interface/bridge/vlan> add bridge=bridge comment="Nebula" \
\... tagged=bond0,bridge vlan-ids=2000
```

Now I just needed to add VLAN interfaces to the bridge.

```
[admin@npsw00n1] /interface/bridge/vlan> /interface/vlan
[admin@npsw00n1] /interface/vlan> add interface=bridge name="Management" \
\... vlan-id=1000
[admin@npsw00n1] /interface/vlan> add interface=bridge name="Isolated" \
\... vlan-id=1001
[admin@npsw00n1] /interface/vlan> add interface=bridge name="Builder" \
\... vlan-id=1005
[admin@npsw00n1] /interface/vlan> add interface=bridge name="Nebula" \
\... vlan-id=2000
```

There's a default IP, so I need to drop that in favor of an IP relevant to my
management subnet while attaching that to the right VLAN.

```
[admin@npsw00n1] > /ip/settings/set ip-forward=no
[admin@npsw00n1] > /ip/address
[admin@npsw00n1] /ip/address> remove numbers=0
[admin@npsw00n1] /ip/address> add address=10.100.0.3/24 interface="Management" \
\... network=10.100.0.0
```

I'll set my DNS and NTP settings too, while I'm here.

```
[admin@npsw00n1] > /ip/dns/set servers=10.100.0.1
[admin@npsw00n1] > /system/clock/set time-zone-name=America/Phoenix
[admin@npsw00n1] > /system/ntp/client/set enabled=yes
[admin@npsw00n1] > /system/ntp/client/servers add address=2.rocky.pool.ntp.org
```

Now that everything is mostly setup, time to turn on VLAN filtering on the
bridge. This disconnects briefly if connected via MAC address or IP, but login
should just work again.

```
[admin@npsw00n1] > /interface/bridge/set vlan-filtering=yes numbers=0
```

I'll do a quick ping test just to make sure.

```
[admin@npsw00n1] /ip/address> /ping 10.100.0.1 count=4
  SEQ HOST                                     SIZE TTL TIME       STATUS        
    0 10.100.0.1                                 56  64 396us     
    1 10.100.0.1                                 56  64 362us     
    2 10.100.0.1                                 56  64 469us     
    3 10.100.0.1                                 56  64 428us     
    sent=4 received=4 packet-loss=0% min-rtt=362us avg-rtt=413us max-rtt=469us 
```

In the end, that actually wasn't too bad.

### A Broken Console Port

This isn't a complaint about the switch. This is something I noticed and I
believe I may have been the cause of this. At a certain point, the RJ45 console
port on my switch decided to stop working. However, I only noticed it stopped
working after I powered off everything in my rack to rearrange and recable.
everything.

When I noticed the serial wasn't working but WinBox was still fine, I figured
something was up with the port, my cable, or my system that was connecting to
it. I wasn't sure why it stopped working. So I rebooted the switch, figuring
RouterBOOT would show up. This wasn't the case.[^3]

I believe I may have damaged the port on accident. I was working in the office
and I remember something tugging hard and fast on the cable plugged into the
RJ45 serial. I also remember being able to still type on the console via
minicom. It just has never come back and worked. I did notice that sometimes
while trying to get output, random characters will come back. So there is
life... but I believe it was me that damaged it. I just hope that it's easy to
repair if I ever decide to open it up and look.

### Final Thoughts

### Footnotes

[^1] When I mean substantially lower, I mean about 70% lower.

[^2] I would love to be proven wrong about this. They don't seem to allow you to
have a public-domain email address (e.g. gmail.com).

[^3] I should note that I did do a complete factory reset and even updated the
firmware of RouterBOOT.

[^4] You really should do this. You need to allow the bridge to manage your
VLAN's in some way. This goes hand-in-hand with VLAN filtering and hardware
offloading.
