---
title: Video signal interception using SDR
published: true
---

# Introduction
Any electrical equipment emits side electromagnetic waves. An intruder with special equipment and software is able to intercept interference and recover any information from specific interface. And image transfer interfaces are not exception. 

In this article I will describe my experience of capturing images using software-defined radio (SDR), what were the difficulties and how I struggled with them.

I performed all actions based on information (but not limited to it) from Martin Marinov's dissertation ["Remote video eavesdropping using a software-defined radio platform"](https://raw.githubusercontent.com/martinmarinov/TempestSDR/master/documentation/acs-dissertation.pdf). It describes the theoretical foundations of this type of attack. Despite some pretty serious math used in this article, I advise you to read this work for a deeper understanding of what is happening/

# Rules of the game

I will try to attack within the following rules:
- The user uses an HDMI/DVI cable to display the image on the screen;
- The attacker has a broadband SDR receiver and a directional antenna with significant gain;

The goal is to intercept an image from the output device, obtain data and metadata about the user's work, espionage.

RTL-SDR was used as a broadband receiver. The host system was a laptop based on the Windows 10 operating system with the installed [TempestSDR](https://github.com/martinmarinov/TempestSDR) and [SDR#](https://airspy.com/download/) software.

# Stage one: recon

First of all, I needed to understand at what frequencies the victim's computer emits electromagnetic waves. For these purposes, a test stand was created, which was a computer connected to a monitor via DVI. The SDR# utility was used to determine the required frequencies. The required frequency was found by searching through all possible frequencies and viewing the signal level at them. To validate the received frequency, the computer was disconnected from the monitor: if the signal received by the receiver fell, then the desired frequency was determined correctly. The signal transmitted by the DVI cable and received by the radio module was a sequence of "peaks" and looks something like this:

![](/resources/videosignalinterception/sdrsharp.png)
I assume that the "peaks" are the signal at the harmonic frequencies after the Fourier transform. I will be glad if you explain to me the real reason for the appearance of such "peaks".

I will make further research based on the assumption that interference is emitted at a frequency of 593.405 MHz.

# Stage two: preparation

After determining the main (carrier) frequency, I decided to assemble my own directional antenna suitable for a specific frequency. I chose [Yagi-Uda antenna](https://en.wikipedia.org/wiki/Yagi%E2%80%93Uda_antenna) as the antenna type. The reasons for choosing this type of antenna are pretty obvious:
- It allows you to significantly increase the power of the received signal;
- It is quite easy to assemble;

The principle of this antenna is that the radio signal in the front parts (directors) is radiated with a certain phase shift, so that the signal in the direction of the antenna is significantly amplified as a result of constructive interference.

To assemble the antenna, we need to create a reflector, a dipole, and a director(s), each with a length approximately equal to the wavelength in half.
Calculate the wavelength using the formula $\lambda = \frac{c}{\nu}$, where $c$ is the speed of light and $\nu$ is the frequency:

$\lambda = \frac{c}{\nu} = \frac{{299792458}{\space}{m/s}}{593405000 \space Hz} = 0.50520716542 \space m \approx 50.5 \space cm$ 


The reflector, dipole and director(s) should have a length of about $\frac{\lambda}{2}$. The reflector should be slightly longer than the dipole (about 5%), and the dipole should be slightly longer than the directors (also about 5%). The distance between reflector and dipole should be equal to $\frac{\lambda}{4}$. The distance between the directors and the distance between an active element and the director varies between $0.35\lambda$ to $0.4\lambda$. 

This means that the reflector, dipole and director(s) should have a length of about $\frac{\lambda}{2} \approx 20.25 \space cm$. The distance between reflector and dipole should be equal to $\frac{\lambda}{4} \approx 10 \space cm$ and the distance between dipole and director should be equal to  $0.35\lambda \approx 7 \space cm$


## Creating an antenna: the first version

The first version of the antenna was a Yagi-Uda antenna with three elements (one reflector, one dipole and one director), a telescopic antenna from the Aliexpress acted as a dipole, pieces of coaxial cable acted as a reflector and director, and the pencils glued together were the body :D

Antenna photo: ![](/resources/videosignalinterception/antenna1.jpg)

Do not focus on the fact that the dipole in the photo is longer than all the other elements. Experimentally I found that this configuration has the best SWR :D. 

Despite the fact that the antenna was made with love, a good signal could not be achieved, the picture was very blurry, the interception range was very small and a [standing wave ratio](https://en.wikipedia.org/wiki/Standing_wave_ratio) was 2, which made me very sad... I must also admit that I calculated the characteristics for the first antenna using an online calculator, apparently it was not very good (or me).

An example of an intercepted image, the text "РЕДТИМ" (RedTeam in Russian) was displayed on the screen, the interception was carried out at a distance of 2 m from the victim:

![](/resources/videosignalinterception/antenna1example.jpg)

I see several reasons for this result:
- Inaccuracy of measurements, assembly "in haste";
- Poor quality of reflector and director;

I also note that I made the first test image captures on a macbook, connecting the receiver via a USB hub and running the software on Windows in a VMWare virtual machine. Perhaps this antenna would not have shown such poor results if I had tested directly on a computer running Windows and without using the USB-Hub. However, SWR = 2 (which I measured with a separate special device) seemed to me a very bad result and I decided to remake the antenna.

## Creating an antenna: the second version

The second version of the antenna was also Yagi-Uda, but with 7 elements (one reflector, one dipole and five directors). It was assembled from pieces of copper wire with a diameter of 1 mm and the same pencils :D

The characteristics of the second version of the antenna were much better, SWR was about 1.15 and the intercepted image was a fairly good.

Antenna V2 photo:
![](/resources/videosignalinterception/antenna2.png)

# Stage three: interception

In order to test the resulting configuration, I took several measurements at various distances from the victim's computer and with various accessibility to it.

1) 3 meters to the target, direct accessibility;
2) 5 meters to the target, one wall;
3) 8 meters to the target, two walls;
4) 11 meters to the target, three walls;

