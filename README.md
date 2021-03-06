# Linux-Fake-Background-Webcam

## Background
Video conferencing software support for background blurring and background 
replacement under Linux is relatively poor. The Linux version of Zoom only 
supports background replacement via chroma key. The Linux version of Microsoft 
Team does not support background blur. Over at Webcamoid, we tried to figure out
if we can do these reliably using open source software 
([issues/250](https://github.com/webcamoid/webcamoid/issues/250)).

Benjamen Elder wrote a
[blog post](https://elder.dev/posts/open-source-virtual-background/), describing
a background replacement solution using Python, OpenCV, Tensorflow and Node.js.
The scripts in Elder's blogpost do not work out of box. In this repository, I
tidied up his scripts, and provide a turn-key solution for creating a virtual
webcam with background replacement and additionally foreground object placement,
e.g. a podium. 

Rather than using GPU for acceleration as described by the original blog post, 
this version is CPU-only to avoid all the unnecessary complexities. By 
downscaling the image sent to bodypix neural network, and upscaling the 
received mask, this whole setup runs sufficiently fast under Intel i7-4900MQ. 

## Prerequisite
You need to install v4l2loopback. If you are on Debian Buster, you can do the
following:
    
    sudo apt install v4l2loopback-dkms

I added module options for v4l2loopback by creating
``/etc/modprobe.d/v4l2loopback.conf`` with the following content:

    options v4l2loopback devices=1  exclusive_caps=1 video_nr=2 card_label="v4l2loopback"
    
``exclusive_caps`` is required by some programs, e.g. Zoom and Chrome.
``video_nr`` specifies which ``/dev/video*`` file is the v4l2loopback device.
In this repository, I assume that ``/dev/video2`` is the virtual webcam, and
``/dev/video0`` is the physical webcam.

I also created ``/etc/modules-load.d/v4l2loopback.conf`` with the following content:
    
    v4l2loopback
    
This automatically loads v4l2loopback module at boot, with the specified module
options.

If you get an error like
```
OSError: [Errno 22] Invalid argument
```

when opening the webcam from Python, please install v4l2loopback from the [github](https://github.com/umlaeute/v4l2loopback) repo, 
as you could have an old version from your package manager.

When using Ubuntu 18.04 (it may work for other versions too) to install v4l2loopback correclty follow [the instructions](https://github.com/jremmons/pyfakewebcam/issues/7#issuecomment-616617011):

1. remove apt package
    - `sudo modprobe -r v4l2loopback`
    - `sudo apt remove v4l2loopback-dkms`
2. install aux
    - `sudo apt-get install linux-generic`
    - `sudo apt install dkms`
3. install v4l2loopback from the repository
    - `git clone https://github.com/umlaeute/v4l2loopback.git`
    - `cd v4l2loopback`
    - `make`  # The other commands are not needed
4. instal mod
    - `sudo cp -R . /usr/src/v4l2loopback-1.1`
    - `sudo dkms add -m v4l2loopback -v 1.1`
    - `sudo dkms build -m v4l2loopback -v 1.1`
    - `sudo dkms install -m v4l2loopback -v 1.1`
5. reboot
    - `sudo reboot`

Do not forget to thank to @pedrodiamel for the guide!

## Installing with Docker
Please refer to [DOCKER.md](DOCKER.md). The updated Docker related files were
added by [liske](https://github.com/liske).

Using Docker is unnecessary. However it makes starting up and shutting down
the virtual webcam very easy and convenient. The only downside is that you
lose the ability to change background and foreground images on the fly.

## Installing without Docker
Please also make sure that your TCP port ``127.0.0.1:9000`` is free, as we will
be using it.

You need to have Node.js. Node.js version 12 is known to work. 

You will need Python 3. You need to have pip installed. Please make sure that 
you have installed the correct version pip, if you have both Python 2 and 
Python 3 installed. Please make sure that the command ``pip3`` runs.

I am assuming that you have set up your user environment properly, and when you
install Python packages, they will be installed locally within your home
directory.

You might want to add the following line in your ``.profile``. This line is
needed for Debian Buster.

    export PATH="$HOME/.local/bin":$PATH

### Installation
Run ``./install.sh``.

### Usage
You need to open two terminal windows. In one terminal window, do the following:

    cd bodypix
    node app.js

In the other terminal window, do the following:

    cd fakecam
    python3 fake.py

The files that you might want to replace are the followings:

  - ``fakecam/background.jpg`` - the background image
  - ``fakecam/foreground.jpg`` - the foreground image
  - ``fakecam/foreground-mask.jpg`` - the foreground image mask

If you want to change the files above in the middle of streaming, replace them
and press ``CTRL-C``

#### fakecam.py
Note that animated background is now support. The background image does not have 
to be a jpeg file. For the implementation details, please refer to commit 
[ee867be](https://github.com/fangfufu/Linux-Fake-Background-Webcam/commit/ee867be88e8fe5c9cfdd7d7a69f12ed3c3fb904c).
Basically you can use any video file that your OpenCV can read. 

If you are not running fakecam.py under Docker, it supports the following options:

    usage: fake.py [-h] [-W WIDTH] [-H HEIGHT] [-F FPS] [-S SCALE_FACTOR]
                [-B BODYPIX_URL] [-w WEBCAM_PATH] [-v V4L2LOOPBACK_PATH]
                [-i IMAGE_FOLDER] [-b BACKGROUND_IMAGE] [--tile-background]
                [--no-foreground] [-f FOREGROUND_IMAGE]
                [-m FOREGROUND_MASK_IMAGE] [--hologram]

    Faking your webcam background under GNU/Linux. Please make sure your bodypix
    network is running. For more information, please refer to:
    https://github.com/fangfufu/Linux-Fake-Background-Webcam

    optional arguments:
    -h, --help            show this help message and exit
    -W WIDTH, --width WIDTH
                            Set real webcam width
    -H HEIGHT, --height HEIGHT
                            Set real webcam height
    -F FPS, --fps FPS     Set real webcam FPS
    -S SCALE_FACTOR, --scale-factor SCALE_FACTOR
                            Scale factor of the image sent to BodyPix network
    -B BODYPIX_URL, --bodypix-url BODYPIX_URL
                            Tensorflow BodyPix URL
    -w WEBCAM_PATH, --webcam-path WEBCAM_PATH
                            Set real webcam path
    -v V4L2LOOPBACK_PATH, --v4l2loopback-path V4L2LOOPBACK_PATH
                            V4l2loopback device path
    -i IMAGE_FOLDER, --image-folder IMAGE_FOLDER
                            Folder which contains foreground and background images
    -b BACKGROUND_IMAGE, --background-image BACKGROUND_IMAGE
                            Background image path, animated background is
                            supported.
    --tile-background     Tile the background image
    --no-foreground       Disable foreground image
    -f FOREGROUND_IMAGE, --foreground-image FOREGROUND_IMAGE
                            Foreground image path
    -m FOREGROUND_MASK_IMAGE, --foreground-mask-image FOREGROUND_MASK_IMAGE
                            Foreground mask image path
    --hologram            Add a hologram effect
    
## License

    Linux Fake Background Webcam
    Copyright (C) 2020  Fufu Fang

    This program is free software: you can redistribute it and/or modify
    it under the terms of the GNU General Public License as published by
    the Free Software Foundation, either version 3 of the License, or
    (at your option) any later version.

    This program is distributed in the hope that it will be useful,
    but WITHOUT ANY WARRANTY; without even the implied warranty of
    MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
    GNU General Public License for more details.

    You should have received a copy of the GNU General Public License
    along with this program.  If not, see <https://www.gnu.org/licenses/>.
    
Please note that Benjamen Elder's 
[blog post](https://elder.dev/posts/open-source-virtual-background/)
is licensed under CC BY 4.0 (see the bottom of that webpage). According to 
[FSF](https://www.fsf.org/blogs/licensing/cc-by-4-0-and-cc-by-sa-4-0-added-to-our-list-of-free-licenses),
CC BY 4.0 is a noncopyleft license that is compatible with the GNU General 
Public License version 3.0 (GPLv3), meaning I can adapt a CC BY 4.0 
licensed work, forming a larger work, then release it under the terms 
of GPLv3.
