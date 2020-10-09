# c64bot
A general method to create twitter code bot inspired by my [@bbcmicrobot](https://www.twitter.com/bbcmicrobot) using c64 as an example

## Background

This repo outlines techniques for those inerested in creating a Twitter bots for other emulated machines - which remotely executes code on an emulator and returns a video of the execution in a similar style to [BBC Micro Bot](https://www.8bitkick.cc/bbc-micro-bot.html). You'll notice this method of video capture can in fact be used with any X11 application. Applications of this method now implemented include [Atari8bitbot](https://atari8bitbot.com), [AppleIIbot](https://atari8bitbot.com/apple-ii-bot/), and beta [Acorn Archimedes bot](https://twitter.com/bbcbasicbot).

The method differs from BBC Micro Bot. BBC Micro Bot provides a great starting point for the Twitter bot mechanics for any application. However the emulation piece is specific to the BBC Micro - it has deep hooks enabling faster than real-time images and audio capture, and direct injection of input into the emulator memory. 

Here we propose a method for any machine.


## Prerequisites

General
 - xvfb
 - ffmpeg
 - pulseaudio
 
c64 specific (this example)
 - [c64 KERNAL files](http://vice-emu.sourceforge.net/vice_4.html) 
 - vice 
 

## Running the emulator in the cloud

A key piece of this project is to get an emulator running and capture video output purely via Linux command line. In the case of Commodore 64 [VICE](http://vice-emu.sourceforge.net) looks like the best emulator for the job. However any X11 based application / emulator could be used.

To do this we use X virtual framebuffer ([xvfb](https://www.x.org/releases/X11R7.6/doc/man/man1/Xvfb.1.xhtml)) as the X server for the headless emulator instance and capture the Display to video using [ffmpeg](https://ffmpeg.org) x11grab as follows:

```
Xvfb :99 -screen 0 1280x1024x16 

DISPLAY=:99 x64 &

ffmpeg -f x11grab -y -r 50 -video_size 718x546 -i :99.0+0,27  -pix_fmt yuv420p output.mp4
```

It works! Here's a screenshot of video back from a VICE Commodore 64 emulator running on an AWS EC2 instance:

![C64 boot screen](https://github.com/8bitkick/c64bot/blob/master/cloud-c64.png)

## Capturing Audio

It is possible to capture an audio stream with `ffmpeg` by adding the `-f pulse` switch. An example currently used by [Acorn Archimedes bot](https://twitter.com/bbcbasicbot):

```
DISPLAY=:99 timeout 35s ./arculator my & sleep 30 ; ffmpeg  -framerate 15 -f x11grab -draw_mouse 0 -s 640x512 -y -t 3 -i :99.0+16,12  -f pulse -thread_queue_size 1024  -t 3 -i default -ar 22050 -ab 64k -c:v libx264 -pix_fmt yuv420p -vf "scale=1280:1024" ./tmp/FRAME_CAPTURE.mp4
```

This worked in initial tests. However more investigation is needed on synchronisation and to avoid any audio being dropped. It's possible that running `parecord` to capture raw audio and muxing it with `ffmped` in a second stage might be more effective. The best solution found so far, as used by BBC Micro Bot, is to modify the emulator save audio to a buffer file directly. 

## Formatting input

Using the Twitter API we can retrieve tweet text containing a Commodore 64 BASIC program listing and then convert it into a .prg file ready for autorun on the emulator.

It is important to note that by default the C64 is in PETSCII character mode. To allow for mixed case we are going to prepend the user's code with a POKE to switch to mixed case mode.

```BASIC
poke 53272,23
```

Next use the `petcat` utility included with vice to tokenize the basic code:

```s
petcat -w2 -o basic.prg -- input.bas
```
DISPLAY=:99 timeout 35s ./arculator my & sleep 30 ; ffmpeg  -framerate 15 -f x11grab -draw_mouse 0 -s 640x512 -y -t 3 -i :99.0+16,12  -f pulse -thread_queue_size 1024  -t 3 -i default -ar 22050 -ab 64k -c:v libx264 -pix_fmt yuv420p -vf "scale=1280:1024" ./tmp/FRAME_CAPTURE.mp4

## Loading data into the emulator

We have two choices here

* Emulator autoboot of disc image or room

Where the emulator supports it, the twitter bot just needs to inject user status text into a disc image read by the emulator.

For the commodore 64 using the [Autostart C64 plus](https://csdb.dk/release/viewpic.php?id=96916&zoom=1) disk, the idea is to load the disk, copy the basic program you outputted onto the autostart disk, and then begin execution. This command will do this all automatically:

```s
DISPLAY=:99 x64 -default -parallel8 1 -autostartprgmode 2 -8 "/home/working/autostart.d64" -autostartprgdiskimage "/home/working/autostart.d64" -autostart "/home/mark/basic.prg"
```

* Use Xdotool to simulate keyboard presses

Similarly for the Acorn Archimedes this is achieved using a floppy image in hostfs folder. 

This is theory could be generalized to any machine but needs more investigation




That's all for now... let me know if you have any corrections / suggestions / implementation