![](/resources/videosignalinterception/secret1.png)
![](/resources/videosignalinterception/secret2.png)
![](/resources/videosignalinterception/secret3.png)
![](/resources/videosignalinterception/secret4.png)

It is clear that the range of signal interception has become much higher along with the quality of the intercepted image.

Also, during the tests, images from neighboring rooms in the office were accidentally intercepted. On the first of them, you can observe the work of an employee in some kind of document management application, on the second, you can see the computer splash screen:

![](/resources/videosignalinterception/extra.png)
![](/resources/videosignalinterception/extra2.png)

# Possible improvements

The current configuration of the attacker's equipment, unfortunately, cannot allow capturing the image in a higher quality, for example, to read text smaller than 50pt. I see several reasons for this:
- The second version of the antenna, although it was made better than the first, is still a homemade craft, and, of course, not made in the best possible way;
- The cheapest software-controlled radio RTL-SDR module was used to intercept the image;
- Various filters and amplifiers were not used due to their absence :D

I suppose that after eliminating the above reasons, the quality of the captured image will become better.

However, I have one more idea, which, unfortunately, I have not yet implemented, and which, in my opinion, can greatly increase the quality of the captured image. 

The captured image is a video signal, but with a lot of noise. Since the text is basically a static image, and the intercepted signal is a series of frames (many frames per second, in our example about 60), I assume that it is possible to increase the quality of the picture by combining several frames into one, for example, take average pixel color for three adjacent frames, or come up with something smarter... (maybe I will implement something similar in the future)

Also, I do not deny the possible use of machine learning technologies to increase image quality...

I also think that with a similar attack, you can intercept the sound transmitted through the cable...

# Conclusion

A side-channel attack that allows you to intercept and restore an image using electromagnetic interference is possible and even very simple to implement.

Such an attack can be used to obtain both metadata about the work of the attacked user (software used in the work, working hours, etc.), and directly the data displayed on the user's output device.
