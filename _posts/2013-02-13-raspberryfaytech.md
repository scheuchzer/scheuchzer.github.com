---
layout: post
title: "RaspberryPi with Faytech Tochscreen"
description: ""
category: 
tags: [RaspberryPi, JavaFX]
---
{% include JB/setup %}

Just a quick note for anybody who's struggling with Touchscreens 
on the RaspberryPi.

I just downloaded the latest Raspbian Image 
[2013-02-09-wheezy-raspbian.zip](http://downloads.raspberrypi.org/images/raspbian/2013-02-09-wheezy-raspbian/2013-02-09-wheezy-raspbian.zip)
and started it. My Faytech 7" Touchscreen just works out of the box and it even works
with JavaFX applications. No more kernel compilation!

One tiny thing you might want to do is to calibrate the touch controller. `xinput_calibrator`
will do the job but you have to compile it yourself.

    sudo apt-get install libx11-dev libxext-dev libxi-dev x11proto-input-dev
    wget http://github.com/downloads/tias/xinput_calibrator/xinput_calibrator-0.7.5.tar.gz
    ./configure
    make
    sudo make install
    xinput_calibrator
    
Touch the points until the application closes. Follow the instructions on the console output and 
create the configuration for X11. The directory for the configuration file is 
`/usr/share/X11/xorg.conf.d` instead of the directory shown by `xinput_callibrator`.

Reboot and you're done!



  