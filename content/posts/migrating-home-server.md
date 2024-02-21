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
* Moving dockerized services was very straightforward
* Using DNS and FQDNs everywhere made migration seamless
* Caddy is cool too.
{{< /tldr >}}

## My old setup

I have a couple little boxes in a downstairs room that run a lot of my self-hosted applications[^1].  For a long time it
consisted of a Synology NAS and a Raspberry Pi 4B.  The RPi was enough to run [Home
Assistant](https://home-assistant.io), the Unifi Controller for my Unifi APs, and a few other services for many years,
but a little while ago I decided to run some surveillance cameras locally and I added [Frigate](https://frigate.video)
to the mix. That is beyond the capabilities of my little Pi, so I dug out an old 2012 Intel Mac Mini I had sitting
around.  I upgraded its RAM for $20, installed Debian, bought a [Coral TPU](https://coral.ai/products/accelerator), and
set it to work.

[^1]: A few others run in the cloud on a [Vultr](https://www.vultr.com/) VPS

## Not-very-new hotness

This worked pretty well, and I started adding a few services to it, but sometimes it would just...stop working. It would
drop from the network (which makes it pretty useless as an NVR), I'd get an alert[^2], and I'd have to run downstairs to
hard-reset it.
![A still from the IT Crowd captioned "Have you tried turning it off and back on
again?"](/img/hello-it.jpeg)
I spent a little time attempting to diagnose it by googling various log messages but
eventually decided that for a couple hundred dollars I could get a much more recent SFF PC that would have more
headroom, expandability, and maybe even out-of-band management.

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


## Server setup

My home server is much more of a workhorse than a playground.  It runs the house, and I run it, and while it's fun to
have a project now and then I really just want to set it up once and have it chugging merrily away in the basement.  I
don't need a desktop, I don't need anything cutting-edge; Debian is solid, staid, stable, I know how to use `apt`, and
it's my choice for server OS.  So I wiped Windows 11 off the server, threw on Debian stable (Debian 12 Bookworm as of
this writing) using a MicroSD-to-USB adapter as a boot drive, and got to work.

Not very much is run on the bare OS, but I had two major prerequisites: Docker, and [Caddy](https://caddyserver.com/)
for reverse-proxy.  For Docker I just used whatever you get when you [google "install docker
Debian"](https://docs.docker.com/engine/install/debian/#install-using-the-repository).  I installed the Caddy package
from `apt` before I remembered that Caddy is written in go, and if you want optional modules installed you need to build
it with those modules.  I needed the Cloudflare DNS module for acme certificate requests (more on that later), so I
added that modules in their cute little [web-based configurator](https://caddyserver.com/download) and put the resulting
binary at `/usr/bin/caddy`.  I left the systemd service that was installed by the `apt` package in place.

## Time to move stuff

With docker installed, I started moving services over, starting with the least critical services and working my way
"up".  Almost all my services have a FQDN at my home domain, which I'm gonna pretend is `home.emoses.example` (it's
not).  So [Grocy](https://grocy.info/) is at `grocy.home.emoses.example`.  The DNS for that domain is managed by
Cloudflare, and those entries are all CNAMEs pointing to my old server.  So now, I:

1. I took a look at the existing service's `docker-compose.yaml`, checking what mounts and ports were mapped in.  I
   generally setup my services so they just mount directories under the project directory.
2. I copied over the whole directory from the old server to the new one
    ```bash
    $ scp -r project/ emoses@newserver:~/docker/
    ```
3. I bring up the new service and test it on its locally exposed port (e.g. `https://newserver.local:8081`)
4. I add an entry to my Caddyfile for the new service
    ```yaml
    grocy.home.emoses.example {
        reverse_proxy localhost:8081
    }
    ```
    and reload Caddy
5. I update the CNAME record to point to `newserver.emoses.example`.

And...that's it really!  For most of the services I run, that Just Works and I'm all done:

* [VictroiaMetrics](https://victoriametrics.com/)
* [Grafana](https://grafana.com/)
* [Grocy](https://grocy.info/)
* [openWakeWord](https://github.com/dscripka/openWakeWord)
* [whisper](https://github.com/rhasspy/wyoming-faster-whisper)
* [piper](https://github.com/rhasspy/piper)

Those last three are part of Home Assistant's open voice services, text-to-speech, speech-to-text, and wake word
detection.  I'm actually barely using them but they're fun to play with a bit and take up basically no resources if
you're not using them actively.

### Unifi controller

I was a little scared of moving this one because I really don't want my wireless to go down, but it turned out to be
almost as trivial.  In particular, I had already set up the "Inform" address that the Unifi APs use to talk to the
controller to be a FQDN `unifi.home.emoses.example` instead of an IP address.  That meant I could

1. Back up the controller (using the web ui), which produces a JSON file.
1. Bring up a brand-new controller on the new host
1. Restore the backup
1. Change the DNS for `unifi.home.emoses.example` to the new host, and wait for the APs to show up
1. Bring down the old controller.

It was basically the same process.

### Frigate

I have a few security cameras and I
