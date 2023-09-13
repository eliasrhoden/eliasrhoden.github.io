---
layout: post
title:  "SAAB Bluteooth AUX"
date:   2023-09-13 12:00:00 +0200
categories: embedded systems, programming
---


This post will cover an old project of mine, where I built a small module that enabled me to have Bluetooth handsfree audio in my first car, 
a SAAB 93 from 2001.

![](/assets/images/saab.jpg)

After my first year at university where I did a course in embedded software development, 
I really got interested in embedded systems. I started with some smaller arduino projects 
but wanted to do something outside of the arduino framework, and since we had been using 
STM32 microcontrollers in University, I decided to use one of their Nucleo boards.

Another big inspiration that increased my interest in embedded systems was 
this video: https://www.youtube.com/watch?v=aEFPU6iUo-k  where Matt Denton goes through 
how his Mantis 6-legged robot works. I really recommend this one if you are into big computer 
controlled machines (Let’s be real, who is not?).

Anyway, back to the bluetooth device, I first heard about a product called BlueSaab 
through this video:https://www.youtube.com/watch?v=30qahGvou3k where a SAAB owner showed 
how the BlueSaab module worked. I got really excited, it was a guy in the US that had designed 
a PCB that emulated the CD-changer normally found in the trunk of SAAB and replaced the audio 
input with a bluetooth chip. Here is the homepage of Bluesaab if you’re interested: https://bluesaab.blogspot.com/

For me this sounded like a perfect project, since the hardware components had already been decided, 
it enabled me to mainly focus on learning how to design a PCB, and writing the software. 
I used the same bluetooth module, and reused some of their CAN-message code, the rest I wrote myself.

Sourcing the parts was not too difficult, I remember that the most difficult part to find was the connector to the wire harness.

![](/assets/images/pcb_1.jpg)

![](/assets/images/pcb_2.jpg)

My code, although I’m not very proud of it, can be found on my github: https://github.com/eliasrhoden/SAAB_BluetoothAUX/tree/master/SAAB_Bluetooth_AUX

In the end I consider this project a success, I used the device for the rest of the time I owned the car, 
and even included it with the car when I sold it. The most fun with the project was that it was really useful for me, 
there were no other audio input devices other than CD and FM radio, and I thought it was really cool that I had 
Bluetooth AUX input with the original stereo.

![](/assets/images/saab_tr.jpg)

One last SAAB story before the end of this pos, during my short time in the automotive industry I heard about the *steer-by-wire SAAB*, 
a car built by SAAB using aerospace hydraulics and was controlled by a joystick, which I think is so cool.
Check out this old Swedish news-story about it:
https://youtu.be/eDUG2UuTve8?si=7oASPSDw_wGW88W5&t=19

That was all for this time, I have some new hardware projects ongoing that I hope to write about soon.






