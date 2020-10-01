---
layout: post
title: "Homebrewing a MUD Engine - Part One"
day: 14
tags: [nodejs, moleculer, mud, gaming]
comments: true
---

The [MUD](https://en.wikipedia.org/wiki/MUD) is **THE** OG of the MMO world. 
[Created by two University of Essex attendees]https://en.wikipedia.org/wiki/MUD1)
it defined the very idea of an online multiplayer game. One would think, that in
a world where this concept has evolved into a billion dollar industry with titans
such as World of Warcraft, EVE Online, and Elder Scrolls Online, the age of the
MUD is long passed. Nothing could be further from the truth.

MUDs still hold strong as a niche game, and most avid players would rather play
a MUD than any modern MMO. Many feel there is a richness that the imagination
can provide with text based online games that you don't get with the "big box"
products.

I am one of those. While I am not one who has played MUDs since the dawn of the
computer, I was interoduced to the style of game in the early ninties and it
still remains one of my favorite game generas over nerly thirty years later.

## The Concept
If you want to build your own MUD **NOW** then it doesn't make sense to create 
your own engine. With many amazing modern frameworks to choose from, 
([Ranvier](https://ranviermud.com/), [Evennia](http://www.evennia.com/), or
[ExVenture](https://exventure.org/) to name a few). Most actively managed and
developed by the excellent [Mud Coders Guild](https://mudcoders.com/).

For me creating a MUD doesn't scratch the itch. Instead it is creating the
framework behind it that revs my engine.

### Defining Requirements

There are a few things I want from a MUD engine:
1. **Easily extended** - I want something that is highly pluggable. That can be 
  extended with something as small as a single file, or something as complex
  as an entire library.
2. **Scalable** - Yeah, I know. Most MUDs only have a handful of active
  users at any given time. But again this is all to scratch an itch. I can
  overengineer all I want!
3. **Developer Friendly** - I am a Rubyist at heart, as such developer 
  friendliness and happiness are as apart of my DNA as whatever makes it so 
  that I grow excessively long eyebrows.

Outside of those three points, I will probably pull concepts and ideas from some
of these other awesome frameworks, and pull upon my own experiences working in
the AAA game industry on flagship MMO projects.

## High Level Design

![Chimera High Level](/img/chimera-high-level.png)

Keep in mind this is a **very** high level view of what I have in mind, but it
suffices for this article's sake.

### The Portal

The Portal will be the primary point for Telnet clients to connect to the game.
It shouldn't do much more than that. Ideally any socket server that speaks on
the same bus should suffice for a portal, regardless of the language it is
written in.

### The Bus

The Bus is a messaging system that will be used to facilitate communication
between the Portal and the World instances. I'm not going to go into the choice
of on what I will use for this layer just yet.

### The World
The World represents the game itself. In fact, it can be multiple worlds. A
game may have 1 or 50 world instances all running at once. This allows for some
pretty interesting flexibility in the future. Maybe creating 
(instance dungeons)[https://en.wikipedia.org/wiki/Instance_dungeon] for example.

### The Data Layer
We need a place to store information. This is the data layer. It should simply
do that, store information. This could potentially be a SQL database, a flat
file, a NoSQL datbase, or sending packets to the moleman that lives under my
house. What we will use for this is a descision for another time.

---

That's it for today! For now we have a high level concept that lives only in
my mind. In the next of what I hope to be a series of articles we will tackle
the intracacies of creating such a framework. I want these articles to be
relatively short and consumable. Stay tuned!

