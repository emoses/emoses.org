+++
title = 'Migrating My Home Server'
date = 2024-02-19
categories = ['blog']
tags = ['selfhosted', 'ops', 'linux', 'docker']
draft = true
+++

# Migrating My Home Server

{{< tldr >}}
* I migrated services on my homeserver to a new machine
* Moving dockerized services was very straightfoward
* Using DNS and FQDNs everywhere made migration seamless
* Caddy is cool too.
{{< /tldr >}}

## My old setup

I have a couple little boxes in a downstairs room that run a lot of my self-hosted applications[^1].  For a long time it
consisted of a Synology NAS and a Raspberry Pi 4B.  The RPi was enough to run [Home
Assistant](https://home-assistant.io), the Unifi Controller for my Unifi APs, and a few other services for many years,
but a little while ago I decided to run some surveillance cameras locally and I added [Frigate](https://frigate.video)
to the mix. That is beyond the capabilties of my little Pi, so I dug out an old 2012 Intel Mac Mini I had sitting
around.  I upgraded its RAM for $20, installed Debian, bought a [Coral TPU](https://coral.ai/products/accelerator), and
set it to work.

[^1]: A few others run in the cloud on a [Vultr](https://www.vultr.com/) VPS

## Not-very-new hotness

This worked pretty well, and I started adding a few services to it, but sometimes it would just...stop working. It would
drop from the network (which makes it pretty useless as an NVR), I'd get an alert[^2], and I'd have to run downstairs to
hard-reset it. ![A still from the IT Crowd captioned "Have you tried turning it off and back on
again?"](/img/hello-it.jpeg). I spent a little time attempting to diaganose it by googling various log messages but
eventually decided that for a couple hundred dollars I could get a much more recent SFF PC that would have more
headroom, expandibility, and maybe even out-of-band management.

{{< aside >}}
#### A solution?

As it turns out, I may have diagnosed and fixed the problem after I had pulled the trigger on the new server.  Some
suspicious log lines in the `syslog` mentioned iommu, and Googling a little harder led me to [this Reddit
post](https://www.reddit.com/r/linux_on_mac/comments/w3hisc/network_dropout_fix_for_linux_on_mac_with_kernel/).  I added

```
GRUB_CMDLINE_LINUX="iommu.passthrough=1"
```

to `/etc/default/grub` (and ran `update-grub2`), and I didn't see the problem for a couple weeks, which means...not much
really, as it only happened at most every few days anyway.
{{< /aside >}}

I hopped on eBay and picked up a Dell Optiplex 7070 with a i5-9600T and 16GB RAM for under $250.  I spent a bunch of
time trying to figure out what I should get but at the end of the day I just kinda aimed around $200 and saw what was
there on eBay.

## Time to move stuff

My home server is much more of a workhorse than a playground.  It runs the house, and I run it, and while it's fun to
have a project now and then I really just want to set it up once and have it chugging merrily away in the basement.  I
don't need a desktop, I don't need anything cutting-edge; Debian is solid, staid, stable, I know how to use `apt`, and
it's my choice for server OS.  So I wiped Windows 11 off the server, threw on Debian stable (Debian 12 Bookworm as of
this writing), and got to work.

I don't run much on the bare OS.
