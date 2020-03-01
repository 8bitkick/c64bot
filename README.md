# c64bot
Commodore 64 twitter bot inspired by [@bbcmicrobot](https://www.twitter.com/bbcmicrobot)

## Background

These are initial notes on a project to create a Commodore 64 Twitter bot which runs users' tweets on an emulator and returns a video of the execution in a similar style to my [BBC Micro Bot](https://www.8bitkick.cc/bbc-micro-bot.html).

## Prerequisites

 - vice
 - xvfb
 - ffmpeg
 - [c64 KERNAL files](http://vice-emu.sourceforge.net/vice_4.html)
 - pulseaudio

## Cloud Commodore 64

A key piece of this project is to get a C64 emulator running programs and generating video purely via Linux command line. [VICE](http://vice-emu.sourceforge.net) looks like the best emulator for the job.

We *don't* want to run VICE in headless mode because we want video output to capture. However in the cloud we have no display to attach to and even trying to load VICE causes it to segfault.

The solution is to use X virtual framebuffer ([xvfb](https://www.x.org/releases/X11R7.6/doc/man/man1/Xvfb.1.xhtml)) as the X server for the headless c64 cloud instance and capture the Display to video using [ffmpeg](https://ffmpeg.org) x11grab as follows:

```s
Xvfb :99 -screen 0 1280x1024x16 

DISPLAY=:99 x64 &

ffmpeg -f x11grab -y -r 50 -video_size 718x546 -i :99.0+0,27  -pix_fmt yuv420p output.mp4
```

It works! Here's a screenshot of video back from a VICE Commodore 64 emulator running on an AWS EC2 instance:

![C64 boot screen](https://github.com/8bitkick/c64bot/blob/master/cloud-c64.png)


## Input

Using the Twitter API we can take tweet text containing a Commodore 64 BASIC program listing and then convert it into a .prg file ready for autorun on the emulator.

It is important to note that by default the C64 is in PETSCII character mode. To allow for mixed case we are going to prepend the user's code with a POKE to switch to mixed case mode.

```BASIC
poke 53272,23
```


Next use the `petcat` utility included with vice to tokenize the basic code:

```s
petcat -w2 -o basic.prg -- input.bas
```


## Auto loader

Using the [Autostart C64 plus](https://csdb.dk/release/viewpic.php?id=96916&zoom=1) disk, the idea is to load the disk, copy the basic program you outputted onto the autostart disk, and then begin execution. This command will do this all automatically:

```s
DISPLAY=:99 x64 -default -parallel8 1 -autostartprgmode 2 -8 "/home/working/autostart.d64" -autostartprgdiskimage "/home/working/autostart.d64" -autostart "/home/mark/basic.prg"
```


That's all for now... let me know if you have any corrections / suggestions