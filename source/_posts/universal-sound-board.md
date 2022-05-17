---
title: Universal Sound Board
tags:
  - Audio
  - Microphone
  - Soundboard
  - Java
  - Dropwizard
  - Derby
  - Discord
  - Zoom
  - Memes
  - Emojis
date: 2022-05-17 09:00:00
---


![Universal Sound Board Diagram](USBD-diagram-revised-2022-05-16.jpg)

## Introduction

Ever wonder how you might play audio clips to your listeners as if it came through your microphone?  I've wondered this a lot myself, and after many long, frustrating, and fruitless searches on the internet, Stack Overflow, Reddit, etc., I decided to write my own software to do just that.  My solution is called [UniversalSoundBoard](https://github.com/sethmachine/universal-sound-board) and uses the [Java Sound API](https://docs.oracle.com/javase/tutorial/sound/) with virtual audio devices to play arbitrary audio as input to the microphone.  

This functionality is provided by various [soundboard programs](https://en.wikipedia.org/wiki/Soundboard_(computer_program)), which are already available in a variety of paid, freemium, or open source flavors.  However, every implementation I came across lacked at least one of these essential features:

1.  A programmatic interface to completely control the soundboard (no GUI required)
1.  100% open source
1.  Free as in free (no freemium, trials, nagware, etc.)
1.  App agnostic: it works on any voice chat application (Discord, Zoom, etc.)
1.  OS agnostic: it works on any major operating system (Windows, macOS)

UniversalSoundBoard **provides all of these features**.  

My article covers the following topics:

* [What is a soundboard](#What-is-a-soundboard)
* [Why UniversalSoundBoard](#Why-UniversalSoundBoard)
* [How UniversalSoundBoard works](#How-it-works)
* [Next steps and future work for UniversalSoundBoard](#Next-steps)

My ultimate vision is to build a soundboard that plays audio clips from a vast library, triggered by fuzzy matching typed commands like ["@troll in the dungeon"](https://www.youtube.com/watch?v=oynJcSnLSI4&t=12s), rather than having to input a predefined hotkey combination or manually choose the file.  

If you'd like to dive straight into the code and tutorial, check out the open source GitHub repo.  I've provided a detailed technical guide on getting started.  

* GitHub: https://github.com/sethmachine/universal-sound-board

## What is a soundboard?

![Troll in the dungeon](troll-dungeon.gif)

Taken from the [Wikipedia article]((https://en.wikipedia.org/wiki/Soundboard_(computer_program))):

> A soundboard is a computer program, Web application, or device...that catalogues and plays many audio clips...The soundboard is also used to facilitate humor, highlighting some of celebrities' more unusual utterances, or allowing for juxtaposition or even "composition" of quotes and sounds that would otherwise not go together.

In summary: 

* Soundboards can be used to be great comedic affect, and act as form of audio meme/emoji.
* Soundboards "catalogue and play many audio clips"--soundboards allow for playing a single audio clip from a potentially massive library of sounds.  
* Soundboards are natural companions for online multiplayer video games, where one is immersed and constant text chat is not feasible


## Why UniversalSoundBoard


Why build or use UniversalSoundBoard (USBD) when there many other higher quality soundboards available?  USBD is a free, 100% open source, programmable, app and OS agnostic soundboard program.  No other soundboard I found in my search met all 5 criteria.  

### Programmable 

USBD is written as a [Dropwizard](https://www.dropwizard.io/en/latest/) RESTful webservice and provides an HTTP API to programmatically control the entire soundboard.  One can issue GET requests to figure out what sound devices (microphones, speakers) are installed, make POST requests to use with with certain audio formats, and make additional POST requests to play any audio file.  The USBD API is extendable and can be used without being aware of nuances of the underlying sound APIs.  Further, one could build a GUI by implementing the USBD HTTP API.  

In my search, I only came across soundboard software that were GUIs with no advertised programmatic interface
 
### 100% open source

UniversalSoundBoard is open source.  Anyone can contribute to it, learn from, or build their own software on top of it.  

There are some free soundboard programs, but they aren't open sourced and it makes learning from them hard.  By open sourcing this, I hope others are able to learn from my work and result in saving themselves the countless hours of research that I had to do.  

### 100% free

The higher quality soundboard software are usually paid (and also still don't offer a programmatic API?!).  My use case for a soundboard is quite simple: programmatically play an audio file through the microphone anytime.  I have no need for more intricate or advanced audio setups, therefore others who desire simple functionality shouldn't have to pay for features they don't need.   

### App agnostic

Discord, Zoom, and other applications do offer integration APIs to build powerful customizations.  However, this comes with a penalty of learning a 3rd party API and being restricted to what is offered.  Further, this require maintaining one or more implementations of the same software.  

Since the functionality of USBD is so simple (play an audio file through the microphone), there is no need to build integrations.  Instead, a single standalone app can interface with the OS sound system, which is then used by any voice chat application.  

USBD accomplishes this by use the Java Sound API and leveraging virtual audio devices.  

### OS agnostic

Some soundboard software only works on Windows, others only work on macOS, and some only on Linux.  I happen to jump between macOS and Windows a lot when I'm gaming, and it'd be nice to just use a single software on both.  

USBD accomplishes this by being 100% Java.  Meaning once it's compiled, it should theoretically be able to run on any Java compatible operating system.  Indeed, I can compile USBD on macOS and the jar runs perfectly on my Windows machine.  In theory Linux should also work, though I have yet to test this (I welcome any readers confirming this!).  

## How it works

At a high level, USBD works by streaming microphone input and audio file input (separately) into a virtual audio device (VAD).  The VAD receives audio bytes from both streams and automatically turns this into microphone input.  This is made more clear in the following steps:

1.  USBD runs a background thread that listens to physical microphone input, until muted or turned off.  All microphone input bytes are streamed to a chosen virtual audio device.  
1.  USBD runs another background thread for each input audio file.  All audio file bytes are streamed to the same virtual audio device as the microphone.  
1.  Optionally, USBD can simultaneously stream audio file input to physical speakers as well, allowing both ends of the voice chat to hear the audio playback at the same time.  
1.  The user sets the audio input device (microphone) to the virtual audio device for the voice chat application (e.g. Discord, Zoom)

In the next sections, I explain the different technologies underlying USBD and how they work together to create a programmable soundboard.  

* [Sinks and sources](#Sinks-and-Sources): these are high level abstractions of audio devices that are the building block of USBD
* [Virtual audio devices](#Virtual-audio-devices): virtual audio devices allow for turning audio output into audio input
* [API](#API): this describes the HTTP API USBD offers to act as a programmable soundboard
* [Technologies](#Technologies): this provides a high level overview of the specific technologies USBD uses
* [Systems Design](#Systems-Design): this provides detailed systems design diagrams on the main flows of USBD and how it all comes together

### Sinks and Sources

USBD organizes audio devices into 2 types: (1) sinks and (2) sources.  These more or less correspond exactly to Java's notions of (1) [target audio devices](https://docs.oracle.com/javase/7/docs/api/javax/sound/sampled/TargetDataLine.html) and (2) [source audio devices](https://docs.oracle.com/javase/7/docs/api/javax/sound/sampled/SourceDataLine.html).  I chose to use sink instead of target as I found the naming more intuitive.  This terminology is also heavily influenced by [PulseAudio](https://www.freedesktop.org/wiki/Software/PulseAudio/).  Thus defined:

* Sink: an audio device that collects audio data and can be read from.  One example is a microphone.  
* Source: an audio device that plays back audio and can be written to.  One example is speakers.

Since microphones are sinks, they only offer APIs to read any captured audio bytes.  This means there is no direct way to programmatically provide input to a microphone.  This is where a virtual audio device is required.  

In contrast, sources offer only offer APIs to write audio bytes.  Hence it is trivial to write a Java program to play an audio file on physical speakers.  This is done by streaming audio bytes and using the SourceDataLine write method.  The reverse--capturing audio bytes being sent to a speaker, is not of interest here, since we'll always know what was played in the sound board program.  

Some audio devices can offer both sink and source interfaces.  In particular, virtual audio devices can support both sink and source interfaces.  This means they can both receive bytes (like a source/speaker) but also output bytes like a microphone (sink).  More importantly, they can be programmed to automatically take bytes written and turn these into input bytes.  Thus, acting like a microphone that can be written to.  

### Virtual audio devices

Virtual audio devices act as programmable audio devices that can receive audio input and provide audio output.  They can be used exactly like physical audio devices when configuring voice chat applications like Discord or Zoom.  However, on their own, they do nothing, because they need a way to receive audio bytes for input or output.  This is because they have no physical apparatus to capture audio or play it back, unlike physical speakers or a built-in microphone.  Instead, other programs must send audio bytes to them.  This is exactly what USBD does: wire a physical audio device to a virtual audio device.  

Thankfully, there are multiple free and open source virtual audio devices available for all major operating systems.  I've tested the ones below with USBD on both Discord and Zoom.  

#### Windows
* [Virtual Audio Cable](https://vb-audio.com/Cable)

#### macOS
* [Virtual Audio Cable](https://vb-audio.com/Cable)
* [BlackHole](https://github.com/ExistentialAudio/BlackHole)

### API
There are many actions USBD needs to support to be a functional programmable soundboard.  Each soundboard action generally corresponds to a single HTTP request.  

1.  [GET /audio-mixer-metadata/descriptions](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/AudioMixerMetadataResource.java#L31-L39)
: List all available sounds devices on the user's machine.  This includes both physical and virtual audio devices.

1.  [GET /audio-mixer-metadata/audio-formats](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/AudioMixerMetadataResource.java#L41-L58)
: For a specific audio device (e.g. microphone), list all audio formats it supports.

1. [POST /audio-mixers](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/AudioMixersResource.java#L31-L34)  
: Specify which sink or source audio device to use, along with the expected format of audio input.

1. [POST /sinks/start](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/SinksResource.java#L26-L30)  
: Continuously listen to all input audio bytes from a sink (e.g. physical microphone).

1. [POST /sources/play](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/SourcesResource.java#L30-L46) 
: Accept audio files as input to a source audio device (e.g. Blackhole, physical speakers, etc.).

1.  [POST /audio-mixer-wiring](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/AudioMixerWiringResource.java#L35-L38)
: Wire together a sink to a source (e.g. physical microphone to Blackhole).  All input bytes read from the sink are automatically written to the source.
  
1.  [POST /sound-board/play](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/SoundBoardResource.java#L28-L45)
: Simultaneously stream an audio file to 1 or more sources (e.g. play to Blackhole and physical speakers at the same time, so both ends of the voice chat hear the audio file).

1. [POST /sinks/stop](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/java/io/sethmachine/universalsoundboard/resources/SinksResource.java#L32-L36)
: Turn off the sink (i.e. a mute function for a microphone).    

### Technologies

USBD runs as a [Dropwizard](https://www.dropwizard.io/en/latest/) RESTful webservice that accepts incoming HTTP requests to perform different soundboard commands as outlined in [API](#api).  A webservice approach provides two key benefits: 
1.  soundboards inherently need to run forever as a background process, which is what a webservice also does
1.  HTTP requests are language agnostic, so this allows for building abstractions on top of USBD easily and are not restricted to Java.  E.g. one could build a Python soundboard that uses USBD under the hood.  

USBD provides persistence through an embedded [Apache Derby SQL database](https://db.apache.org/derby/).  This allows maintaining the different audio wirings of physical and virtual audio devices so users do not have to re-create these each time the server is started.  I use [Liquibase](https://www.liquibase.org/) to automatically create the database (if it doesn't exist) and write its [schema in SQL](https://github.com/sethmachine/universal-sound-board/blob/master/src/main/resources/migrations/migrations.sql).  To seamlessly map database rows to Java objects, I use [Rosetta](https://github.com/HubSpot/Rosetta).  

### Systems Design

The follow diagrams illustrate the two key flows of USBD and how the different technologies interact.

1.  Determine which audio devices are to act as sinks and sources, and then wire these together.  
1.  Capture audio from the sink (microphone) in the previous step and write it to the virtual audio device (source).  Do the same for any audio file input.  

#### Specify audio devices

![USBD create mixers](USBD-create-mixers.png)

In this flow, a user wishes to use their microphone as a sink and BlackHole virtual audio device as a source.  The user queries the USBD API to find the exact name and audio format of their microphone.  They do the same for the BlackHole virtual audio device as well.  These are written to the embedded database as generic audio mixers, so that they can be referenced again in future uses.  Finally, the user wires the sink to the source, which lets USBD know that all audio input from the microphone should be directed to the BlackHole virtual audio device.  

#### Play audio

![USBD create mixers](USBD-play-audio.png)

This flow illustrates how the microphone and arbitrary audio files are written to a virtual audio device, ultimately becoming a single audio input for a voice chat application.  In the first part, the user references the ID of the sink set up in the previous flow, which is the microphone being used.  The user asks USBD to start the sink, which will turn the microphone on and cause all audio input bytes to be written to the BlackHole virtual audio device.  With BlackHole specified as the audio input on the voice chat application, listeners will hear the user's microphone as normal.  Separately, the user then specifies to play an audio file to the BlackHole device, referencing the source ID set up in the previous flow.  This causes all audio bytes to be written to the BlackHole device, used as input for the voice chat application, and ultimately heard by listeners on the other end.  Throughout this flow, the database is queried to get the correct audio devices and formats.  




## Next steps

In this article I explained why I created UniversalSoundBoard (USBD), the different use cases of a soundboard, and how USBD works.  

UniveralSoundBoard is a free, programmable, open source, app and OS agnostic soundboard program.  I created USBD for my own personal usage, as a building block for more complex soundboard applications, and to share my learnings with others looking to learn the same.  

Next steps are:

* How can we trigger playing an arbitrary audio clip in a frictionless manner?    
* How can we gather a library of high quality audio memes in a semi-automatic fashion?
* How could we develop our own virtual audio devices without relying on 3rd parties?   

I invite others to explore these next steps and what could be built on top of UniveralSoundBoard.  I've enjoyed using UniversalSoundBoard in my own Discord and Zoom sessions with friends and I continue to plan to do so.  