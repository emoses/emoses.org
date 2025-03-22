+++
title = 'Making a Family Dashboard display with Home Assistant and a Frame TV'
date = 2024-12-27
categories = ['blog']
tags = ['home-assistant', 'python']
draft = true
+++

## A shared dashboard

Our family's schedule lives in a set of shared calendars (hosted on [Fastmail](https://fastmail.com)); if it's not on the calendar, no one's
gonna do it or go to it.  My kid is a young grade schooler as of this writing, and she's at the age that it's really
useful for her to have an idea what's happening over the next couple days, but's she's defintely not old enough to have
her own phone yet.

One of the projects I've been considering for a long time is a smart mirror (TODO: link) or family dashboard that we
could all look at.  I thought about building an eInk dashboard using an old Kindle or a purpose-bought eInk display, but
it always seemed like *just a little more work* than I had time to do.

## The Frame

I came across a discount on Samsung's The Frame TV (TODO: link), and it checked a number of boxes for me:

* It's pretty and can blend in with the decor, with a matte screen and a selection of custom frames that make it look like a painting.
* It's got a local API for cloud-free control
* "Art mode" is a relatively low-power mode for displaying paintings that is supposed to make it look like a painting,
  but you can use it for your own "art"; I figured I could use this to display my dashboard.

So I bought one and mounted it our main living/dining room.

TODO: PICTURE HERE

## Static only

At first I assumed I'd just be able to turn on "Art Mode" and point it at my Home Assisant dashboard, but I was quickly
disabused of that notion.

It turns out the custom art in Art Mode is only for static images[^1]. This makes a certain amount of sense, as it's
intended to appear to be a painting, but it meant I would have to start getting a little clever.

[^1]: There are actually a number of video-type images in the Samsung "Art Store", but it's not supported for custom
    user-uploaded art.

## First pass: An Art Pipeline

I'm a coder and a maker, I've got a home server, I figured I could whip up something pretty simple to display what I
wanted on the Frame.  It would consist of a few parts:

* A Home Assistant dashboard to display
* A web scraper that could log in to Home Assistant and take a screenshot of the dashboard
* Some code to hit the Frame API to upload the screenshot.

### The Frame client

Some googling a few forum posts later I found out that someone (TODO: credits) had developed a [Python
library](https://github.com/NickWaterton/samsung-tv-ws-api.git) that knows
how to talk to the Frame's local WebSocket API and can upload custom art to Art Mode.  I wrote a [small CLI application](https://github.com/emoses/frame-scraper/blob/3da04e52511abfd41fbafc536a05e4fbf261a432/tv/test.py)
that used that library and did some test uploads of images; it worked without much trouble.

### The scraper

Now I needed to write a web scraper to take a screenshot.  I've written scrapers before using Selenium, and I have my
pick of languages with Selenium bindings. While I've worked professsionally
with Python in the past, I'm always frustrated by how hard it is to reliably distribute a Python application, so I
decided I would use Go to write the scraper.  This would:

* Start a brower
* Log into HA using username/password[^2] for a user I'd created specifically for this scraper
* Navigate to the frame dashboard in kiosk mode (TODO: link)
* Wait for the page to load
* Take a screenshot
* Write it to a file

[^2]: While home assistant can create long-lived API tokens, they're not valid to log to use to log in to the UI, so
    we'll need to go through the username/password login flow.

Secrets would be passed in to the applications via environment variables, which let me maintain a `.env` on my
development laptop for testing.  With a bit of debugging, [this worked](https://github.com/emoses/frame-scraper/blob/3da04e52511abfd41fbafc536a05e4fbf261a432/main.go).

### Containerization

I decided to pack these little applications up into Docker containers for a couple reasons.

1. My home server runs a number of self-hosted apps in Docker containers (using Docker compose) already.
1. Managing python libraries and versions in the host OS can be a nightmare
1. Managing Selenium Driver versions, which must be coordinated with browser versions, can be an even bigger nightmare.

If I containerize these apps I should be able to deploy them to my home server without worrying about managing the
various versions of things.

I built [a Dockerfile](https://github.com/emoses/frame-scraper/blob/3da04e52511abfd41fbafc536a05e4fbf261a432/Dockerfile)
for the scraper first, using the base image provided by Selenium, and tested it.

I ran into problems loading the page; a little bit of debugging showed me that the chromium browser was having trouble
loading the massive number of JS files served up by Home Assistant.  The fix was to increase the amount of memory
available to the container by adding `--shm-size="2g"` to the `docker run` command, which you can see in [the
Makefile](https://github.com/emoses/frame-scraper/blob/87c5c41f75e590288f45716732ed62e555ce4837/Makefile) in a later commit.

I wanted a 4k screenshot to send to the TV so I set the browser size to 4k (or 3840 x 2160 pixels) by passing
`"--window-size=3840,2160"` to Chromium.  However, it turned out that unlike my desktop browser, it didn't scale the
fonts to account for "HiDPI" screens.  To fix this, I changed it to `"--window-size=1920x1080"` and added
`"--force-device-scale-factor=2"`, which essentially tells chromium to zoom in.  This produced a 4k image that was
exactly what I wanted, and could be uploaded to the Frame.
