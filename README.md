# midi-over-network

Setup a midi comunication between two computers to trigger video with midi signal over the network.

## Before we start

Our physical setup consist of:

- Stage mixer [Midas MR18](https://www.midasconsoles.com/product.html?modelCode=0605-AAF) - Not relevant for this setup.
- [Cymatic LP-16 Live Player](https://cymaticaudio.com/product/lp-16-live-player-16-track-backing-track-system/).
- A router connected that provides wireless communication to the stage mixer (also used to set volumes for master and in-ear monitor via app).
- A FOH laptop used to receive MIDI signal over network to trigger and start videos connected to a monitor or a projector that often venues offer as an output source.
- A laptop/minipc/Raspberry connected to the Cymatic that receives midi signals and broadcast them on the router network.

Midas and Raspberry on the stage will be connected via cable to the router.

Laptop at FOH will be connected via WIFI to the router.

## Raspberry setup

In my case i'll be using an old Raspberry pi 3 model B

### OS setup

1. Download [Raspberry Pi Imager](https://www.raspberrypi.com/software/) and a [OS for the Raspberry](https://www.raspberrypi.com/software/operating-systems/) (Lite version is better for performance)
2. Follow instructions to install the OS on the SSD card of the raspberry - setup hostname, username, password, wifi (useful for initial configuration) and enable SSH

### Initial configuration

Once everything is installed, you should be able to SSH to the raspberry and start setting up the software needed.



